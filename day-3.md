# Day 03 - Data Analyst / Data Engineer Interview Practice

# SQL (5 Questions)

## 1. Running Total with Tied Timestamps

### Data

`payments(payment_id, account_id, paid_at, amount)`

| payment_id | account_id | paid_at | amount |
|---:|---:|---|---:|
| 1 | 10 | 2026-03-01 09:00 | 20 |
| 2 | 10 | 2026-03-01 09:00 | 30 |
| 3 | 10 | 2026-03-01 10:00 | 15 |

### Query

```sql
SELECT payment_id, paid_at, amount,
       SUM(amount) OVER (
         PARTITION BY account_id ORDER BY paid_at
       ) AS running_amount
FROM payments;
```

### Question

Why can the first two rows show the same running total, and how would you make the result deterministic and row-by-row?

---

## 2. Consecutive Failure Islands

### Data

`job_runs(job_id, run_date, status)`

| job_id | run_date | status |
|---:|---|---|
| 7 | 2026-03-01 | FAILED |
| 7 | 2026-03-02 | FAILED |
| 7 | 2026-03-04 | FAILED |
| 7 | 2026-03-05 | SUCCESS |
| 7 | 2026-03-06 | FAILED |

### Question

Write SQL that returns each consecutive calendar-day failure streak with its start date, end date, and length. A missing date breaks a streak.

---

## 3. Recursive Hierarchy with a Cycle

### Data

`org_edges(manager_id, employee_id)`

| manager_id | employee_id |
|---:|---:|
| 1 | 2 |
| 2 | 3 |
| 3 | 4 |
| 4 | 2 |

### Question

Write a recursive CTE that starts at employee `1`, returns each reachable employee and path, and terminates safely when the bad `2 -> 4 -> 3 -> 2` cycle is encountered.

---

## 4. Weighted Average After Aggregation

### Data

`trades(symbol, trade_date, quantity, price)`

| symbol | trade_date | quantity | price |
|---|---|---:|---:|
| X | 2026-03-01 | 10 | 100 |
| X | 2026-03-01 | 90 | 110 |
| X | 2026-03-02 | 5 | 120 |

### Question

Write SQL for daily volume-weighted average price. Explain why `AVG(price)` is wrong and how you would prevent divide-by-zero for zero-net-quantity days.

---

## 5. Late-Arriving SCD Type 2 Record

### Tables

```text
customer_history(customer_id, tier, valid_from, valid_to)
incoming(customer_id, tier, effective_at)
```

Existing row: `(42, 'GOLD', '2026-01-01', '9999-12-31')`  
Incoming late row: `(42, 'SILVER', '2025-11-01')`

### Question

Describe the transactional SQL changes required to insert the late record without overlaps or gaps while preserving the current `GOLD` row.

---

# Python (5 Questions)

## 1. Streaming Grouping Bug

### Code

```python
from itertools import groupby

rows = [
    {"region": "E", "amount": 10},
    {"region": "W", "amount": 20},
    {"region": "E", "amount": 30},
]

totals = {
    key: sum(row["amount"] for row in group)
    for key, group in groupby(rows, key=lambda row: row["region"])
}
print(totals)
```

### Question

What is printed, why is the eastern total wrong, and how would you fix it while keeping memory usage appropriate for a large file?

---

## 2. CSV with an Embedded Newline

### Data

```csv
id,comment
1,"first line
second line"
2,ok
```

### Code

```python
import csv

with open("input.csv") as file:
    rows = list(csv.DictReader(file))
```

### Question

What platform-dependent parsing problem can occur, and what `open` arguments should a production ingestion job use?

---

## 3. Exact Money Calculation

### Code

```python
from decimal import Decimal, ROUND_HALF_UP

prices = [0.10, 0.20, 2.675]
total = sum(Decimal(value) for value in prices)
print(total.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP))
```

### Question

Why can this produce an unexpected monetary result? Correct the conversion while preserving source precision and state where rounding should occur.

---

## 4. One-Pass File Reconciliation

### Input

Two sorted iterators yield `(business_key, amount)` from files too large for memory. Keys may be missing from either file, but each key appears at most once per file.

### Question

Implement a lazy Python function that emits missing keys and amount mismatches in one pass. State its time and auxiliary-space complexity.

---

## 5. Silent Truncation in `zip`

### Code

```python
ids = [101, 102, 103]
amounts = [9.5, 8.0]
rows = list(zip(ids, amounts))
```

### Question

Why is this dangerous in an ETL validation step, and what modern Python change makes unequal lengths fail immediately?

---

# pandas (5 Questions)

## 1. Missing Group Disappears

### Code

```python
import pandas as pd

df = pd.DataFrame({
    "region": ["E", None, "E", "W"],
    "sales": [10, 50, 20, 30],
})
print(df.groupby("region")["sales"].sum())
```

### Question

Why does the output total only `60` instead of `110`? Return all groups, including the missing region, without filling the source column.

---

## 2. DST Fall-Back Duplication

### Code

```python
import pandas as pd

times = pd.to_datetime([
    "2026-11-01 00:30",
    "2026-11-01 01:30",
    "2026-11-01 01:30",
    "2026-11-01 02:30",
])
localized = times.tz_localize("US/Eastern")
```

### Question

Why does localization fail? Produce timezone-aware values that distinguish both `01:30` occurrences, then convert them to UTC.

---

## 3. Unequal Multi-Column Explode

### Code

```python
import pandas as pd

df = pd.DataFrame({
    "order_id": [1, 2],
    "items": [["A", "B"], ["C"]],
    "qty": [[1], [2]],
})
df.explode(["items", "qty"])
```

### Question

Why does this fail? Define a validation policy and reshape the data without silently assigning a quantity to the wrong item.

---

## 4. Resample Boundary Leakage

### Data

```python
events = pd.DataFrame({
    "ts": pd.to_datetime([
        "2026-03-01 09:59:59",
        "2026-03-01 10:00:00",
        "2026-03-01 10:59:59",
    ]),
    "value": [1, 2, 3],
}).set_index("ts")
```

### Question

Create hourly sums labeled by the interval end for intervals `(09:00, 10:00]`, `(10:00, 11:00]`, and explain the required `closed` and `label` settings.

---

## 5. Unobserved Categories Inflate Output

### Data

```python
df = pd.DataFrame({
    "region": pd.Categorical(["E", "E"], categories=["E", "W"]),
    "channel": pd.Categorical(["web", "store"], categories=["web", "store", "partner"]),
    "sales": [10, 20],
})
```

### Question

Write a grouped sales summary that returns only combinations present in the data. Explain the memory risk of returning every category combination in a large cube.

---

# Snowflake (5 Questions)

## 1. Schema Evolution Did Not Add a Column

A Parquet pipeline uses `COPY INTO raw_events` and receives a new top-level column `device_type`, but the table remains unchanged. `ENABLE_SCHEMA_EVOLUTION` is already `TRUE`.

### Question

Name the two other required load conditions, the privilege required by the loading role, and where you would audit the recorded evolution.

---

## 2. Clustering Key Cost Trap

A 12 TB event table receives hourly `MERGE` operations. Most queries filter on a 7-day range of `event_ts`; an engineer proposes `CLUSTER BY (event_ts, event_id)`, where both values are nearly unique. Reclustering credits and retained storage are rising.

### Question

Choose a better clustering expression and justify it using cardinality, pruning, DML frequency, and Time Travel storage effects.

---

## 3. Query Acceleration Decision

A medium warehouse runs one daily scan of 8 TB with a selective filter and aggregation. It takes 300 seconds. `SYSTEM$ESTIMATE_QUERY_ACCELERATION` estimates 152 seconds at scale factor `2` and 120 seconds at `8`; the SLA is 150 seconds.

### Question

What scale factor would you test first, and which Account Usage metrics would you compare to prove SLA improvement without accepting uncontrolled serverless cost?

---

## 4. Data Quality Check Design

`fact_orders` receives 20 million rows hourly. The SLA requires freshness under 90 minutes, fewer than 0.1% NULL `customer_id` values, and no duplicate `order_id` values.

### Question

Design Snowflake data quality checks using system or custom data metric functions, expectations, and a schedule. How would you monitor violations and serverless credit use?

---

## 5. Safe Recovery from a Bad Batch

At 10:00 a `MERGE` incorrectly updated 40 million rows in a 3 TB production table. Valid writes continued until 10:20, so replacing the whole table with its 09:59 state would lose good data.

### Question

Design a recovery using Time Travel and affected business keys that preserves valid later writes. Include validation and cutover steps.

---

# AWS (5 Questions)

## 1. Athena Iceberg Small-File Degradation

An Iceberg table receives 5-minute micro-batches and now has 900,000 Parquet files averaging 3 MB plus many position-delete files. Athena queries scan the correct partitions but latency rose from 20 seconds to 4 minutes.

### Question

Which Athena maintenance statements would you run, in what order, and what retention risk must be reviewed before removing old snapshots and orphan files?

---

## 2. Kinesis Consumers Compete for Throughput

A 12-shard stream has five independent consumers. Each shard receives 1.5 MB/s. Shared consumers see roughly 1-second propagation delay and read throttling; one fraud consumer requires under 150 ms.

### Question

Would you use enhanced fan-out for all consumers or only selected ones? Quantify the throughput isolation and discuss latency and cost tradeoffs.

---

## 3. MWAA Scheduler Starvation

An `mw1.large` environment has 1,200 DAG files. DAG parsing consumes most scheduler CPU, new tasks wait 8 minutes, and DAG code changes need only appear within 10 minutes.

### Question

Which parsing interval/process settings would you tune first, what is the scheduler thread limit for this class, and which metrics would confirm improvement?

---

## 4. S3 Event Delivery Is Not Exactly Once

An S3-triggered ingestion writes one output per input key. Retries and duplicate event notifications occasionally create duplicate warehouse rows, while overwriting an existing key can produce a newer object version.

### Question

Design an idempotency key and state store using bucket, key, and version metadata. Explain when an ETag is insufficient.

---

## 5. Cross-Account Lake Access Without Public Data

Account A owns encrypted S3 data and a Glue catalog; Account B runs Athena. Data must remain private, analysts in B need table-level access, and direct long-lived IAM user keys are prohibited.

### Question

Design the IAM role assumption, Lake Formation grants, S3 bucket policy, and KMS key policy. Identify the most likely permission layer to miss during troubleshooting.