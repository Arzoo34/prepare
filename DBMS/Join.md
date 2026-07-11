#  JOINs In Depth

> The most tested SQL topic — know every type and when to use each.

## JOIN Types

| JOIN Type | Returns | Unmatched Rows? |
|-----------|---------|-----------------|
| **INNER JOIN** | Only rows matching in **both** tables | ❌ Dropped from both sides |
| **LEFT JOIN** | All left rows + matching right rows | ✅ Left rows kept (`NULL` for right) |
| **RIGHT JOIN** | All right rows + matching left rows | ✅ Right rows kept (`NULL` for left) |
| **FULL OUTER JOIN** | All rows from both tables | ✅ Both sides kept with `NULL`s |
| **CROSS JOIN** | Cartesian product (`m × n` rows) | N/A |
| **SELF JOIN** | Table joined with itself | Depends on JOIN type used |

---

# INNER JOIN

Returns only the rows that have matching values in **both** tables.

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d
ON e.dept_id = d.dept_id;
```

---

# LEFT JOIN

Returns **all rows from the left table** and matching rows from the right table.

If there is no match, the right-side columns become `NULL`.

```sql
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d
ON e.dept_id = d.dept_id;
```

---

# Find Employees Without a Department

A common interview question.

```sql
SELECT e.name
FROM employees e
LEFT JOIN departments d
ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```

> **Why does this work?**
>
> Employees with no matching department get `NULL` values from the right table after the `LEFT JOIN`.

---

# SELF JOIN

Used when a table references itself.

Example: Find each employee's manager.

```sql
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m
ON e.manager_id = m.emp_id;
```

---

# 3-Table JOIN

Join multiple tables together.

```sql
SELECT
    e.name AS employee,
    d.dept_name AS department,
    l.city AS location
FROM employees e
JOIN departments d
ON e.dept_id = d.dept_id
JOIN locations l
ON d.location_id = l.location_id
WHERE l.country = 'India'
ORDER BY d.dept_name;
```

---

# JOIN Summary

| JOIN | Keeps All Left Rows | Keeps All Right Rows | Matching Rows |
|------|:-------------------:|:--------------------:|:-------------:|
| INNER | ❌ | ❌ | ✅ |
| LEFT | ✅ | ❌ | ✅ |
| RIGHT | ❌ | ✅ | ✅ |
| FULL OUTER | ✅ | ✅ | ✅ |
| CROSS | N/A | N/A | Every possible pair |
| SELF | Depends on JOIN used | Depends | Depends |

---

# Set Operations

Set operations combine results from **two SELECT statements**.

> **Rules**
>
> - Both queries must return the **same number of columns**
> - Corresponding columns should have **compatible data types**

---

# UNION

Combines results and removes duplicates.

```sql
SELECT name
FROM employees

UNION

SELECT name
FROM contractors;
```

---

# UNION ALL

Combines results and keeps duplicates.

```sql
SELECT name
FROM employees

UNION ALL

SELECT name
FROM contractors;
```

> **Faster than `UNION`** because it doesn't perform duplicate removal.

---

# INTERSECT

Returns only rows present in **both** queries.

```sql
SELECT product_id
FROM orders_2023

INTERSECT

SELECT product_id
FROM orders_2024;
```

---

# EXCEPT (or MINUS in Oracle)

Returns rows present in the first query but **not** in the second.

```sql
SELECT customer_id
FROM all_customers

EXCEPT

SELECT customer_id
FROM active_customers;
```

---

# Set Operations Summary

| Operation | Meaning | Removes Duplicates? |
|-----------|---------|:-------------------:|
| **UNION** | A ∪ B | ✅ Yes |
| **UNION ALL** | A + B | ❌ No |
| **INTERSECT** | A ∩ B | ✅ Yes |
| **EXCEPT / MINUS** | A − B | ✅ Yes |

---

# Interview Tips

> ✅ Use **INNER JOIN** when only matching records are needed.
>
> ✅ Use **LEFT JOIN** when all rows from the left table must be preserved.
>
> ✅ Use **RIGHT JOIN** when all rows from the right table must be preserved.
>
> ✅ Use **FULL OUTER JOIN** when you need every row from both tables.
>
> ✅ Use **CROSS JOIN** to generate all possible combinations.
>
> ✅ Use **SELF JOIN** for hierarchical data like employees and managers.
>
> ✅ Prefer **UNION ALL** over **UNION** when duplicate removal isn't required for better performance.