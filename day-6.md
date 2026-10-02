# Day 06 - Snowflake + ETL Quick Q&A

Rapid-fire short questions with short answers — good for a quick refresher pass, not deep scenarios.

---

# Snowflake Quick Q&A

**Q1. What are the three Snowflake architecture layers?**
Storage, compute (virtual warehouses), and cloud services (auth, optimization, metadata).

**Q2. What is a micro-partition?**
An immutable, compressed columnar block of ~50-500 MB (uncompressed) that Snowflake automatically creates and tracks with min/max metadata.

**Q3. What enables partition pruning?**
Stored min/max metadata per column per micro-partition, matched against query filter predicates.

**Q4. Scale up vs scale out — one-line difference?**
Scale up = bigger warehouse for faster single queries; scale out (multi-cluster) = more clusters for handling concurrency.

**Q5. What is a result cache and how long does it last?**
A cached result for an identical query on unchanged data, reused automatically; persists up to 24 hours and resets on each reuse.

**Q6. Default Time Travel retention?**
1 day on Standard edition, up to 90 days on Enterprise+ (configurable per object).

**Q7. Time Travel vs Fail-safe?**
Time Travel is user-accessible for a set retention window; Fail-safe is a 7-day Snowflake-only recovery period after Time Travel expires, not self-service.

**Q8. What is zero-copy cloning?**
Creating a new table/schema/database that shares the same underlying micro-partitions as the source until either side changes (copy-on-write) — instant and storage-free until divergence.

**Q9. Difference between permanent, transient, and temporary tables?**
Permanent has Fail-safe; transient skips Fail-safe (cheaper, less recovery); temporary exists only for the session.

**Q10. What is a Stream?**
An object that tracks row-level change data (inserts/updates/deletes) on a table since it was last consumed.

**Q11. What is a Task?**
A scheduled (cron or chained) unit of SQL execution, often used to consume Streams for incremental processing.

**Q12. COPY INTO vs Snowpipe?**
`COPY INTO` is a manual/batch bulk load; Snowpipe is continuous, event-driven micro-batch loading (serverless, triggered by file arrival).

**Q13. What is a clustering key for?**
Hints Snowflake how to keep co-located data physically organized so pruning stays effective as a table grows or changes insert order.

**Q14. When should you NOT add a clustering key?**
When the column is near-unique/high-cardinality and DML is frequent — reclustering cost outweighs pruning benefit.

**Q15. Standard view vs secure view?**
Secure views hide the underlying query definition/logic from users and bypass some optimizations for privacy; standard views expose definition via `SHOW`/`GET_DDL`.

**Q16. What is a materialized view good for?**
Precomputing/storing an expensive aggregation that's queried often on data that changes infrequently; Snowflake auto-refreshes it.

**Q17. What's the difference between a Resource Monitor and a warehouse auto-suspend setting?**
Resource Monitor caps total credit spend (account/warehouse level) and can suspend warehouses or alert; auto-suspend just pauses an idle warehouse after N seconds of inactivity.

**Q18. What privilege is typically required to enable schema evolution on COPY INTO?**
The table must have `ENABLE_SCHEMA_EVOLUTION = TRUE` and the loading role needs the `EVOLVE SCHEMA` privilege on the table.

**Q19. How do you query a VARIANT array field?**
`LATERAL FLATTEN(input => column:path)` to explode array elements into rows.

**Q20. What is `QUALIFY` used for?**
Filtering on a window function result (e.g., `ROW_NUMBER()`) without needing a subquery wrapper.

**Q21. How do you share data without copying files?**
Secure Data Sharing — create a `SHARE`, grant privileges on objects/views, add the consumer account.

**Q22. What causes a warehouse query to "spill"?**
The working set (sorts, joins, aggregations) exceeds available memory/local SSD, forcing intermediate results to local disk or remote storage — visible in Query Profile.

**Q23. What's the difference between `ACCOUNT_USAGE` and `INFORMATION_SCHEMA` views?**
`INFORMATION_SCHEMA` is real-time but limited retention/scope (per database); `ACCOUNT_USAGE` has longer retention and account-wide scope but has some latency (minutes to hours).

**Q24. What is Dynamic Data Masking?**
A column-level policy that conditionally masks sensitive data based on the querying role, without changing the underlying stored data.

**Q25. What is a Row Access Policy?**
A security policy that filters which rows a role can see, enabling row-level security on a shared table.

**Q26. Why might `ENABLE_SCHEMA_EVOLUTION=TRUE` still not add a new column?**
The `COPY INTO` statement must also use `MATCH_BY_COLUMN_NAME` (or equivalent) and the file format/column mapping must allow it — the flag alone isn't sufficient.

**Q27. What does `SYSTEM$ESTIMATE_QUERY_ACCELERATION` do?**
Estimates potential runtime improvement if the Query Acceleration Service offloads part of a scan-heavy query at a given scale factor, before you enable it.

**Q28. What is the main risk of over-provisioning warehouse size?**
Wasted credits — warehouse billing is per-second based on size regardless of whether the query needs that much compute.

**Q29. How do you avoid duplicate rows when a Snowpipe file is reprocessed?**
Snowpipe has built-in load history deduplication per file (same file name + checksum within 64 days won't reload) — don't rely on it beyond that window; still recommend an idempotent downstream `MERGE`.

**Q30. What's a quick way to check if a query benefited from pruning?**
Query Profile → table scan node → compare "Partitions scanned" vs "Partitions total."

---

# ETL Quick Q&A

**Q1. What does "idempotent" mean in an ETL context?**
Running the same job/batch multiple times produces the same end state as running it once — no duplicates or side effects from retries.

**Q2. Why prefer `MERGE` over blind `INSERT` for loads?**
`MERGE` keyed on a business key naturally handles retries/reprocessing without creating duplicate rows.

**Q3. What's the difference between full load and incremental load?**
Full load reloads the entire dataset each run; incremental load processes only new/changed records since the last successful run.

**Q4. What is CDC (Change Data Capture)?**
A technique for identifying and capturing only the rows that changed (insert/update/delete) in a source system since the last capture.

**Q5. Why should raw/landing data be immutable?**
It preserves a reliable source of truth for replay/reprocessing if downstream transformation logic has a bug.

**Q6. What's the medallion/layered pattern (bronze/silver/gold or raw/staging/curated)?**
Raw layer keeps unmodified source data; staging/silver applies cleaning and typing; curated/gold applies business logic for consumption.

**Q7. What is schema-on-read vs schema-on-write?**
Schema-on-read applies structure only when querying (e.g., VARIANT/JSON); schema-on-write enforces structure at load time (typed tables).

**Q8. How do you handle late-arriving data without breaking historical aggregates?**
Key facts by event date, `MERGE` late rows in, and recompute only the affected date partitions' aggregates.

**Q9. What is a dead-letter/quarantine pattern?**
Routing rows that fail validation to a separate table/location with an error reason, instead of failing the entire batch.

**Q10. What three numbers should always reconcile after a load?**
Source row count = loaded count + rejected count + deduped count.

**Q11. Why avoid hardcoding "today" in a pipeline?**
It prevents the same code from being reused for backfills/reprocessing of historical dates; parameterize by run date instead.

**Q12. What's the risk of writing backfill data directly into a live production table?**
Lock contention / inconsistent reads with the concurrently running daily pipeline; prefer a staging table merged in atomically.

**Q13. What should a pipeline log at minimum for observability?**
Run id, start/end time, status, rows read/written/rejected, and source file identifiers.

**Q14. What's a freshness/SLA monitor?**
An alert that fires when the time since the last successful load exceeds an agreed threshold, catching silent pipeline staleness.

**Q15. How do you detect a stale/stuck pipeline that isn't technically "failed"?**
Monitor data freshness (max loaded/event timestamp vs now) and watch for suspiciously flat/unchanged metrics, not just job exit codes.

**Q16. What's the difference between batch and streaming processing?**
Batch processes data in discrete chunks on a schedule; streaming processes each event continuously with low latency.

**Q17. When would you choose streaming over batch?**
When decisions need to be made in near real time (fraud detection, live alerting) and the operational complexity/cost is justified.

**Q18. What is exactly-once vs at-least-once delivery?**
At-least-once may redeliver the same event (requiring idempotent handling); exactly-once guarantees no duplicates but is harder/costlier to guarantee end-to-end.

**Q19. How do you handle a source schema adding a new column?**
Keep the raw/landing layer flexible (semi-structured) so it isn't broken, then explicitly add/map the column in the typed curated layer.

**Q20. How do you handle a source field changing data type?**
Detect via schema diffing, use defensive casting (`TRY_CAST`), and migrate the typed column deliberately rather than auto-widening blindly.

**Q21. What's a good way to prevent duplicate loads from a resent file?**
Track processed files by name + checksum/hash in a control table, and use a `MERGE` keyed on business key as a second safety net.

**Q22. What is data lineage and why does it matter?**
Tracking which source records/files produced which downstream rows — critical for tracing and reprocessing a bad batch precisely.

**Q23. How would you validate row counts without comparing whole datasets?**
Use control totals: row counts, sum/hash of key columns, or checksums computed at extraction and compared after load.

**Q24. What's the main risk of silently widening an `INNER JOIN` to a `LEFT JOIN` to "fix" missing rows?**
It can mask a real data quality problem (e.g., a broken foreign key or type mismatch) instead of surfacing and fixing the root cause.

**Q25. Why separate compute (warehouse) for ETL vs BI/reporting?**
Avoids resource contention/noisy-neighbor issues and allows independent scaling, auto-suspend, and cost attribution per workload.

**Q26. What's the purpose of a control/audit table in orchestration?**
Tracking per-run state (started/loaded/validated/published) so retries can resume safely instead of blindly rerunning from scratch.

**Q27. How do you decide a retraining/refresh cadence for a daily aggregate table?**
Balance how fast the underlying data/business pattern changes against compute cost and how much new data accumulates between refreshes.

**Q28. What's a common cause of a transformation job going from 20 minutes to 2 hours?**
Check in order: source volume/skew change, loss of partition pruning (predicate stopped being sargable), warehouse/compute contention, or a join fan-out from new duplicate keys upstream.

**Q29. Why version/track metric definitions over time?**
So historical trend comparisons remain valid and discontinuities from definition changes are explainable, not mistaken for real shifts.

**Q30. What's the fastest way to confirm whether a performance regression is data vs query vs compute?**
Compare before/after Query Profiles — spill, scan size, and pruning stats usually reveal whether it's a data/SQL change or a pure compute/queueing issue.
