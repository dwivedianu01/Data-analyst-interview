# Day 02 - Data Engineer Interview Practice

---

# SQL (5 Questions)

## SQL Question 1

### Tables

```sql
CREATE TABLE employees (
    id INT,
    name VARCHAR(50)
);

CREATE TABLE departments (
    emp_id INT,
    dept VARCHAR(50)
);
```

### Data

#### employees

| id | name |
|----|------|
| 1 | John |
| 2 | Alice |
| 3 | Bob |
| 4 | David |

#### departments

| emp_id | dept |
|---------|----------|
| 1 | IT |
| 2 | HR |
| 2 | Finance |
| 5 | IT |

### Query

```sql
SELECT e.id,
       e.name,
       d.dept
FROM employees e
LEFT JOIN departments d
ON e.id = d.emp_id
WHERE d.dept = 'IT';
```

### Question

Explain the output and whether this behaves as a LEFT JOIN or INNER JOIN.

---

## SQL Question 2

```sql
SELECT e.*
FROM employees e
LEFT JOIN departments d
ON e.id = d.emp_id
WHERE d.emp_id IS NULL;
```

### Question

What business requirement does this query solve?

---

## SQL Question 3

```sql
SELECT *
FROM employees e
INNER JOIN departments d
ON e.id = d.emp_id;
```

### Question

Can this query return more rows than the employee table? Explain.

---

## SQL Question 4

```sql
SELECT COUNT(*),
       COUNT(d.emp_id)
FROM employees e
LEFT JOIN departments d
ON e.id = d.emp_id;
```

### Question

Why might the two counts be different?

---

## SQL Question 5

Compare:

```sql
NOT EXISTS
```

and

```sql
NOT IN
```

### Question

Which approach is safer when NULL values are present and why?

---

# Python (5 Questions)

## Python Question 1

```python
def add_item(value, items=[]):
    items.append(value)
    return items

print(add_item(1))
print(add_item(2))
```

### Question

What is the output and why?

---

## Python Question 2

```python
a = [1, 2, 3]
b = a

b.append(4)

print(a)
```

### Question

What is printed and why?

---

## Python Question 3

```python
x = 10

def test():
    x = 20
    print(x)

test()
print(x)
```

### Question

Explain the output and the scope involved.

---

## Python Question 4

```python
data = {
    "id": 1,
    "id": 2,
    "id": 3
}

print(data)
```

### Question

What is the output and why?

---

## Python Question 5

Given:

```python
rows = [
    {"user_id": "U1", "amount": 100},
    {"user_id": "U2", "amount": 200},
    {"user_id": "U1", "amount": 300}
]
```

### Question

Write Python code to keep only the latest record for each user.

---

# Snowflake (5 Questions)

## Snowflake Question 1

A table contains billions of records and users frequently execute:

```sql
SELECT *
FROM orders
WHERE country = 'US'
AND order_date >= CURRENT_DATE - 30;
```

### Question

How would you optimize this table for query performance?

---

## Snowflake Question 2

Explain the differences between:

- COPY INTO
- Snowpipe

### Question

When would you choose each approach?

---

## Snowflake Question 3

A production table was accidentally updated with incorrect data.

### Question

How would you restore the table using Snowflake features?

Discuss:

- Time Travel
- Fail-safe

---

## Snowflake Question 4

A dashboard query suddenly increased from 20 seconds to 6 minutes.

### Question

What steps would you follow to troubleshoot the issue?

---

## Snowflake Question 5

Compare the following:

- Standard View
- Secure View
- Materialized View

### Question

When should each be used?

---

# AWS (5 Questions)

## AWS Question 1

A company receives 500 GB of CSV files daily.

### Question

Design an AWS-based ingestion pipeline from source system to analytics layer.

---

## AWS Question 2

Your Athena query scans 20 TB every time it runs.

### Question

How would you reduce query cost and improve performance?

---

## AWS Question 3

You need to trigger a processing job whenever a new file arrives in S3.

### Question

Which AWS services would you use and why?

---

## AWS Question 4

A Glue ETL job that normally runs in 10 minutes now takes 45 minutes.

### Question

How would you troubleshoot the problem?

---

## AWS Question 5

Your organization has:

- Development Account
- Test Account
- Production Account

### Question
