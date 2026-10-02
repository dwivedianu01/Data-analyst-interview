# Day 10 - ETL Real-World Problems Quick Q&A

Short, practical questions on problems that actually show up running ETL pipelines in production — not textbook definitions.

---

**Q1. A load job fails halfway through a large file — what's your first move?**

Check whether the partial write already landed in the target table; if the load wasn't wrapped in a transaction/staging step, truncate or delete the partial batch before retrying so you don't end up with half a file's worth of rows plus a full reload.

**Q2. The scheduler retried a job that had actually already succeeded — how do you stop duplicate rows?**

Key the load on a business key (or file name + checksum) and use `MERGE`/upsert instead of blind `INSERT`, and check a control/audit table for `SUCCESS` status before allowing a retry to re-run the load step.

**Q3. A source API starts returning HTTP 429 (rate limited) mid-extraction — how do you handle it?**

Implement exponential backoff with jitter and respect any `Retry-After` header; checkpoint progress (last successfully pulled page/cursor) so a retry resumes instead of re-pulling everything from page 1.

**Q4. Today's CSV has an extra column that wasn't there yesterday — what breaks, and how do you handle it?**

A strict positional load (`COPY INTO` with fixed column order) either errors or silently shifts data into the wrong columns; load by column name/header mapping into a schema-flexible staging layer, and only add the column to the typed curated table deliberately.

**Q5. An upstream team renames a column without telling you — how do you catch it fast?**

Run an automated schema-diff check at the start of each pipeline run (compare incoming schema to a stored expected schema) and fail/alert before the load, rather than discovering it when a downstream report shows nulls.

**Q6. The nightly job that normally takes 20 minutes took 90 minutes last night — what are the first three things you check?**

Source row count/volume growth, whether partition pruning or an index is still being used (query plan comparison), and warehouse/compute contention or a resize that happened recently.

**Q7. Files are landing in cloud storage but Snowpipe isn't loading them — where do you look?**

Check `COPY_HISTORY`/pipe status for errors (bad file format, schema mismatch), confirm the event notification (SQS/SNS) is actually firing for that prefix, and check if the file was already loaded before under the same name (dedup by checksum).

**Q8. A downstream dashboard shows zero rows right after a pipeline run — how do you triage?**

Check the pipeline's own run log/audit table first (did it actually process rows, or did an empty/failed extract get silently treated as a "successful" no-op?), then check if a `WHERE` filter or join in the dashboard's query is excluding everything due to a type/format change upstream.

**Q9. A partner resends a file with the same name but different content — how do you avoid silently overwriting good data?**

Track processed files by name **and** content checksum/hash in a control table; treat a same-name-different-hash file as a new version requiring review rather than auto-reloading over it.

**Q10. The job succeeded but the row count looks suspiciously low — what check would have caught this automatically?**

A volume anomaly check comparing today's row count to a trailing average (e.g., alert if it drops more than X% vs the last 7 days), run as a post-load validation step before marking the run fully successful.

**Q11. A date column suddenly starts parsing in the wrong format (`MM/DD/YYYY` vs `DD/MM/YYYY`) — how do you catch it before it corrupts data?**

Validate parsed dates against a sane range (not in the future, not before a known minimum) right after casting, and prefer an explicit format string in the parser instead of relying on auto-detection, which can silently swap day/month for ambiguous dates like `03/04/2026`.

**Q12. Credentials/secrets expire in the middle of a long-running job — how do you prevent a silent partial load?**

Fail the entire run loudly on an auth error rather than catching and continuing, and keep the control/audit table in a non-`SUCCESS` state so a retry (with refreshed credentials) knows the prior attempt didn't complete.

**Q13. Two dependent jobs ran out of order because the orchestrator didn't enforce the dependency — how do you prevent it?**

Define an explicit DAG dependency (e.g., Airflow `>>` / task sensors, or `AFTER` in Snowflake Tasks) instead of relying on schedule timing alone, so the downstream job can't start until the upstream task's success state is confirmed.

**Q14. You need to reprocess one bad day from three weeks ago without touching today's pipeline — how?**

Run a parameterized backfill for that single `run_date` on a separate warehouse/compute, write to a staging table, validate, then `MERGE` only the affected date's partition into production — never write backfill output directly into the live table the daily job is also writing to.

**Q15. A transformation step suddenly throws out-of-memory/spill errors on a join that used to work fine — what do you check?**

Check for a new duplicate-key fan-out on one side of the join (row count exploding mid-plan), growth in source data volume, and whether the compute/warehouse size still matches the workload.

**Q16. Source timestamps are in UTC but your warehouse reports are showing data a day off — what's the fix?**

Store timestamps as timezone-aware (or explicitly document/convert to a single standard, usually UTC) at ingestion, and only convert to local time at the presentation/BI layer — never let a naive timestamp get silently reinterpreted in a different timezone partway through the pipeline.

**Q17. A partner's file delivery is inconsistent — sometimes missing, sometimes hours late — how do you design the pipeline around that?**

Use a freshness/SLA monitor that alerts if the expected file hasn't arrived by a cutoff time, make the pipeline able to skip a missing-file day without failing the whole DAG, and reconcile/backfill automatically once the late file does arrive.

**Q18. An upstream table was truncated and reloaded, which broke your incremental/CDC logic — how do you detect and recover?**

Detect via a sudden spike in "changes" from the stream/CDC source or a row count that doesn't match expectations; recover by treating it as a full reload trigger (fall back to a full extract) rather than trusting the incremental delta, and alert whoever owns the upstream table about the broken contract.

**Q19. Two pipelines are writing to the same table concurrently and you're seeing lock timeouts/deadlocks — how do you resolve it?**

Stagger the schedules so they don't overlap, write to separate staging tables and merge sequentially, or scope each pipeline's writes to non-overlapping partitions/keys so they don't contend for the same rows.

**Q20. A bug fix changed transformation logic last week, and now historical numbers in reports don't match older exports — how do you handle it?**

Don't silently let old and new logic coexist unflagged — recompute history under the corrected logic where feasible, document the change and date it took effect, and communicate the discontinuity to report consumers instead of letting them discover a silent number change.

**Q21. A message queue-based pipeline crashes mid-batch — how do you avoid losing or duplicating messages on restart?**

Use at-least-once delivery with an idempotent consumer (dedupe by message ID or a `MERGE` on business key) rather than relying on exactly-once semantics from the queue alone; only acknowledge/commit a message offset after it's durably written downstream.

**Q22. A file loads fine on your machine but fails in production with an encoding error — what's the usual cause?**

A BOM (byte-order mark) or non-UTF-8 encoding (e.g., Latin-1/Windows-1252) in the source file that your local tool silently tolerates but the production loader doesn't; detect/declare encoding explicitly rather than assuming UTF-8.

**Q23. Numbers computed in the pipeline don't exactly match the same numbers recomputed in a BI tool — why, and what's the fix?**

Usually floating-point vs fixed-point arithmetic differences, or different rounding points (rounding per-row vs rounding the final aggregate); standardize on a single source of truth table/metric layer instead of letting two tools recompute the same logic independently.

**Q24. An orchestration task shows "success" in Airflow/the scheduler, but no data actually landed — how do you prevent this gap?**

Don't let task success just mean "the script exited 0" — add an explicit post-load assertion (row count > 0, matches expected range) as part of the task itself, so a silent no-op extract fails the task rather than passing.

**Q25. The staging/scratch disk fills up mid-run and the job dies — how do you prevent a repeat?**

Add disk-space monitoring/alerts on the compute host, clean up staging artifacts from previous runs automatically at the start of each job, and prefer streaming/chunked processing over materializing entire large files in temp storage when possible.
