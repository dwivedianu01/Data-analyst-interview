# Day 09 - SQL Query Writing Practice (Questions + Answers)

Hands-on query-writing problems with sample data and a full worked SQL answer — for practicing writing queries, not just explaining concepts.

---

## Q1. Second (Nth) highest salary

### Data

`employees(emp_id, name, dept, salary)`

| emp_id | name | dept | salary |
|---:|---|---|---:|
| 1 | Ravi | IT | 90000 |
| 2 | Meena | IT | 90000 |
| 3 | Suresh | HR | 75000 |
| 4 | Divya | IT | 85000 |
| 5 | Kiran | HR | 60000 |

**Question:**

Find the second-highest distinct salary overall, without using `LIMIT`/`OFFSET` (so it also works on engines that don't support them the same way).

**Answer:**

```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

General Nth-highest version using `DENSE_RANK`:

```sql
SELECT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;
```
`DENSE_RANK` is used (not `ROW_NUMBER`) so tied top salaries (Ravi/Meena both 90000) count as one rank, correctly landing on 85000 as the second-highest distinct value.

---

## Q2. Find duplicate records

### Data

`customers(customer_id, email)`

| customer_id | email |
|---:|---|
| 1 | a@x.com |
| 2 | b@x.com |
| 3 | a@x.com |
| 4 | c@x.com |
| 5 | a@x.com |

**Question:**

Find every email that appears more than once, along with its count.

**Answer:**

```sql
SELECT email, COUNT(*) AS occurrences
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;
```

---

## Q3. Top N earners per department

### Data

`employees(emp_id, name, dept, salary)`

| emp_id | name | dept | salary |
|---:|---|---|---:|
| 1 | Ravi | IT | 95000 |
| 2 | Meena | IT | 90000 |
| 3 | Divya | IT | 90000 |
| 4 | Arjun | IT | 80000 |
| 5 | Suresh | HR | 75000 |
| 6 | Kiran | HR | 70000 |
| 7 | Nisha | HR | 65000 |

**Question:**

Return the top 2 earners per department, handling ties sensibly (don't arbitrarily drop one of two equal salaries).

**Answer:**

```sql
SELECT emp_id, name, dept, salary
FROM (
    SELECT emp_id, name, dept, salary,
           DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk <= 2;
```
Using `DENSE_RANK` means IT returns 3 rows (Ravi at rank 1, Meena **and** Divya tied at rank 2) instead of silently cutting off at 2 physical rows — call that out as a design choice in an interview (use `ROW_NUMBER` instead if the requirement is strictly "exactly 2 rows per group").

---

## Q4. Employees earning more than their manager

### Data

`employees(emp_id, name, salary, manager_id)`

| emp_id | name | salary | manager_id |
|---:|---|---:|---:|
| 1 | Alice | 100000 | NULL |
| 2 | Bob | 90000 | 1 |
| 3 | Carol | 110000 | 1 |
| 4 | Dave | 70000 | 2 |
| 5 | Eve | 95000 | 2 |

**Question:**

Find employees who earn more than their direct manager.

**Answer:**

```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```
A self-join is required because manager and employee rows live in the same table — `e` and `m` are two aliases of `employees` joined on the manager relationship.

---

## Q5. Customers who ordered in every month of a quarter

### Data

`orders(order_id, customer_id, order_date)` — orders placed across Jan, Feb, Mar 2026.

| order_id | customer_id | order_date |
|---:|---:|---|
| 1 | 1 | 2026-01-05 |
| 2 | 1 | 2026-02-10 |
| 3 | 1 | 2026-03-02 |
| 4 | 2 | 2026-01-15 |
| 5 | 2 | 2026-03-20 |
| 6 | 3 | 2026-02-01 |

**Question:**

Find customers who placed at least one order in **every** month of Q1 2026 (Jan, Feb, and Mar) — a relational-division problem.

**Answer:**

```sql
SELECT customer_id
FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2026-04-01'
GROUP BY customer_id
HAVING COUNT(DISTINCT EXTRACT(MONTH FROM order_date)) = 3;
```
Customer 1 qualifies (Jan+Feb+Mar); customer 2 doesn't (missing Feb); customer 3 doesn't (only Feb). The `HAVING` count must equal the total number of required months, not just "> 1".

---

## Q6. Longest streak of consecutive login days

### Data

`logins(user_id, login_date)`

| user_id | login_date |
|---:|---|
| 1 | 2026-03-01 |
| 1 | 2026-03-02 |
| 1 | 2026-03-03 |
| 1 | 2026-03-05 |
| 1 | 2026-03-06 |

**Question:**

Find the longest run of consecutive calendar days each user logged in.

**Answer:**

```sql
WITH grp AS (
    SELECT user_id, login_date,
           DATEDIFF(day, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date), login_date) AS island
    FROM logins
)
SELECT user_id,
       COUNT(*) AS streak_length,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end
FROM grp
GROUP BY user_id, island
ORDER BY user_id, streak_length DESC;
```
Subtracting a running row number from the date is the classic "gaps and islands" trick: dates in an unbroken consecutive run all produce the same constant `island` value, so grouping by it isolates each streak. User 1 gets a 3-day streak (Mar 1-3) and a 2-day streak (Mar 5-6).

---

## Q7. Month-over-month growth percentage

### Data

`monthly_revenue(month, revenue)`

| month | revenue |
|---|---:|
| 2026-01 | 10000 |
| 2026-02 | 12000 |
| 2026-03 | 9000 |

**Question:**

Calculate the percentage change in revenue compared to the previous month.

**Answer:**

```sql
SELECT month,
       revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
       ROUND(
         100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
         / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2
       ) AS pct_change
FROM monthly_revenue;
```
`NULLIF(..., 0)` guards against a divide-by-zero if a prior month had zero revenue; the first row naturally returns `NULL` since there's no prior month to compare.

---

## Q8. Concatenate related rows into one string per group

### Data

`order_items(order_id, product_name)`

| order_id | product_name |
|---:|---|
| 1 | Keyboard |
| 1 | Mouse |
| 2 | Monitor |
| 2 | HDMI Cable |
| 2 | Monitor |

**Question:**

Return one row per order with all distinct product names joined into a comma-separated list.

**Answer:**

```sql
SELECT order_id,
       LISTAGG(DISTINCT product_name, ', ') WITHIN GROUP (ORDER BY product_name) AS products
FROM order_items
GROUP BY order_id;
```
(`LISTAGG` is Snowflake/Oracle syntax; the equivalent is `STRING_AGG(DISTINCT product_name, ', ')` in Postgres/SQL Server, and `GROUP_CONCAT(DISTINCT product_name SEPARATOR ', ')` in MySQL.) `DISTINCT` avoids "Monitor" appearing twice for order 2.

---

## Q9. Find missing IDs in a sequence

### Data

`tickets(ticket_id)` with values `1, 2, 3, 5, 6, 9`.

**Question:**

Find which IDs are missing from an otherwise sequential range from 1 to the current max.

**Answer:**

```sql
WITH bounds AS (
    SELECT 1 AS min_id, MAX(ticket_id) AS max_id FROM tickets
),
seq AS (
    SELECT min_id AS n FROM bounds
    UNION ALL
    SELECT n + 1 FROM seq, bounds WHERE n + 1 <= max_id
)
SELECT n AS missing_id
FROM seq
WHERE n NOT IN (SELECT ticket_id FROM tickets)
OPTION (MAXRECURSION 0); -- SQL Server only; omit on engines without a recursion cap
```
Builds a complete reference sequence from 1 to the current max with a recursive CTE, then finds which expected numbers never appear in the real table — here, 4, 7, and 8.

---

## Q10. Median without a built-in `MEDIAN()` function

### Data

`scores(student_id, score)` with values `70, 75, 80, 85, 90`.

**Question:**

Compute the median score without relying on a database-specific `MEDIAN()` function.

**Answer:**

```sql
SELECT AVG(score) AS median_score
FROM (
    SELECT score,
           ROW_NUMBER() OVER (ORDER BY score) AS rn,
           COUNT(*) OVER () AS total
    FROM scores
) ranked
WHERE rn IN ((total + 1) / 2, (total + 2) / 2);
```
Works for both odd and even row counts: for an odd count both expressions point to the same single middle row (its value is simply returned); for an even count they point to the two middle rows and `AVG` averages them.

---

## Q11. Cohort retention — % of users active again the following month

### Data

`activity(user_id, activity_month)` — one row per user per month they were active.

| user_id | activity_month |
|---:|---|
| 1 | 2026-01 |
| 1 | 2026-02 |
| 2 | 2026-01 |
| 3 | 2026-01 |
| 3 | 2026-02 |
| 4 | 2026-02 |

**Question:**

For users active in January 2026, what percentage were also active in February 2026?

**Answer:**

```sql
WITH jan_users AS (
    SELECT DISTINCT user_id FROM activity WHERE activity_month = '2026-01'
),
feb_users AS (
    SELECT DISTINCT user_id FROM activity WHERE activity_month = '2026-02'
)
SELECT
    COUNT(*) AS jan_active_users,
    COUNT(f.user_id) AS retained_in_feb,
    ROUND(100.0 * COUNT(f.user_id) / COUNT(*), 2) AS retention_pct
FROM jan_users j
LEFT JOIN feb_users f ON j.user_id = f.user_id;
```
`COUNT(f.user_id)` (not `COUNT(*)`) correctly counts only the matched/non-null rows after the `LEFT JOIN`, giving retained users out of the full January base.

---

## Q12. Unpivot columns into rows

### Data

`quarterly_sales(product, q1, q2, q3, q4)`

| product | q1 | q2 | q3 | q4 |
|---|---:|---:|---:|---:|
| Widget | 100 | 150 | 130 | 170 |
| Gadget | 80 | 90 | 85 | 95 |

**Question:**

Convert the wide quarterly columns into a long/tidy format: one row per `product`, `quarter`, `sales`.

**Answer:**

```sql
SELECT product, 'Q1' AS quarter, q1 AS sales FROM quarterly_sales
UNION ALL
SELECT product, 'Q2', q2 FROM quarterly_sales
UNION ALL
SELECT product, 'Q3', q3 FROM quarterly_sales
UNION ALL
SELECT product, 'Q4', q4 FROM quarterly_sales;
```
Portable across engines without a dedicated `UNPIVOT` keyword; Snowflake/SQL Server also support a native `UNPIVOT` operator as a shorter alternative.
