# Aggregation & Grouping

> **Purpose:** Aggregate functions summarize multiple rows into a single value. They are commonly used with `GROUP BY` to perform data analysis.

---

# Aggregate Functions

| Function | Description |
|-----------|-------------|
| `COUNT()` | Counts rows |
| `SUM()` | Adds values |
| `AVG()` | Calculates average |
| `MIN()` | Smallest value |
| `MAX()` | Largest value |

---

# Basic Aggregation

```sql
SELECT
    COUNT(*) AS total_employees,
    AVG(salary) AS average_salary,
    MAX(salary) AS highest_salary,
    MIN(salary) AS lowest_salary,
    SUM(salary) AS total_salary
FROM employees;
```

---

# GROUP BY

Groups rows having the same value into one group.

```sql
SELECT
    dept_id,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary,
    MAX(salary) AS max_salary,
    SUM(salary) AS total_cost
FROM employees
GROUP BY dept_id;
```

### Result

| dept_id | headcount | avg_salary |
|----------|-----------|------------|
| 1 | 10 | 72000 |
| 2 | 7 | 69000 |
| 3 | 5 | 81000 |

---

# HAVING

`HAVING` filters **groups** after aggregation.

```sql
SELECT
    dept_id,
    COUNT(*) AS headcount
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 5
ORDER BY headcount DESC;
```

### Execution

```
FROM
    ↓
GROUP BY
    ↓
Aggregate Functions
    ↓
HAVING
    ↓
SELECT
    ↓
ORDER BY
```

---

# WHERE vs HAVING

| WHERE | HAVING |
|--------|---------|
| Filters rows | Filters groups |
| Before GROUP BY | After GROUP BY |
| Cannot use aggregate functions | Can use aggregate functions |

Example:

```sql
SELECT
    dept_id,
    AVG(salary) AS avg_sal
FROM employees
WHERE hire_date >= '2020-01-01'
GROUP BY dept_id
HAVING AVG(salary) > 70000;
```

---

# Execution Flow

```
FROM
    ↓
WHERE
    ↓
GROUP BY
    ↓
Aggregate Functions
    ↓
HAVING
    ↓
SELECT
    ↓
ORDER BY
```

---

# GROUP BY Rule

When using `GROUP BY`, every column in the `SELECT` list must satisfy **one** of these conditions:

- Be included in the `GROUP BY` clause
- Be inside an aggregate function

✅ Valid

```sql
SELECT
    dept_id,
    COUNT(*)
FROM employees
GROUP BY dept_id;
```

❌ Invalid

```sql
SELECT
    dept_id,
    employee_name,
    COUNT(*)
FROM employees
GROUP BY dept_id;
```

Why?

```
Which employee_name should SQL return
for each department?
```

The value is **ambiguous**.

---

# Multiple Column GROUP BY

```sql
SELECT
    dept_id,
    job_title,
    COUNT(*)
FROM employees
GROUP BY dept_id, job_title;
```

Groups are created using the combination of both columns.

---

# ROLLUP

Produces hierarchical subtotals and a grand total.

```sql
SELECT
    dept_id,
    job_title,
    SUM(salary)
FROM employees
GROUP BY ROLLUP(dept_id, job_title);
```

Produces:

| dept_id | job_title |
|----------|-----------|
| Sales | Manager |
| Sales | Executive |
| Sales | NULL *(Department Total)* |
| HR | Manager |
| HR | Executive |
| HR | NULL *(Department Total)* |
| NULL | NULL *(Grand Total)* |

Hierarchy:

```
(dept, job)
      ↓
(dept)
      ↓
(grand total)
```

---

# CUBE

Produces **every possible subtotal**.

```sql
SELECT
    dept_id,
    year,
    SUM(salary)
FROM employees
GROUP BY CUBE(dept_id, year);
```

Produces:

```
(dept, year)
(dept, NULL)
(NULL, year)
(NULL, NULL)
```

Example:

| dept | year | Meaning |
|------|------|---------|
| Sales | 2024 | Sales in 2024 |
| Sales | NULL | Total Sales across all years |
| NULL | 2024 | Total of all departments in 2024 |
| NULL | NULL | Grand Total |

---

# ROLLUP vs CUBE

| Feature | ROLLUP | CUBE |
|----------|---------|------|
| Grand Total | ✅ | ✅ |
| Hierarchical Subtotals | ✅ | ✅ |
| All Possible Combinations | ❌ | ✅ |
| Best For | Reports with totals | Multi-dimensional analysis |

---

# Interview Tips

✅ `WHERE` filters rows.

✅ `HAVING` filters groups.

✅ `GROUP BY` creates groups.

✅ Every non-aggregated column in `SELECT` must appear in `GROUP BY`.

✅ `ROLLUP` creates hierarchical totals.

✅ `CUBE` creates every subtotal combination.

---

# Common Interview Questions

### Q1. Difference between WHERE and HAVING?

- `WHERE` filters rows before grouping.
- `HAVING` filters groups after aggregation.

---

### Q2. Can HAVING be used without GROUP BY?

Yes.

```sql
SELECT COUNT(*)
FROM employees
HAVING COUNT(*) > 100;
```

The entire table is treated as one group.

---

### Q3. Why can't we select a non-grouped column?

Because SQL doesn't know which value to return from that group.

---

### Q4. Which executes first: WHERE or HAVING?

`WHERE` executes before `GROUP BY`.

---

### Q5. When should ROLLUP be used instead of CUBE?

Use `ROLLUP` when you need hierarchical totals (e.g., Department → Team → Grand Total).

Use `CUBE` when you need every possible subtotal for multi-dimensional analysis.