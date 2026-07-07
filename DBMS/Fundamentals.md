# Written order

##### SELECT ->  FROM  ->  WHERE  ->  GROUP BY  ->  HAVING  ->  ORDER BY  ->  LIMIT

## Actual execution order
##### 1. FROM --- identifies tables, apply Joins

##### 2. WHERE --- filters rows (before grouping)

##### 3. GROUP BY --- group remaining rows

##### 4. HAVING --- filter groups (after grouping)

##### 5. SELECT  --- choose columns, compute expressions

##### 6. DISTINCT --- remove duplicates

##### 7. ORDER BY --- sort result

##### 8. LIMIT  --- restrict rows returned

```

SELECT SUM(salary) AS total
FROM employees
WHERE total > 50000; -- ❌ ERROR: alias not yet defined at WHERE step
```


```
SELECT SUM(salary) AS total
FROM employees
HAVING SUM(salary) > 50000; -- ✅ HAVING runs after SELECT
```

This is because WHERE is evaluated before aggregation functions like SUM() are computed.

##### Q: Why can't we use a SELECT alias in WHERE?

WHERE executes before SELECT, so the alias doesn't exist yet when WHERE is processed.

#### Example

```
SELECT salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 500000;

This is invalid because the execution order is :
1. FROM
2. WHERE <- annual_salary does not exist
3. SELECT <- alias is created here
```

```
SELECT salary * 12 AS annual_salary
FROM employees
WHERE salary * 12 > 500000;

repeat the expression in the where clause
```


#### Q: What's the difference between WHERE and HAVING?

WHERE filters individual rows before grouping. HAVING filters groups after GROUP BY. You can't use aggregate functions in WHERE.


# DDL Commands - CREATE, ALTER, DROP, TRUNCATE

#### DDL (Data Definition Language) defines the structure of your database. Changes are auto-committed in most RDBMS.

``` sql
-- CREATE with all common constraints
CREATE TABLE employees (
  emp_id    INT          PRIMARY KEY AUTO_INCREMENT,
  name      VARCHAR(100) NOT NULL,
  email     VARCHAR(100) UNIQUE,
  dept_id   INT          REFERENCES departments(dept_id),
  salary    DECIMAL(10,2) CHECK (salary > 0),
  hire_date DATE         DEFAULT CURRENT_DATE
);

-- ALTER: add / modify / drop column
ALTER TABLE employees ADD COLUMN phone VARCHAR(15);
ALTER TABLE employees MODIFY COLUMN salary DECIMAL(12,2);
ALTER TABLE employees DROP COLUMN phone;

-- DROP vs TRUNCATE vs DELETE
DROP TABLE employees;          -- removes table + structure + data
TRUNCATE TABLE employees;     -- removes all data, keeps structure, faster
DELETE FROM employees;        -- removes data row-by-row, can be rolled back
```

|Command|Removes Structure?|Rollback?|WHERE clause?|Triggers?|
|---|---|---|---|---|
|DROP|✅ Yes|❌ No (DDL)|❌ No|❌ No|
|TRUNCATE|❌ No|❌ No (DDL)|❌ No|❌ No|
|DELETE|❌ No|✅ Yes (DML)|✅ Yes|✅ Yes|

# Constraints - PK, FK, UNIQUE, CHECK, NOT NULL

##### Constraints enforce data integrity rules at the database level — not the application level.

##### Primary Key vs Unique Key?

PK: uniquely identifies a row, cannot be NULL, only one per table. UNIQUE: ensures no duplicate values, can have NULL (one NULL per column in most DB), multiple per table.

##### What is a Foreign Key?

A column (or set) that references the PRIMARY KEY of another table. Enforces referential integrity — you can't insert a value that doesn't exist in the referenced table.

##### ON DELETE CASCADE vs ON DELETE SET NULL?

CASCADE: deleting parent row also deletes all child rows. SET NULL: deleting parent sets FK column to NULL in child rows. RESTRICT: prevents delete if child rows exist.

##### Can a table have no Primary Key?

Yes, but it's bad practice. Without a PK, duplicate rows can exist and referential integrity can't be enforced.

## SELECT - Filtering with WHERE, LIKE, IN, BETWEEN

```SQL
-- Basic filtering
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 100000;
SELECT * FROM employees WHERE dept_id IN (1, 2, 3);
SELECT * FROM employees WHERE manager_id IS NULL;

-- LIKE pattern matching
SELECT * FROM employees WHERE name LIKE 'A%';     -- starts with A
SELECT * FROM employees WHERE name LIKE '%son';   -- ends with son
SELECT * FROM employees WHERE name LIKE '_a%';    -- 2nd char is 'a'

-- NULL comparisons (== never works with NULL)
SELECT * FROM employees WHERE manager_id = NULL;   -- ❌ Returns nothing!
SELECT * FROM employees WHERE manager_id IS NULL; -- ✅ Correct

-- DISTINCT
SELECT DISTINCT dept_id FROM employees; -- unique dept IDs only
```

# DML (Data Manipulation Language) - INSERT, UPDATE, DELETE

```SQL
-- INSERT single row
INSERT INTO employees (name, dept_id, salary)
VALUES ('Alice', 1, 75000);

-- INSERT multiple rows
INSERT INTO employees (name, dept_id, salary) VALUES
  ('Bob', 2, 60000),
  ('Carol', 1, 80000);

-- INSERT from SELECT
INSERT INTO archive_employees
SELECT * FROM employees WHERE hire_date < '2020-01-01';

-- UPDATE with WHERE (always use WHERE!)
UPDATE employees
SET salary = salary * 1.10
WHERE dept_id = 1;

-- DELETE with WHERE
DELETE FROM employees
WHERE emp_id = 42;
```

##### What happens if you run UPDATE without WHERE?

Every row in the table gets updated. This is a very common destructive mistake. Always use WHERE with UPDATE and DELETE.

# NULL Handling -COALESCE, NULLIF, IS NULL

##### NULL means "unknown" — not zero, not empty string. Any comparison with NULL returns NULL (not TRUE or FALSE), which is why = NULL never works.

```SQL
-- COALESCE: returns first non-NULL value
SELECT COALESCE(phone, email, 'no contact') AS contact
FROM employees;

-- NULLIF: returns NULL if both args are equal, else first arg
SELECT NULLIF(score, 0) FROM results; -- turns 0 into NULL

-- Safe division (avoid divide by zero)
SELECT revenue / NULLIF(units, 0) AS avg_price FROM sales;

-- COUNT behavior with NULL
SELECT
  COUNT(*)           -- counts all rows including NULLs
  COUNT(manager_id) -- counts only non-NULL values
FROM employees;
```


# CASE WHEN - Conditional Logic in SQL

```SQL
-- Simple CASE
SELECT name,
  CASE dept_id
    WHEN 1 THEN 'Engineering'
    WHEN 2 THEN 'Marketing'
    ELSE 'Other'
  END AS department
FROM employees;

-- Searched CASE (more flexible)
SELECT name, salary,
  CASE
    WHEN salary >= 100000 THEN 'Senior'
    WHEN salary >= 60000  THEN 'Mid'
    ELSE 'Junior'
  END AS level
FROM employees;

-- CASE inside aggregate (pivot-style)
SELECT
  SUM(CASE WHEN dept_id = 1 THEN salary ELSE 0 END) AS eng_total,
  SUM(CASE WHEN dept_id = 2 THEN salary ELSE 0 END) AS mkt_total
FROM employees;
```