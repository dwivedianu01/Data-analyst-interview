# Day 04 - Senior Snowflake and ETL Interview Practice

Designed for a **9-year Data Analyst / Data Engineer** candidate. Mixes design, performance, reliability, and troubleshooting, with practical SQL where useful.

---

# Snowflake

## 1. Architecture

**Q: Explain Snowflake's storage, compute, and cloud-services layers. How do they work together when a query runs?**

- **Storage layer**: Data is stored as compressed, columnar micro-partitions (50-500 MB uncompressed) in cloud object storage (S3/Blob/GCS). Snowflake manages all file layout, compression, and metadata (min/max values, distinct counts per column per partition).
- **Compute layer**: Virtual warehouses — independent MPP clusters of compute nodes. Each warehouse reads from shared storage but has its own local SSD cache; warehouses don't contend with each other.
- **Cloud services layer**: Handles authentication, access control, query parsing/optimization, metadata management, and transaction coordination. This layer decides *which* micro-partitions to scan (pruning) before compute even starts.
- **Query flow**: Client sends SQL → cloud services parses, optimizes, checks result cache and access rights → if not cached, compiles a plan, picks a warehouse → warehouse pulls only pruned micro-partitions (first from local cache, else from storage) → executes in parallel → returns result, and metadata/statistics get updated.

---

## 2. Warehouse sizing

**Q: A dashboard is slow during peak hours, but other workloads must not be affected. How would you investigate and improve performance?**

- Investigate: check `QUERY_HISTORY`/Query Profile for the slow queries — look at queuing time (`EXECUTION_TIME` vs `QUEUED_PROVISIONING`/`QUEUED_OVERLOAD`) to see if it's **contention** (too many queries on the same warehouse) vs genuine **compute-bound** work (large scans, spilling to disk/remote).
- If queuing/concurrency is the issue: isolate the dashboard onto its **own warehouse** so other workloads can't starve it, and/or enable a **multi-cluster warehouse** (auto-scale out) to absorb concurrent bursts.
- If it's compute-bound (large joins, spilling shown in profile as "bytes spilled to local/remote storage"): scale **up** (bigger node size) for that warehouse temporarily during peak hours, or fix the query (better filters, clustering, materialized views/aggregate tables).
- Use **Resource Monitors** and separate warehouses per workload to guarantee isolation, and consider a **dashboard-specific warehouse with auto-suspend** tuned low to control cost while still being warm during peak hours.

---

## 3. Scaling

**Q: When would you scale a warehouse up, scale it out with a multi-cluster warehouse, or use separate warehouses?**

- **Scale up (bigger size)**: a single query is slow because it needs more compute/memory per query — large joins/sorts, spilling to disk, big scans. Helps one query run faster.
- **Scale out (multi-cluster, same size)**: Many *concurrent* queries queue up but each individual query is already fast enough. Multi-cluster adds more clusters of the same size to absorb concurrency, not to speed up a single query.
- **Separate warehouses**: Different workloads (ETL vs BI vs ad-hoc analytics) have different usage patterns, SLAs, and cost owners. Isolating them avoids noisy-neighbor issues and gives cleaner cost attribution/chargeback, independent auto-suspend policies.

---

## 4. Micro-partitions

**Q: How would you tell whether a query is benefiting from partition pruning? What might prevent effective pruning?**

- Check **Query Profile**: compare "Partitions scanned" vs "Partitions total" on the table scan node. Good pruning = scanned ≪ total.
- Pruning relies on Snowflake's stored min/max metadata per column per micro-partition matching the filter predicate.
- Things that prevent pruning:
  - Filtering on a column that isn't naturally correlated with insert order (high cardinality, randomly distributed values) and no clustering key defined.
  - Wrapping the filter column in a function/expression (`WHERE TO_DATE(ts) = ...` instead of a sargable range on `ts`) — defeats min/max matching.
  - Implicit type casting or comparing incompatible types.
  - Table not clustered on the filtered column and natural ingestion order doesn't align (e.g., out-of-order backfills).
- Fix: define a clustering key on frequently filtered columns, rewrite predicates to be sargable, or reorganize load order.

---

## 5. Cost and performance

**Q: A query scans a large table every hour. How would you use Query Profile and warehouse settings to investigate its cost and runtime?**

- Query Profile: check bytes scanned, partitions scanned vs total, whether it's using the **result cache** (should show near-zero cost if identical query+data unchanged), and look for spilling or skew (one node doing most of the work).
- Check **warehouse size vs query needs** — oversized warehouse for a small scan wastes credits; undersized causes spilling.
- Consider: materializing the result into a smaller **summary table** refreshed hourly (so downstream queries hit a small table instead of rescanning raw data), or a **materialized view** if the aggregation pattern is stable.
- Check `WAREHOUSE_METERING_HISTORY` / `QUERY_HISTORY` in `ACCOUNT_USAGE` to trend credit consumption per run and confirm improvements after changes.
- Also check **auto-suspend**: if the warehouse resumes just for this hourly query, cold-start overhead matters; if it's always on for other work, compute is probably not the bottleneck — the query/table design is.

---

## 6. Incremental processing

**Q: How would you implement incremental ingestion and transformations using Snowflake Streams and Tasks? What limitations or failure cases would you plan for?**

- Create a **Stream** on the source table to capture row-level changes (inserts/updates/deletes as a CDC-like metadata overlay) without copying data.
- Create a **Task** (or a DAG of tasks via `AFTER`) scheduled on a cron, which runs `INSERT/MERGE ... SELECT * FROM stream WHERE SYSTEM$STREAM_HAS_DATA(...)` — consuming the stream advances its offset only on successful commit, which is what makes it safe/idempotent-ish.
- Limitations/failure cases to plan for:
  - A stream becomes **stale** if not consumed within the data retention period (default matches table's Time Travel retention) — must ensure task runs frequently enough or extend retention.
  - Only **one** consumer should consume a given stream in a transaction at a time; concurrent consumption can cause one to see no data.
  - Task failures: use `TASK_HISTORY` monitoring, set up error notifications (`ALTER TASK ... SUSPEND` on repeated failure, or integrate with alerting), and make the downstream MERGE idempotent in case a task is retried.
  - Streams only track changes since last consumption — if the underlying table is **recreated/swapped** the stream can become invalid.

---

## 7. Semi-structured data

**Q: How would you ingest and query nested JSON in a `VARIANT` column? When would you flatten it into relational columns?**

```sql
-- Ingest
CREATE TABLE raw_events (payload VARIANT, loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP());
COPY INTO raw_events (payload) FROM @stage/events/ FILE_FORMAT = (TYPE = JSON);

-- Query nested fields directly
SELECT payload:order_id::STRING AS order_id,
       payload:customer:email::STRING AS email,
       f.value:sku::STRING AS sku
FROM raw_events, LATERAL FLATTEN(input => payload:items) f;
```

- Query directly with `:` path notation and `LATERAL FLATTEN` for arrays while data is small/exploratory or schema is highly variable.
- **Flatten into relational columns** (a proper dimensional/typed table) once: the schema stabilizes, the same fields are queried repeatedly (performance — typed columns get proper pruning/clustering stats; VARIANT access is slower), BI tools need simple typed columns, or you need strong data quality guarantees (types, not-null, constraints).

---

## 8. Data recovery

**Q: A pipeline accidentally overwrites data. How could Time Travel or zero-copy cloning help with recovery and investigation?**

- Immediately **clone** the current (bad) table and the pre-incident state for investigation without touching production:
  `CREATE TABLE orders_bad_clone CLONE orders;`
  `CREATE TABLE orders_before CLONE orders AT (TIMESTAMP => '2026-10-02 09:55:00'::timestamp_ntz);`
- Compare the two clones (e.g., `MINUS`/`EXCEPT` on primary keys) to identify exactly which rows changed.
- Recovery options:
  - If the whole table should revert and nothing valid happened after: `UNDROP`/`CREATE OR REPLACE TABLE orders CLONE orders AT (...)`.
  - If valid writes happened after the bad overwrite (common case): don't blanket-restore; instead `MERGE` the affected business keys from `orders_before` back into current `orders`, preserving later legitimate changes.
- Time Travel window is limited (1 day default, up to 90 days on Enterprise+) — zero-copy clones taken quickly extend your investigation window without extra storage cost until data diverges.

---

## 9. Security

**Q: How would you design role-based access for analysts, engineers, and service accounts? How would you protect sensitive columns?**

- Follow Snowflake's **RBAC hierarchy**: functional roles (e.g., `ANALYST_RO`, `ENGINEER_RW`, `SVC_ETL`) granted to users, which inherit from access roles tied to specific object privileges, following least privilege. Use a role hierarchy (e.g., `SYSADMIN` owns objects, functional roles get granted privileges, not direct user grants).
- Use separate **warehouses per role group** for cost isolation and separate **databases/schemas per domain** so grants stay simple.
- Service accounts: dedicated users with **key-pair authentication** (not passwords), scoped to only the objects/stages/tasks they need, ideally with network policies restricting source IPs.
- Protect sensitive columns:
  - **Dynamic Data Masking** policies on PII columns (e.g., mask SSN/email unless role is in an allowed list).
  - **Row Access Policies** for row-level security (e.g., regional analysts only see their region).
  - **Object tagging** + classification to track which columns hold sensitive data, combined with masking policies applied via tags for scale.

---

## 10. Data sharing

**Q: How would you share curated data with another Snowflake account while avoiding file exports?**

- Use **Secure Data Sharing**: create a `SHARE`, grant it usage on a database/schema and `SELECT` on specific curated views/tables, then add the consumer account.
- Prefer sharing **secure views** over raw tables so you can filter/mask columns and avoid exposing underlying logic.
- For cross-cloud-region or cross-cloud-provider consumers, use **auto-fulfillment via Snowflake Marketplace / Listings**, which replicates data transparently.
- Benefits over file export: no data duplication/storage cost on the provider side, consumer always sees live data (or a controlled snapshot), no ETL/export pipeline to maintain, and access can be revoked instantly by dropping the share.

---

# ETL and Data Engineering

## 1. Pipeline design

**Q: Design a daily pipeline that ingests files from cloud storage, validates them, transforms them, and publishes analytics-ready tables. How would you handle each stage?**

1. **Discovery**: event-driven (S3 event → SQS/SNS → Snowpipe or Lambda) or scheduled listing with a manifest/control table tracking which files have been processed (`file_name`, `status`, `loaded_at`, `row_count`).
2. **Raw load**: `COPY INTO` a raw/staging table (VARIANT or typed, matching source schema), keep raw files immutable, record load metadata (file name, row count, checksum) for traceability.
3. **Validation**: a quarantine pattern — load everything into staging, then run checks (schema, nulls on required keys, duplicates, referential integrity, volume anomaly) and route failing rows to a `rejects` table/quarantine stage with reasons, rather than failing the whole batch.
4. **Transform**: idempotent `MERGE`/`INSERT OVERWRITE` into curated/analytics tables using dbt or SQL tasks, with clear staging → curated → mart layering.
5. **Publish**: swap/merge into reporting tables only after validation passes a threshold (e.g., <X% rejects), update a `_load_audit` table, and notify downstream consumers (or refresh a dashboard cache) only on success.

---

## 2. Idempotency

**Q: A job fails after loading data but before marking the run successful. How would you make a retry safe and prevent duplicate rows?**

- Make load + status update **transactional** where possible, or design each step to be idempotent independent of transactions:
  - Use a **unique natural key or dedupe key per batch** (e.g., `file_name` + `run_id`) and `MERGE` instead of blind `INSERT`, so re-running the same file doesn't duplicate rows.
  - Track progress in a **control/audit table** with states (`STARTED`, `LOADED`, `VALIDATED`, `PUBLISHED`); on retry, check this table and resume/skip already-completed steps instead of re-running blindly.
  - For staged files, load into a **temp/staging table scoped to the run**, then `MERGE` into the final table and only then mark the control table `SUCCESS` — if the job dies mid-way, the staging table can simply be truncated and retried with no effect on the final table.
  - Example idempotent merge:
    ```sql
    MERGE INTO fact_orders t
    USING stg_orders_batch s
      ON t.order_id = s.order_id
    WHEN MATCHED THEN UPDATE SET t.amount = s.amount, t.updated_at = s.loaded_at
    WHEN NOT MATCHED THEN INSERT (order_id, amount, updated_at) VALUES (s.order_id, s.amount, s.loaded_at);
    ```

---

## 3. Late-arriving data

**Q: How would you process records that arrive several days after their event date while keeping historical aggregates correct?**

- Don't treat pipelines as strictly append-only for facts with late arrivals — design downstream aggregates to be **recomputable incrementally** for a trailing window (e.g., reprocess the last 7-14 days of aggregates every run, not just "today").
- Load late rows into the fact table keyed by **event_date** (not load_date) via `MERGE`, then trigger re-aggregation only for the affected `event_date` partitions.
- Maintain a `last_updated`/`load_date` audit column separate from `event_date` so you can always tell what changed and when, and detect how late data typically arrives (lateness SLA) to size the recompute window appropriately.
- For very late or out-of-SLA data, have a separate manual backfill path rather than reprocessing everything on every run.

---

## 4. Data quality

**Q: What checks would you add for schema changes, missing keys, duplicates, invalid dates, and unexpected volume changes? What should happen when a check fails?**

- **Schema**: compare incoming file/table schema against an expected schema (column count, names, types) before loading; fail fast or route to quarantine on mismatch.
- **Missing keys**: `NOT NULL`/existence checks on primary/foreign keys; percentage threshold (e.g., alert if >0.1% nulls) rather than failing on a single bad row.
- **Duplicates**: `COUNT(*) vs COUNT(DISTINCT key)` check post-load; dedupe using `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY load_ts DESC) = 1`.
- **Invalid dates**: range checks (not in the future, not before a sane minimum), type validation before casting.
- **Volume anomalies**: compare today's row count to a trailing average/stddev (e.g., z-score or simple % deviation vs 7-day average); flag rather than hard-fail since legitimate spikes happen.
- **On failure**: quarantine the offending rows (don't block the whole batch unless the failure is structural, like schema mismatch), log details to a DQ results table, alert on-call/owning team, and optionally halt downstream publish until acknowledged for severe failures.

---

## 5. Schema evolution

**Q: An upstream source adds a column and later changes a field's type. How should the pipeline detect and handle these changes?**

- **New column**: detect via schema comparison at load time; for semi-structured (VARIANT/JSON) landing zones this is often automatic — simply start selecting the new field once stable. For typed staging tables, use `ALTER TABLE ... ADD COLUMN` driven by an automated schema-diff step, defaulting new columns to NULL for historical rows.
- **Type change**: riskier — don't blindly widen in place. Detect via schema diff/monitoring, then handle by:
  - Adding a new column with the new type and backfilling/migrating, or
  - Casting defensively (`TRY_CAST`) in the transform layer and flagging rows that fail to cast under the new expected type.
- Always keep the **raw landing layer schema-flexible** (VARIANT or wide/permissive types) so upstream changes don't break ingestion — apply strict typing only in the curated layer, where you control the migration.
- Version/track schema changes (e.g., schema registry or a metadata table) so you have an audit trail of when a column appeared or changed type.

---

## 6. Backfills

**Q: How would you rerun six months of historical data without disrupting the current daily pipeline?**

- Parameterize the pipeline by `run_date`/partition rather than hardcoding "today," so the same code path can process historical dates.
- Run backfills on a **separate warehouse/compute pool** from the daily production pipeline to avoid resource contention.
- Write backfill output to a **separate/staging table or partition**, validate it, then swap or `MERGE` into the production table in one atomic step (or use partition-level swap) rather than streaming backfill writes directly into the live table.
- Chunk the backfill (e.g., month by month or day by day) to keep transactions small, make restartability easy, and avoid giant long-running jobs that are hard to recover from if interrupted.
- Confirm idempotency: the backfill should use the same `MERGE`-based idempotent pattern as the daily run so partial reruns are safe.

---

## 7. Observability

**Q: What would you log and monitor to identify failed runs, delayed data, partial loads, and unusual row counts?**

- **Per-run metadata**: run id, start/end time, duration, status, rows read/written/rejected, source file names and checksums.
- **Freshness**: timestamp of the most recent successfully loaded record vs now, alert if it exceeds SLA (data latency monitor).
- **Volume**: row counts per run compared to historical baseline (trend/anomaly detection), alert on large deviations.
- **Failures**: structured error logs with stage (ingest/validate/transform/publish), error type, and sample failing rows; integrate with alerting (PagerDuty/Slack/email) and a dashboard summarizing pipeline health.
- **Lineage/traceability**: which source file(s) produced which rows, so a bad batch can be traced and reprocessed precisely.
- In Snowflake specifically: `TASK_HISTORY`, `COPY_HISTORY`, `QUERY_HISTORY`, and `WAREHOUSE_METERING_HISTORY` views in `ACCOUNT_USAGE`/`INFORMATION_SCHEMA` feed most of this.

---

## 8. Performance

**Q: A transformation that used to finish in 20 minutes now takes two hours. How would you isolate whether the cause is source data, compute, SQL, or orchestration?**

- **Source data**: check if input volume/row count or data skew grew significantly (query source row counts over time); check for new duplicate keys causing join fan-out.
- **Compute**: check warehouse size/queueing — was the warehouse resized down, or is it now contending with more concurrent jobs (multi-cluster queueing)? Check credit/metering history for anomalies.
- **SQL**: pull Query Profile — look for spilling to local/remote disk (memory pressure), a sudden join fan-out (exploding row counts mid-plan), or loss of partition pruning (e.g., a predicate stopped being sargable after an upstream schema/type change).
- **Orchestration**: check if the job is now waiting on an upstream dependency longer (task/DAG start delay), or if parallelism changed (e.g., it used to run concurrently with other tasks and now runs serially, or vice versa causing contention).
- Approach: compare before/after Query Profiles side by side first — that single artifact usually tells you whether it's data/SQL (scan size, spill, join explosion) vs purely a scheduling/compute issue (long queue time, warehouse resize).

---

## 9. Reconciliation

**Q: How would you prove that all source records were accounted for after an ETL run?**

- Capture **source row count and/or checksum/hash** (e.g., `COUNT(*)`, `SUM` of a control total column, or a hash of key columns) at extraction time, store it in a control table.
- After load, compare: `source_count = loaded_count + rejected_count + duplicate_count` — every source row must land in exactly one bucket (loaded, rejected, or deduped-away) with a reason.
- For incremental loads, reconcile by key: `source keys MINUS target keys` should be empty (or match known rejects); do the reverse too (`target MINUS source`) to catch unexpected duplication.
- Automate this as a post-load validation step that fails/alerts the pipeline if the reconciliation doesn't balance, rather than relying on manual spot checks.

---

## 10. Trade-offs

**Q: When would you choose batch processing over streaming, and what factors would guide the decision?**

- **Choose batch** when: latency requirements are hours/day (not seconds), source systems naturally deliver data in files/batches, transformations require large joins/aggregations better suited to bulk processing, team/infra maturity favors simpler batch tooling, and cost efficiency matters more than freshness.
- **Choose streaming** when: near-real-time decisions are needed (fraud detection, live dashboards, alerting), data volume arrives continuously rather than in discrete drops, or downstream systems need to react to individual events.
- Factors: latency SLA, cost (streaming infra generally costs more to run continuously), complexity/operational overhead (exactly-once semantics, windowing, late data handling are harder in streaming), and whether the business logic actually needs per-event granularity or just periodic freshness.

---

# Combined Scenario

**A source system sends daily order files to cloud storage. Some files arrive late, may be resent, and occasionally contain malformed rows. Design a reliable pipeline that loads them into Snowflake and publishes trusted reporting tables.**

**Sample design covering all required areas:**

- **File discovery and tracking**: S3 event notification → SQS → triggers Snowpipe (or a Lambda/orchestrator) for near-real-time ingestion; maintain a `file_control` table (`file_name`, `file_hash`, `received_at`, `status`, `row_count`) keyed by file name + hash so resent/duplicate files are recognized rather than reloaded blindly.
- **Validation and rejected-row handling**: load raw files into a staging table as-is (even malformed rows land as VARIANT/text), then run validation (schema, type casts via `TRY_CAST`, required keys, date ranges) and route failures to a `rejected_rows` table with the reason, rather than failing the whole file.
- **Deduplication and safe retries**: use `MERGE` keyed on business key (e.g., `order_id` + `source_system`) rather than blind `INSERT`; resent files naturally no-op or update rather than duplicate. Use `QUALIFY ROW_NUMBER()` to pick the latest version per key if a file contains internal duplicates.
- **Late data and backfills**: load by `order_date`/event date, not arrival date; trigger re-aggregation of only the affected date partitions when late data lands, instead of a full rebuild. Track lateness distribution to tune the recompute window.
- **Snowflake warehouse and table design**: separate warehouse for ingestion/ETL vs reporting/BI to avoid contention; raw (landing, semi-structured/permissive) → staging (typed, validated) → curated/mart (clustered on `order_date` or region, analytics-ready) layering; cluster curated fact tables on commonly filtered columns.
- **Monitoring, recovery, security, and cost control**: control table + `COPY_HISTORY`/`TASK_HISTORY` for observability; alert on freshness SLA breach, rejection rate spikes, and volume anomalies; Time Travel + zero-copy clones for recovering from bad loads without disrupting valid later data; RBAC with masking on sensitive order/customer fields; resource monitors and auto-suspend warehouses, with the summary/reporting layer materialized so BI tools don't rescan raw data, to control cost.

---

# Practical Exercises

## P1. Write the idempotent incremental load

Given `stg_orders(order_id, customer_id, amount, updated_at)` and target `fact_orders`, write a `MERGE` that is safe to run multiple times for the same batch, including soft-deletes (`is_deleted` flag in staging).

<details>
<summary>Reference answer</summary>

```sql
MERGE INTO fact_orders t
USING stg_orders s
  ON t.order_id = s.order_id
WHEN MATCHED AND s.is_deleted = TRUE THEN DELETE
WHEN MATCHED AND s.is_deleted = FALSE AND s.updated_at > t.updated_at
  THEN UPDATE SET t.amount = s.amount, t.updated_at = s.updated_at
WHEN NOT MATCHED AND s.is_deleted = FALSE
  THEN INSERT (order_id, customer_id, amount, updated_at)
  VALUES (s.order_id, s.customer_id, s.amount, s.updated_at);
```
Note the `updated_at` guard on the UPDATE branch — without it, an out-of-order retry could overwrite a newer row with stale data.
</details>

---

## P2. Detect and quarantine duplicates in a single query

Given `stg_orders(order_id, amount, loaded_at)` that may contain duplicate `order_id`s from a resent file, write a query that returns the deduplicated rows (latest `loaded_at` wins) and a separate query that returns only the rows being discarded, for audit.

<details>
<summary>Reference answer</summary>

```sql
-- Keep latest per order_id
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY loaded_at DESC) AS rn
  FROM stg_orders
) QUALIFY rn = 1;

-- Discarded duplicates (for audit)
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY loaded_at DESC) AS rn
  FROM stg_orders
) QUALIFY rn > 1;
```
</details>

---

## P3. Reconciliation query

Given `source_file_log(file_name, source_row_count)` and `fact_orders(order_id, source_file, ...)` plus `rejected_rows(order_id, source_file, reason)`, write a query that flags any file where `source_row_count <> loaded + rejected`.

<details>
<summary>Reference answer</summary>

```sql
SELECT s.file_name,
       s.source_row_count,
       COALESCE(f.loaded_count, 0) AS loaded_count,
       COALESCE(r.rejected_count, 0) AS rejected_count,
       s.source_row_count - (COALESCE(f.loaded_count,0) + COALESCE(r.rejected_count,0)) AS diff
FROM source_file_log s
LEFT JOIN (SELECT source_file, COUNT(*) AS loaded_count FROM fact_orders GROUP BY source_file) f
  ON f.source_file = s.file_name
LEFT JOIN (SELECT source_file, COUNT(*) AS rejected_count FROM rejected_rows GROUP BY source_file) r
  ON r.source_file = s.file_name
WHERE s.source_row_count <> COALESCE(f.loaded_count,0) + COALESCE(r.rejected_count,0);
```
</details>

---

## P4. Clustering key trade-off

A 12 TB `fact_events` table is loaded continuously and queried mostly by `event_date` (7-day ranges) and occasionally by `customer_id` (point lookups). `customer_id` is near-unique. Would you cluster on `customer_id`, `event_date`, or both? Why?

<details>
<summary>Reference answer</summary>

Cluster on `event_date` (possibly with a coarser expression like `DATE_TRUNC('day', event_ts)`), not `customer_id`. Clustering on a near-unique, high-cardinality column causes constant reclustering (every new row is "out of order" relative to existing micro-partitions) at high credit cost for little pruning benefit on point lookups, which are rare. `event_date` aligns with natural load order and the dominant query pattern (range scans), giving strong pruning with low reclustering overhead. Point lookups on `customer_id` can be served fine by the normal scan plus Snowflake's automatic micro-partition metadata, or a secondary structure (e.g., a search optimization service) if lookups become frequent — not by clustering the whole table on it.
</details>

---

## P5. Backfill without disrupting the daily pipeline

Describe the exact steps (and a sample statement) to backfill `fact_orders` for `2026-01-01` to `2026-06-30` into a table that is also being loaded daily for the current date.

<details>
<summary>Reference answer</summary>

1. Run the transform logic parameterized by date range into a **separate staging table**, e.g. `fact_orders_backfill`, on a separate warehouse from the daily production load.
2. Validate row counts/reconciliation against source for the backfilled range.
3. Merge into production using the date range as a partition boundary so the daily job's concurrent writes (for the current date, outside the backfill range) are never touched:
   ```sql
   MERGE INTO fact_orders t
   USING fact_orders_backfill s
     ON t.order_id = s.order_id
   WHEN MATCHED THEN UPDATE SET t.amount = s.amount, t.updated_at = s.updated_at
   WHEN NOT MATCHED THEN INSERT (order_id, customer_id, amount, order_date, updated_at)
   VALUES (s.order_id, s.customer_id, s.amount, s.order_date, s.updated_at);
   ```
4. Because the backfill range (`Jan-Jun`) and the daily load range (`today`) don't overlap, this `MERGE` can run concurrently with the daily pipeline without lock contention on the same business keys.
</details>
