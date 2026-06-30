# SQL Cheatsheet — PostgreSQL

> **Legend:** Lines marked `-- [PG]` are PostgreSQL-specific. Everything else is standard SQL.

---

## 1. Basic Queries

```sql
SELECT * FROM employees;                                  -- all columns
SELECT fname, salary FROM employees;                      -- specific columns
SELECT DISTINCT dept_id FROM employees;                   -- remove duplicates

-- Filtering
SELECT * FROM employees WHERE salary > 80000;
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 80000;  -- inclusive on both ends
SELECT * FROM employees WHERE superior_emp_id IS NULL;          -- check for NULL
SELECT * FROM employees WHERE superior_emp_id IS NOT NULL;

-- Pattern matching
SELECT * FROM employees WHERE fname LIKE 'A%';      -- starts with A  (% = any chars)
SELECT * FROM employees WHERE fname LIKE '_ohn';    -- _ = exactly one char
SELECT * FROM employees WHERE fname ILIKE 'alice%'; -- [PG] case-insensitive LIKE

-- IN list (much cleaner than multiple OR conditions)
SELECT * FROM employees WHERE status IN ('Active', 'Pending');
SELECT * FROM employees WHERE status NOT IN ('Inactive', 'Terminated');

-- Sorting
SELECT * FROM employees ORDER BY salary DESC;
SELECT * FROM employees ORDER BY salary DESC NULLS LAST;  -- [PG] control NULL position
SELECT * FROM employees ORDER BY dept_id ASC, salary DESC; -- multi-column sort

-- Pagination
SELECT * FROM products ORDER BY price DESC LIMIT 10;           -- top 10
SELECT * FROM products ORDER BY price DESC LIMIT 10 OFFSET 20; -- [PG] page 3 (rows 21-30)
```

---

## 2. JOINs

| Type | Returns |
|------|---------|
| `INNER JOIN` | Only matching rows from both tables |
| `LEFT JOIN` | All left rows + matched right rows (NULL if no right match) |
| `RIGHT JOIN` | All right rows + matched left rows (NULL if no left match) |
| `FULL OUTER JOIN` | All rows from both tables, NULLs where no match |
| `CROSS JOIN` | Every row from left × every row from right (Cartesian product) |
| `SELF JOIN` | Table joined with itself — useful for hierarchical data |

```sql
-- INNER JOIN — only employees who have a department
SELECT e.fname, d.name AS department
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- LEFT JOIN — all employees, even those with no department
SELECT e.fname, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
-- d.name will be NULL where no department matches

-- FULL OUTER JOIN — everyone from both sides
SELECT e.fname, d.name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;

-- SELF JOIN — find each employee and their manager (both in the same table)
SELECT e.fname AS employee, m.fname AS manager
FROM employees e
LEFT JOIN employees m ON e.superior_emp_id = m.emp_id;

-- CROSS JOIN — all combinations (e.g., 3 sizes × 4 colors = 12 rows)
SELECT s.size, c.color FROM sizes s CROSS JOIN colors c;

-- [PG] DISTINCT ON — keep only the first row per group (latest order per customer)
SELECT DISTINCT ON (customer_id)
    customer_id, order_date, total
FROM orders
ORDER BY customer_id, order_date DESC;
-- Must include DISTINCT ON column(s) as first in ORDER BY
```

---

## 3. Aggregate Functions

| Function | Purpose |
|----------|---------|
| `COUNT(*)` | Count all rows (including NULLs) |
| `COUNT(col)` | Count non-NULL values in a column |
| `SUM(col)` | Total of numeric column |
| `AVG(col)` | Mean value |
| `MIN(col)` | Smallest value |
| `MAX(col)` | Largest value |
| `STRING_AGG(col, sep)` | [PG] Join values into one string with separator |
| `ROUND(val, n)` | Round to n decimal places |

```sql
SELECT COUNT(*) FROM employees;                              -- total rows
SELECT COUNT(manager_id) FROM employees;                     -- excludes NULLs
SELECT SUM(salary), AVG(salary) FROM employees;
SELECT MIN(hire_date), MAX(hire_date) FROM employees;        -- earliest / latest
SELECT ROUND(AVG(salary), 2) FROM employees;                 -- round to 2 decimals

-- [PG] STRING_AGG — concatenate names into a comma-separated list
SELECT STRING_AGG(fname, ', ' ORDER BY fname) AS all_names FROM employees;
```

---

## 4. GROUP BY & HAVING

```sql
-- GROUP BY — collapse rows into groups, then aggregate each group
SELECT dept_id, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM employees
GROUP BY dept_id;

-- Every column in SELECT must either be in GROUP BY or inside an aggregate function

-- GROUP BY + JOIN — group by a name instead of an ID
SELECT d.name AS department, COUNT(*) AS headcount
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.name
ORDER BY headcount DESC;

-- HAVING — filter groups AFTER aggregation (WHERE filters rows BEFORE grouping)
SELECT dept_id, COUNT(*) AS headcount
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 2;           -- only departments with more than 2 employees

-- Combining WHERE + GROUP BY + HAVING
SELECT dept_id, AVG(salary) AS avg_salary
FROM employees
WHERE hire_date >= '2020-01-01'   -- step 1: filter individual rows first
GROUP BY dept_id                   -- step 2: group the remaining rows
HAVING AVG(salary) > 50000;        -- step 3: filter groups by aggregate result
```

---

## 5. CASE WHEN

```sql
-- Ranged conditions (evaluated top to bottom, first match wins)
SELECT fname,
    CASE
        WHEN salary > 90000 THEN 'Senior'
        WHEN salary > 60000 THEN 'Mid'
        ELSE 'Junior'               -- ELSE handles anything not matched above
    END AS level
FROM employees;

-- Simple equality form (like a switch statement)
SELECT fname,
    CASE dept_id
        WHEN 1 THEN 'Engineering'
        WHEN 2 THEN 'Sales'
        ELSE 'Other'
    END AS department
FROM employees;

-- CASE inside an aggregate — conditional count per group
SELECT dept_id,
    COUNT(*)                                              AS total,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END)        AS female_count,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END)        AS male_count
FROM employees
GROUP BY dept_id;

-- [PG] FILTER — cleaner Postgres alternative to CASE inside aggregate
SELECT
    COUNT(*) FILTER (WHERE salary > 80000) AS high_earners,
    COUNT(*) FILTER (WHERE salary <= 80000) AS others
FROM employees;
```

---

## 6. NULL Handling

```sql
-- NULLs are not equal to anything — always use IS NULL / IS NOT NULL
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;

-- COALESCE — returns the first non-NULL argument
-- Useful for providing fallback/default values
SELECT COALESCE(phone, mobile, 'N/A') AS best_contact FROM customers;
SELECT COALESCE(discount, 0) AS discount FROM orders;   -- treat NULL as 0

-- NULLIF — returns NULL if both arguments are equal, otherwise the first value
-- Most common use: prevent division by zero
SELECT revenue / NULLIF(total_orders, 0) AS avg_order_value FROM sales;
-- If total_orders = 0, NULLIF returns NULL, so division becomes NULL (not an error)

-- [PG] Control NULL sort position
SELECT * FROM employees ORDER BY salary DESC NULLS LAST;  -- NULLs go to the end
SELECT * FROM employees ORDER BY salary ASC  NULLS FIRST; -- NULLs go to the front
```

---

## 7. Set Operations

> All queries must return the same number of columns with compatible types.

```sql
-- UNION — combine results and remove duplicates (slower, does a DISTINCT pass)
SELECT emp_id FROM employees WHERE title = 'Teller'
UNION
SELECT open_emp_id FROM accounts WHERE open_branch_id = 2;

-- UNION ALL — combine results and keep duplicates (faster, use when duplicates are OK)
SELECT emp_id FROM employees WHERE title = 'Teller'
UNION ALL
SELECT open_emp_id FROM accounts WHERE open_branch_id = 2;

-- INTERSECT — only rows that appear in BOTH result sets
SELECT emp_id FROM employees WHERE title = 'Teller'
INTERSECT
SELECT open_emp_id FROM accounts WHERE open_branch_id = 2;

-- EXCEPT — rows in the first result that are NOT in the second
-- [PG] Oracle calls this MINUS
SELECT emp_id FROM employees WHERE title = 'Teller'
EXCEPT
SELECT open_emp_id FROM accounts WHERE open_branch_id = 2;
```

---

## 8. Subqueries

```sql
-- Scalar subquery — returns a single value, used in WHERE
SELECT customer_id, name FROM customers
WHERE customer_id = (
    SELECT customer_id FROM accounts ORDER BY balance DESC LIMIT 1
    -- finds the customer with the highest account balance
);

-- Table subquery — used in FROM as a derived table (must be aliased)
SELECT c.name, sub.avg_amount
FROM customers c
JOIN (
    SELECT a.customer_id, AVG(t.amount) AS avg_amount
    FROM accounts a
    JOIN transactions t ON a.account_id = t.account_id
    GROUP BY a.customer_id
) sub ON c.customer_id = sub.customer_id;

-- IN — check if value exists in a subquery result list
SELECT name FROM customers
WHERE customer_id IN (
    SELECT a.customer_id FROM accounts a
    JOIN transactions t ON a.account_id = t.account_id
    WHERE t.amount > 1000
);

-- EXISTS — returns TRUE if subquery finds at least one row (faster than IN for large sets)
SELECT name FROM customers c
WHERE EXISTS (
    SELECT 1 FROM accounts a
    WHERE a.customer_id = c.customer_id AND a.balance = 0
    -- SELECT 1 is a convention — we only care if a row exists, not what it returns
);

-- Correlated subquery — inner query references the outer query's row
-- Runs once per outer row (can be slow on large tables)
SELECT name FROM customers c
WHERE (
    SELECT SUM(a.balance) FROM accounts a
    WHERE a.customer_id = c.customer_id   -- references outer 'c'
) > 5000;
```

---

## 9. CTEs (Common Table Expressions)

```sql
-- Standard CTE — named, temporary result set scoped to one query
-- Makes complex queries easier to read by breaking them into named steps
WITH CustomerTxnCounts AS (
    -- Step 1: count transactions per customer
    SELECT a.customer_id, COUNT(t.transaction_id) AS total_txns
    FROM accounts a
    JOIN transactions t ON a.account_id = t.account_id
    GROUP BY a.customer_id
),
ActiveCustomers AS (
    -- Step 2: keep only customers with more than 2 transactions
    SELECT customer_id, total_txns
    FROM CustomerTxnCounts
    WHERE total_txns > 2
)
-- Step 3: join back to get customer details
SELECT c.customer_id, c.name, ac.total_txns
FROM customers c
JOIN ActiveCustomers ac ON c.customer_id = ac.customer_id;

-- [PG] Recursive CTE — query references itself, used for hierarchical data
WITH RECURSIVE org_chart AS (
    -- Base case: start with the top-level (no manager)
    SELECT emp_id, fname, superior_emp_id, 1 AS depth
    FROM employees
    WHERE superior_emp_id IS NULL

    UNION ALL

    -- Recursive case: find direct reports of the previous level
    SELECT e.emp_id, e.fname, e.superior_emp_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.superior_emp_id = oc.emp_id
)
SELECT * FROM org_chart ORDER BY depth;
```

---

## 10. Window Functions

> Unlike `GROUP BY`, window functions return a result for **every row** without collapsing them.

```sql
-- Syntax: function() OVER (PARTITION BY ... ORDER BY ... ROWS/RANGE ...)
-- PARTITION BY = restart calculation for each group (optional)
-- ORDER BY     = defines row sequence within the partition

-- Aggregate window — total per customer shown on every transaction row
SELECT
    c.name,
    t.amount,
    SUM(t.amount) OVER (PARTITION BY c.customer_id) AS customer_total
FROM customers c
JOIN accounts a  ON c.customer_id = a.customer_id
JOIN transactions t ON a.account_id = t.account_id;

-- Ranking functions
SELECT fname, salary,
    RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rank,
    -- RANK: gaps after ties     → 1, 2, 2, 4
    DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rank,
    -- DENSE_RANK: no gaps       → 1, 2, 2, 3
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num,
    -- ROW_NUMBER: always unique → 1, 2, 3, 4
    NTILE(4)     OVER (ORDER BY salary DESC)                      AS quartile
    -- NTILE(n): splits rows into n equal buckets
FROM employees;

-- LAG / LEAD — look at previous or next row's value
SELECT
    customer_id,
    transaction_date,
    amount,
    LAG(amount, 1, 0)  OVER (PARTITION BY customer_id ORDER BY transaction_date) AS prev_amount,
    -- LAG(col, offset, default): value from 1 row before; 0 if no prior row
    LEAD(amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY transaction_date) AS next_amount
    -- LEAD(col, offset, default): value from 1 row after
FROM transactions;

-- Running total — cumulative sum up to current row
SELECT transaction_date, amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- from first row to here
    ) AS running_total
FROM transactions;

-- Moving average — average of current row + 1 before + 1 after (3-row window)
SELECT transaction_date, amount,
    AVG(amount) OVER (
        ORDER BY transaction_date
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moving_avg
FROM transactions;
```

---

## 11. String Functions

> All standard SQL unless marked `[PG]`.

```sql
UPPER('hello')                        -- 'HELLO'
LOWER('HELLO')                        -- 'hello'
INITCAP('hello world')                -- 'Hello World'    [PG]
TRIM('  hello  ')                     -- 'hello'          (removes both sides)
LTRIM('  hello')                      -- 'hello'          (left only)
RTRIM('hello  ')                      -- 'hello'          (right only)
LENGTH('hello')                       -- 5
SUBSTRING('hello' FROM 2 FOR 3)       -- 'ell'            (start pos, length)
LEFT('hello', 3)                      -- 'hel'
RIGHT('hello', 3)                     -- 'llo'
REPLACE('hello', 'l', 'r')           -- 'herro'
REVERSE('hello')                      -- 'olleh'           [PG]
LPAD('5', 3, '0')                    -- '005'             (pad left to width 3)
RPAD('hi', 5, '.')                   -- 'hi...'           (pad right to width 5)
POSITION('@' IN 'a@b.com')           -- 2                 (index of substring)
CONCAT(fname, ' ', lname)            -- 'John Doe'
fname || ' ' || lname                 -- 'John Doe'        [PG] || string concat operator
SPLIT_PART('a@b.com', '@', 2)        -- 'b.com'           [PG] split by delimiter, get nth part
REGEXP_REPLACE(phone,'[^0-9]','','g')-- digits only       [PG] remove non-numeric chars
```

---

## 12. Date / Time Functions

> All `[PG]` unless otherwise noted — date handling is very database-specific.

```sql
NOW()                                       -- [PG] current timestamp with timezone
CURRENT_DATE                                -- today's date only (standard SQL)
CURRENT_TIMESTAMP                           -- current timestamp (standard SQL, same as NOW())

-- Extracting parts of a date
EXTRACT(YEAR  FROM hire_date)              -- [PG] 2024
EXTRACT(MONTH FROM hire_date)              -- [PG] 7
EXTRACT(DAY   FROM hire_date)              -- [PG] 15
EXTRACT(DOW   FROM hire_date)              -- [PG] day of week: 0=Sun, 6=Sat
EXTRACT(HOUR  FROM NOW())                  -- [PG] current hour
DATE_PART('year', hire_date)               -- [PG] same as EXTRACT (older syntax)

-- Truncating to a period boundary
DATE_TRUNC('month', hire_date)             -- [PG] → 2024-07-01 (start of month)
DATE_TRUNC('year',  hire_date)             -- [PG] → 2024-01-01 (start of year)
DATE_TRUNC('week',  hire_date)             -- [PG] → start of ISO week (Monday)
-- Common use: group by month → GROUP BY DATE_TRUNC('month', order_date)

-- Age / difference
AGE(hire_date)                             -- [PG] interval from hire_date to today
AGE('2024-01-01', '2020-06-15')           -- [PG] interval between two dates

-- Converting between dates and text
TO_DATE('2024-07-01', 'YYYY-MM-DD')       -- [PG] text → date
TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI')      -- [PG] date → formatted text
TO_CHAR(salary, 'FM$999,999.00')          -- [PG] number → formatted text

-- Interval arithmetic
hire_date + INTERVAL '30 days'            -- [PG] add 30 days
hire_date + INTERVAL '1 year 2 months'    -- [PG] add time span
NOW() - INTERVAL '1 year'                 -- [PG] subtract time span
order_date::DATE - ship_date::DATE        -- [PG] difference in days (as integer)
```

---

## 13. Type Casting

```sql
-- [PG] :: shorthand — Postgres preferred style
SELECT '2024-01-01'::DATE;
SELECT '42'::INT;
SELECT '3.14'::NUMERIC;
SELECT NOW()::DATE;              -- strip time from timestamp → date only
SELECT price::NUMERIC(10, 2);   -- cast to exact decimal with 2 decimal places

-- Standard SQL CAST — works in all databases
SELECT CAST('42' AS INT);
SELECT CAST(hire_date AS TEXT);

-- Common use: comparing or formatting mixed types
SELECT order_id::TEXT || ' - ' || customer_name AS label FROM orders;  -- [PG]
```

---

## 14. Regex (PostgreSQL)

> Regex operators are `[PG]` — not available in standard SQL.

```sql
-- Operators
-- ~   case-sensitive match
-- !~  case-sensitive does NOT match
-- ~*  case-insensitive match           [PG]
-- !~* case-insensitive does not match  [PG]

-- Find rows that match a pattern
SELECT * FROM employees WHERE fname ~ '^A';          -- starts with A
SELECT * FROM employees WHERE fname ~* '^alice';     -- case-insensitive

-- Find invalid emails (does not match a valid email pattern)
SELECT customer_id, email FROM customers
WHERE email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';

-- Replace characters using regex
SELECT REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS digits_only
FROM customers;
-- 'g' flag = replace ALL matches (not just first)
```

---

## 15. DML — Insert / Update / Delete

```sql
-- Basic INSERT
INSERT INTO customers (name, email) VALUES ('Alice', 'alice@x.com');

-- INSERT multiple rows at once
INSERT INTO customers (name, email) VALUES
    ('Alice', 'alice@x.com'),
    ('Bob',   'bob@x.com');

-- [PG] INSERT with RETURNING — get back the generated/affected values
INSERT INTO customers (name, email) VALUES ('Alice', 'alice@x.com')
RETURNING customer_id;                          -- returns the auto-generated ID

-- [PG] UPSERT — insert if not exists, update if conflict (Postgres 9.5+)
INSERT INTO customers (customer_id, name, email) VALUES (1, 'Alice', 'alice@x.com')
ON CONFLICT (customer_id) DO UPDATE             -- conflict on the unique/PK column
    SET name  = EXCLUDED.name,                 -- EXCLUDED = the row that was rejected
        email = EXCLUDED.email;

INSERT INTO customers (customer_id, name) VALUES (1, 'Alice')
ON CONFLICT (customer_id) DO NOTHING;           -- silently skip if already exists

-- UPDATE
UPDATE employees
SET salary = salary * 1.1                       -- 10% raise
WHERE dept_id = 3;

-- [PG] UPDATE with RETURNING — see what changed
UPDATE employees SET salary = salary * 1.1 WHERE dept_id = 3
RETURNING emp_id, fname, salary;

-- DELETE
DELETE FROM orders WHERE order_date < '2020-01-01';

-- [PG] DELETE with RETURNING — confirm what was deleted
DELETE FROM orders WHERE order_id = 5
RETURNING *;
```

---

## 16. DDL — Create / Alter / Drop

```sql
-- CREATE TABLE with common constraints
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,            -- [PG] auto-increment integer
    name         VARCHAR(100) NOT NULL,          -- cannot be NULL
    email        VARCHAR(150) UNIQUE,            -- no duplicate emails
    is_active    BOOLEAN DEFAULT TRUE,           -- default value
    created_at   DATE DEFAULT CURRENT_DATE       -- auto-set to today
);

-- Add foreign key constraint (referential integrity)
ALTER TABLE orders
ADD CONSTRAINT fk_customer
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
-- Prevents orders with a customer_id that doesn't exist in customers

-- Modify table structure
ALTER TABLE customers ADD COLUMN phone VARCHAR(20);
ALTER TABLE customers DROP COLUMN phone;
ALTER TABLE customers ALTER COLUMN name TYPE TEXT;          -- [PG] change data type
ALTER TABLE customers RENAME COLUMN fname TO first_name;   -- rename column

-- Remove data
TRUNCATE TABLE customers;        -- delete all rows, keep table structure (fast)
DROP TABLE IF EXISTS customers;  -- delete the table entirely; IF EXISTS avoids error
```

---

## 17. Views & Materialized Views

```sql
-- VIEW — a saved query; runs fresh every time you SELECT from it
CREATE OR REPLACE VIEW active_customers AS
SELECT customer_id, name, email
FROM customers
WHERE is_active = TRUE;
-- Use like a table:
SELECT * FROM active_customers WHERE name ILIKE 'a%';  -- [PG]

DROP VIEW IF EXISTS active_customers;

-- [PG] MATERIALIZED VIEW — cached result stored on disk; must be refreshed manually
-- Use when the underlying query is expensive and you can tolerate slightly stale data
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(total)                      AS revenue
FROM orders
GROUP BY 1;

REFRESH MATERIALIZED VIEW monthly_revenue;  -- manually update the cache
DROP MATERIALIZED VIEW IF EXISTS monthly_revenue;
```

---

## 18. Transactions

```sql
-- A transaction groups statements so they all succeed or all fail together
-- This protects data integrity (e.g., bank transfers)

BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
    UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;
COMMIT;      -- permanently save both changes together

-- If something goes wrong, roll back everything
BEGIN;
    DELETE FROM orders WHERE customer_id = 99;
ROLLBACK;    -- undo — nothing was actually deleted

-- SAVEPOINT — rollback to a mid-transaction checkpoint without losing everything
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1, 250);
    SAVEPOINT after_insert;

    DELETE FROM customers WHERE customer_id = 99;   -- mistake
    ROLLBACK TO after_insert;                       -- undo only the DELETE

COMMIT;      -- INSERT is still committed
```

---

## 19. Indexing

```sql
-- Single column — speeds up WHERE / ORDER BY on that column
CREATE INDEX IX_Products_CategoryID ON Products(category_id);

-- Unique — enforces uniqueness + speeds up lookups
CREATE UNIQUE INDEX IX_Email ON Employees(email);

-- Composite — for queries that filter on multiple columns together
-- Rule: equality columns first → range columns last → most selective first
CREATE INDEX IX_Orders_CustomerDate ON Orders(customer_id, order_date);

-- [PG] Covering — INCLUDE extra columns so the query never hits the main table
CREATE INDEX IX_Orders_Covering
ON Orders(customer_id)
INCLUDE (order_date, total_amount);
-- Query reads everything it needs from the index itself (faster)

-- [PG] Filtered / Partial — index only a subset of rows
CREATE INDEX IX_Orders_Active
ON Orders(customer_id)
WHERE status = 'Active';
-- Smaller index, faster for queries that always filter status = 'Active'

-- Drop and rebuild
DROP INDEX IF EXISTS IX_Products_CategoryID;
REINDEX INDEX IX_TableName_ColumnName;   -- [PG] rebuilds a fragmented index
```

| Type | When to use |
|------|------------|
| **Single column** | Frequent `WHERE`, `ORDER BY`, or `JOIN` on one column |
| **Unique** | Must enforce no duplicates (emails, IDs) |
| **Composite** | Queries filter on two or more columns together |
| **Covering** | Query selects only indexed columns — zero table lookup |
| **Filtered/Partial** | Large table with a common narrow filter condition |

---

## 20. EXPLAIN ANALYZE `[PG]`

```sql
-- Shows the query execution plan AND actual runtime statistics
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

-- Key terms in the output:
-- Seq Scan    → full table scan (no index used — may need an index)
-- Index Scan  → index used to find rows (good)
-- Index Only Scan → covering index used — didn't touch the main table (best)
-- cost=X..Y   → estimated startup cost .. total cost (in arbitrary units)
-- rows=N      → estimated number of rows
-- actual time → real execution time in milliseconds
-- loops=N     → how many times this node was executed

-- Tip: wrap in EXPLAIN (ANALYZE, BUFFERS) for cache hit info
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 123;
```

---

## 21. GENERATE_SERIES `[PG]`

```sql
-- Generate a sequence of numbers
SELECT generate_series(1, 5) AS n;
-- Returns: 1, 2, 3, 4, 5

-- Generate a date range (monthly)
SELECT generate_series(
    '2024-01-01'::DATE,
    '2024-12-01'::DATE,
    '1 month'
) AS month;

-- Common use: fill calendar gaps — show 0 for days with no orders
SELECT
    d.day,
    COUNT(o.order_id)              AS order_count,   -- COUNT returns 0 (not NULL) when no rows match
    COALESCE(SUM(o.amount), 0)    AS total_amount    -- SUM/AVG return NULL with no rows — COALESCE needed here
FROM generate_series(
    '2024-01-01'::DATE,
    '2024-01-31'::DATE,
    '1 day'
) AS d(day)
LEFT JOIN orders o ON o.order_date = d.day
GROUP BY d.day
ORDER BY d.day;
```

---

## 22. Normalization

| Form | Rule | Eliminates |
|------|------|------------|
| **1NF** | Atomic values, no repeating groups | Multi-valued cells |
| **2NF** | 1NF + non-key columns depend on the **full** primary key | Partial dependencies |
| **3NF** | 2NF + non-key columns depend **only** on the primary key | Transitive dependencies |
| **BCNF** | 3NF + every determinant is a candidate key | Remaining anomalies |

**Anomalies that normalization prevents:**
- **Insert anomaly** — can't add new data without unrelated data existing
- **Update anomaly** — same fact stored in many rows, easy to update only some
- **Delete anomaly** — deleting a row accidentally destroys unrelated information

---

## 23. Common PostgreSQL Data Types

| Type | Notes |
|------|-------|
| `SERIAL` / `BIGSERIAL` | [PG] Auto-increment integer — use for primary keys |
| `INT` / `BIGINT` | Whole numbers (BIGINT for very large numbers) |
| `NUMERIC(p, s)` | Exact decimal — use for money (p = total digits, s = decimal places) |
| `FLOAT` / `REAL` | Approximate decimal — avoid for money |
| `VARCHAR(n)` | String with max length n |
| `TEXT` | [PG] Unlimited string — preferred over VARCHAR in Postgres |
| `BOOLEAN` | `TRUE` / `FALSE` / `NULL` |
| `DATE` | Date only — `2024-07-01` |
| `TIMESTAMP` | Date + time, no timezone |
| `TIMESTAMPTZ` | [PG] Date + time with timezone — recommended default |
| `INTERVAL` | [PG] Time span — `'1 year 2 months'` |
| `JSONB` | [PG] JSON stored as binary — supports indexing |
| `UUID` | Universally unique identifier |

---

## 24. psql Meta-Commands `[PG]`

```
\l                  list all databases
\c dbname           connect to a database
\dt                 list all tables in current schema
\d  tablename       describe table (columns, types, constraints, indexes)
\di                 list indexes
\dn                 list schemas
\df                 list functions
\timing             toggle showing query execution time
\e                  open last query in $EDITOR
\q                  quit psql
\?                  help for meta-commands
\h SELECT           syntax help for a specific SQL command
```

---

## 25. Query Execution Order

```
1. FROM          -- identify source tables
2. JOIN          -- combine tables
3. WHERE         -- filter individual rows
4. GROUP BY      -- group filtered rows
5. HAVING        -- filter groups
6. SELECT        -- compute output columns (aliases created here)
7. DISTINCT      -- remove duplicates
8. ORDER BY      -- sort (can use SELECT aliases here)
9. LIMIT/OFFSET  -- paginate
```

> **Why this matters:** You cannot use a `SELECT` alias in a `WHERE` clause because `WHERE` runs before `SELECT`. Use a CTE or subquery to work around this.
