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

Will this query behave like a LEFT JOIN or INNER JOIN? Explain the execution flow.

---

## SQL Question 2

### Query

```sql
SELECT e.*
FROM employees e
LEFT JOIN departments d
ON e.id = d.emp_id
WHERE d.emp_id IS NULL;
```

### Question

What business scenario does this query solve?

---

## SQL Question 3

### Query

```sql
SELECT *
FROM employees e
INNER JOIN departments d
ON e.id = d.emp_id;
```

### Question

Can the result contain more rows than the employee table? Explain with reasoning.

---

## SQL Question 4

### Query

```sql
SELECT COUNT(*),
       COUNT(d.emp_id)
FROM employees e
LEFT JOIN departments d
ON e.id = d.emp_id;
```

### Question

Explain the difference between both counts and why their values may differ.

---

## SQL Question 5

### Query A

```sql
SELECT *
FROM employees e
WHERE NOT EXISTS (
    SELECT 1
    FROM departments d
    WHERE d.emp_id = e.id
);
```

### Query B

```sql
SELECT *
FROM employees
WHERE id NOT IN (
    SELECT emp_id
    FROM departments
);
```

### Question

Which approach is safer and why?

---

# Python (5 Questions)

## Python Question 1

What is the output?

```python
def add_item(value, my_list=[]):
    my_list.append(value)
    return my_list

print(add_item(1))
print(add_item(2))
print(add_item(3))
```

---

## Python Question 2

What is the output?

```python
a = [1, 2, 3]
b = a

b.append(4)

print(a)
print(b)
```

Explain why.

---

## Python Question 3

What is the output?

```python
x = "hello"

def test():
    print(x)

test()
```

What Python concept is being demonstrated?

---

## Python Question 4

What is the output?

```python
data = {
    "A": 10,
    "B": 20,
    "A": 30
}

print(data)
```

Explain what happens when duplicate keys are defined.

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

Write Python code to keep only the latest entry for each user.

---

# Snowflake (5 Questions)

## Snowflake Question 1

Explain Snowflake's three-layer architecture and the responsibility of each layer.

---

## Snowflake Question 2

What is a Virtual Warehouse?

Discuss:

- Compute Separation
- Scaling Up
- Scaling Out

---

## Snowflake Question 3

Explain Micro-Partitions.

How do they improve query performance?

---

## Snowflake Question 4

A table is queried frequently by:

```sql
WHERE order_date = '2024-01-01'
```

How does partition pruning help in this scenario?

---

## Snowflake Question 5

Compare the following table types:

- Permanent
- Transient
- Temporary

When would you choose each one?

---

# AWS (5 Questions)

## AWS Question 1

Design an S3-based Data Lake.

Explain:

- Raw Layer
- Cleansed Layer
- Curated Layer

---

## AWS Question 2

A team wants to analyze large datasets directly from S3.

How would you decide between:

- Athena
- Redshift
- RDS

---

## AWS Question 3

Explain the difference between:

- AWS Glue Data Catalog
- AWS Glue ETL

Provide practical use cases.

---

## AWS Question 4

How would you partition S3 data to improve Athena performance and reduce costs?

Provide an example folder hierarchy.

---

## AWS Question 5

A Glue Job needs access to S3.

How would you securely provide access without storing AWS Access Keys inside the code?

Explain the AWS services involved.

---

# Self-Evaluation Checklist

After answering each question, ask yourself:

- Did I explain the reasoning?
- Did I discuss edge cases?
- Did I mention performance implications?
- Did I mention real-world use cases?
- Could I explain this confidently in an interview?
