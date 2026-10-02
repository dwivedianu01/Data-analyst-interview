# Day 07 - SQL Quick Q&A

Rapid-fire short questions with short answers — good for a quick refresher pass, not deep scenarios.

---

**Q1. `WHERE` vs `HAVING` — when is each applied?**
`WHERE` filters rows before grouping/aggregation; `HAVING` filters groups after aggregation.

**Q2. What is the logical order of SQL clause execution?**
`FROM` → `JOIN` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`.

**Q3. `UNION` vs `UNION ALL`?**
`UNION` dedupes and sorts internally (slower); `UNION ALL` keeps all rows including duplicates (faster, no dedup).

**Q4. Why can `UNION` silently hide a data problem?**
If two queries accidentally produce duplicate rows that should both be kept, `UNION` removes them without warning, masking a bug.

**Q5. `RANK()` vs `DENSE_RANK()` vs `ROW_NUMBER()`?**
`ROW_NUMBER` gives unique sequential numbers even for ties; `RANK` gives the same rank to ties but skips subsequent numbers; `DENSE_RANK` gives the same rank to ties without skipping.

**Q6. When would `ROW_NUMBER()` silently drop a legitimate duplicate row?**
When used for deduplication (`QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY ts DESC) = 1`) but the partition key isn't actually unique per real-world entity — two distinct records sharing a key get collapsed into one.

**Q7. `LEAD()`/`LAG()` — what do they do?**
Access a value from a following (`LEAD`) or preceding (`LAG`) row within the same ordered partition, without a self-join.

**Q8. Window function `OVER ()` with no `PARTITION BY` — what's the scope?**
The entire result set is treated as a single partition.

**Q9. Can you use a window function result directly in `WHERE`?**
No — window functions are evaluated after `WHERE`/`GROUP BY`; you must filter in an outer query/CTE or use `QUALIFY` (where supported).

**Q10. `EXISTS` vs `IN` — which is generally safer with subqueries?**
`EXISTS` short-circuits on first match and handles `NULL`s correctly; `IN` with a subquery returning `NULL` can silently exclude all rows if used with `NOT IN`.

**Q11. Why does `NOT IN (subquery)` returning a NULL break the whole query?**
SQL's three-valued logic makes `x <> NULL` unknown rather than true/false for every row, so `NOT IN` against a list containing `NULL` returns zero rows instead of raising an error.

**Q12. Correlated subquery vs non-correlated subquery?**
A correlated subquery references a column from the outer query and re-evaluates per outer row (can be slow); a non-correlated subquery runs once independently.

**Q13. CTE vs subquery vs temp table — when would you choose each?**
CTE for readability/one-time reuse within a single query; subquery for a quick one-off filter; temp table when the intermediate result is large, reused across multiple statements, or needs its own indexes.

**Q14. Does a CTE get materialized or re-run each time it's referenced?**
Depends on the engine/optimizer — some (e.g., Postgres pre-12, Snowflake) may inline/re-evaluate a CTE referenced multiple times unless forced to materialize; don't assume it's computed once.

**Q15. What's a recursive CTE used for?**
Traversing hierarchical or graph-like data (org charts, bill of materials) by repeatedly joining the result back to itself until no new rows are produced.

**Q16. `INNER JOIN` vs `LEFT JOIN` — what changes if you move a filter on the right table from `ON` to `WHERE`?**
In a `LEFT JOIN`, a filter on the right table in `WHERE` silently turns it into an `INNER JOIN` (non-matching left rows get dropped because their right-side columns are `NULL`); keeping the filter in `ON` preserves the left rows.

**Q17. What does a self-join solve?**
Comparing rows within the same table to each other — e.g., finding employees who earn more than their manager, or pairs of events on the same day.

**Q18. `CROSS JOIN` — what does it produce and when is it legitimate?**
A full Cartesian product (every row paired with every row); legitimate for generating combinations (e.g., all dates × all stores) but a common accidental bug when a join condition is missing.

**Q19. What causes a join "fan-out" and how do you spot it?**
Joining to a table where the join key isn't unique multiplies rows unexpectedly; spot it by comparing row counts before/after the join against the expected grain.

**Q20. `COALESCE` vs `ISNULL`/`NVL`?**
`COALESCE` is ANSI-standard and accepts multiple arguments, returning the first non-null; `ISNULL`/`NVL` are vendor-specific and typically take only two arguments.

**Q21. What does `NULLIF(a, b)` do and what's a practical use?**
Returns `NULL` if `a = b`, otherwise returns `a` — commonly used to avoid divide-by-zero: `a / NULLIF(b, 0)`.

**Q22. Why does `COUNT(column)` sometimes differ from `COUNT(*)`?**
`COUNT(*)` counts all rows; `COUNT(column)` counts only non-null values in that column.

**Q23. What's a sargable predicate and why does it matter?**
A predicate the engine can use an index/pruning on directly (e.g., `col >= '2026-01-01'`); wrapping the column in a function (`WHERE YEAR(col) = 2026`) makes it non-sargable and forces a full scan.

**Q24. What's the difference between a clustered and non-clustered index (traditional RDBMS)?**
A clustered index determines the physical row order on disk (one per table); a non-clustered index is a separate structure pointing back to the row, and a table can have many.

**Q25. When can adding an index make writes slower without meaningfully helping reads?**
When the indexed column has low selectivity (few distinct values) or is rarely used in filters — the optimizer may ignore it while every `INSERT`/`UPDATE` still pays the cost of maintaining it.

**Q26. What is a composite index and why does column order matter?**
An index on multiple columns together; it's only efficiently usable for queries that filter on a left-to-right prefix of those columns, so the most selective/most commonly filtered column usually goes first.

**Q27. What does `EXPLAIN`/query plan tell you that `EXPLAIN ANALYZE` adds?**
`EXPLAIN` shows the planned/estimated execution strategy; `EXPLAIN ANALYZE` actually runs the query and shows real row counts/timings, revealing where estimates were wrong.

**Q28. What's the difference between a view and a materialized view?**
A view is a stored query re-executed on every access (always current, no storage); a materialized view stores the computed result physically and is refreshed on a schedule or trigger (faster reads, can be stale).

**Q29. What are the ACID properties in one line each?**
Atomicity (all-or-nothing), Consistency (valid state to valid state), Isolation (concurrent transactions don't corrupt each other's view), Durability (committed data survives a crash).

**Q30. What problem does an isolation level like `READ COMMITTED` still allow that `SERIALIZABLE` prevents?**
Non-repeatable reads and phantom reads — rows that change or appear/disappear if you re-query within the same transaction.

**Q31. What's a deadlock and how do most databases handle it?**
Two transactions each hold a lock the other needs; the database detects the cycle and kills one transaction (deadlock victim) to let the other proceed.

**Q32. Why avoid `SELECT *` in production queries/views?**
It breaks if upstream schema changes (column added/reordered), pulls unnecessary data/bandwidth, and defeats some optimizer column-pruning benefits.

**Q33. What's the difference between a scalar function and a window/aggregate function in `GROUP BY` context?**
A scalar function operates row-by-row independently; an aggregate collapses multiple rows into one per group; a window function computes per-row but can see the whole partition without collapsing rows.

**Q34. How do you pivot rows into columns in standard SQL without a vendor-specific `PIVOT`?**
Conditional aggregation: `SUM(CASE WHEN category = 'A' THEN amount ELSE 0 END) AS a_total`, grouped by the row key.

**Q35. What's the risk of using `OFFSET`/`LIMIT` for pagination on a large, frequently-changing table?**
Rows can shift between pages (skipped or duplicated results) and `OFFSET` gets slower on large offsets since the engine still scans/discards skipped rows; keyset/seek pagination (`WHERE id > last_seen_id`) avoids both issues.

**Q36. What does a `MERGE`/upsert statement do, and why is it preferred for idempotent loads?**
Combines insert-if-missing and update-if-exists into one atomic statement keyed on a business key, so re-running the same load doesn't create duplicates.

**Q37. Why might two decimal columns compare as unequal even when they look the same?**
Different scale/precision or floating-point storage (`FLOAT` instead of `DECIMAL`) can introduce tiny representation differences; use a fixed-precision numeric type for money and exact comparisons.

**Q38. What's the difference between `TRUNCATE`, `DELETE`, and `DROP`?**
`DELETE` removes rows (can be filtered, logged, rolled back, fires triggers); `TRUNCATE` removes all rows fast with minimal logging and resets identity/auto-increment, typically not filterable; `DROP` removes the entire table/object definition.

**Q39. What's a common reason a `GROUP BY` query errors with "column must appear in the GROUP BY clause"?**
Selecting a non-aggregated column that isn't part of the grouping key — the engine can't determine which row's value to return per group.

**Q40. How would you find duplicate rows based on a subset of columns, keeping one copy?**
`QUALIFY ROW_NUMBER() OVER (PARTITION BY key_columns ORDER BY tiebreaker) = 1` (or a correlated subquery/`DELETE ... WHERE id NOT IN (SELECT MIN(id) ...)` on engines without `QUALIFY`).
