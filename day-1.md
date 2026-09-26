## 1. SQL

Use this table structure for both SQL exercises:

```sql
CREATE TABLE orders (
    transaction_id INT PRIMARY KEY,
    user_id VARCHAR(10),
    transaction_date DATETIME,
    product_id VARCHAR(10),
    quantity INT
);
```

### Exercise 1

Find users who purchased **different products across multiple dates**. For example, U2 bought P2 on one date and P5/P6 on another. U6's repeated purchase of P21 on the same date should not qualify by itself.

| transaction_id | user_id | transaction_date | product_id | quantity |
|---:|---|---|---|---:|
| 1 | U1 | 2020-12-16 | P1 | 2 |
| 2 | U2 | 2020-12-16 | P2 | 1 |
| 3 | U1 | 2020-12-16 | P3 | 1 |
| 4 | U4 | 2020-12-16 | P4 | 4 |
| 5 | U2 | 2020-12-17 | P5 | 3 |
| 6 | U2 | 2020-12-17 | P6 | 2 |
| 7 | U4 | 2020-12-18 | P7 | 1 |
| 8 | U3 | 2020-12-19 | P8 | 2 |
| 9 | U3 | 2020-12-19 | P9 | 8 |
| 10 | U1 | 2020-12-17 | P10 | 1 |
| 11 | U2 | 2020-12-18 | P11 | 2 |
| 12 | U3 | 2020-12-20 | P12 | 1 |
| 13 | U4 | 2020-12-19 | P13 | 3 |
| 14 | U5 | 2020-12-16 | P14 | 2 |
| 15 | U5 | 2020-12-17 | P15 | 1 |
| 16 | U1 | 2020-12-18 | P16 | 4 |
| 17 | U2 | 2020-12-19 | P17 | 1 |
| 18 | U3 | 2020-12-21 | P18 | 2 |
| 19 | U4 | 2020-12-20 | P19 | 1 |
| 20 | U5 | 2020-12-18 | P20 | 3 |
| 21 | U6 | 2020-12-16 | P21 | 1 |
| 22 | U6 | 2020-12-16 | P21 | 2 |

### Exercise 2

Find users who purchased **the same product on multiple dates**. For example, U1 bought P1 on January 1 and January 3.

| transaction_id | user_id | transaction_date | product_id | quantity |
|---:|---|---|---|---:|
| 1 | U1 | 2024-01-01 | P1 | 1 |
| 2 | U1 | 2024-01-03 | P1 | 2 |
| 3 | U1 | 2024-01-03 | P2 | 1 |
| 4 | U2 | 2024-01-01 | P3 | 1 |
| 5 | U2 | 2024-01-01 | P3 | 2 |
| 6 | U2 | 2024-01-04 | P4 | 1 |
| 7 | U3 | 2024-01-01 | P5 | 2 |
| 8 | U3 | 2024-01-02 | P6 | 1 |
| 9 | U4 | 2024-01-02 | P7 | 1 |
| 10 | U4 | 2024-01-05 | P7 | 3 |
| 11 | U5 | 2024-01-03 | P8 | 1 |
| 12 | U5 | 2024-01-04 | P9 | 2 |

### Additional SQL questions

- Find users who purchased at least two different products on the same date.
- Find users who made purchases on at least three distinct dates.

## 2. Python and pandas

### What is the output? Explain why.

```python
def add_value(value, items=[]):
    items.append(value)
    return items

print(add_value(1))
print(add_value(2))
```

### What is the output? Explain what happens when a dictionary key appears more than once.

```python
rows = [
    {"user_id": "U1", "amount": 10},
    {"user_id": "U2", "amount": 20},
    {"user_id": "U1", "amount": 30},
]

latest_by_user = {row["user_id"]: row for row in rows}
print(latest_by_user["U1"])
```

Assume a pandas DataFrame named `df` has the following data:

| user_id | product_id | order_date | quantity | price | region |
|---|---|---|---:|---:|---|
| U1 | P1 | 2024-01-01 | 2 | 10.00 | East |
| U1 | P1 | 2024-01-03 | 1 | 10.00 | East |
| U2 | P2 | 2024-01-01 | 3 | 5.00 | West |
| U2 | P3 | 2024-01-01 | 1 | 20.00 | West |
| U3 | P1 | 2024-01-02 | 2 | 10.00 | East |
| U3 | P2 | 2024-01-03 | 4 | NULL | East |
| U4 | P3 | 2024-01-02 | 2 | 20.00 | South |
| U4 | P3 | 2024-01-05 | 1 | 20.00 | South |
| U5 | P2 | 2024-01-03 | 2 | 5.00 | West |
| U5 | P1 | 2024-01-04 | 1 | 10.00 | West |

### Pandas exercises

- Calculate total revenue per day, where revenue is `quantity * price`.
- Keep only the row with the latest `order_date` for each `user_id`.

## 3. Snowflake

1. What are Snowflake's main architecture layers, and what does each do?
2. What is a virtual warehouse? How does scaling up differ from scaling out?
3. What are micro-partitions, and how does partition pruning improve query performance?
4. What is the difference between permanent, transient, and temporary tables?

## 4. AWS

1. How would you design an AWS data lake using S3 for raw, cleaned, and curated data?
2. How would you choose between Athena, Redshift, and RDS for an analytics workload?
3. How do AWS Glue Data Catalog and Glue ETL differ?
4. How would you partition S3 data to improve Athena query performance and control cost?
5. How would you securely grant a data pipeline access to S3 without storing long-lived credentials?
