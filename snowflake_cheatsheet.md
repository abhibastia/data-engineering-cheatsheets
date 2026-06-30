# Snowflake Cheatsheet

## Core Concepts

| Concept | What it is |
|---|---|
| **Warehouse** | Compute engine that runs queries — you pay per second it's running |
| **Database** | Top-level container for schemas and tables |
| **Schema** | Namespace inside a database — groups related tables/views |
| **Table** | Permanent data storage |
| **View** | Saved SQL query — no data stored, computed on every query |
| **Stage** | Staging area for loading files (internal or S3/GCS/Azure) |
| **Role** | Set of privileges — controls what a user can see/do |
| **Account** | Your Snowflake instance — `ICKJVKI-CR66247` |

---

## Object Hierarchy

```
Account
└── Database (SNOWFLAKE_LEARNING_DB)
    └── Schema (PUBLIC)
        ├── Tables
        ├── Views
        ├── Stages
        └── Stored Procedures
```

---

## Warehouse Management

```sql
-- Create
CREATE WAREHOUSE my_wh
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60          -- suspend after 60s of inactivity
    AUTO_RESUME = TRUE;        -- auto-start when a query arrives

-- Use
USE WAREHOUSE COMPUTE_WH;

-- Suspend manually
ALTER WAREHOUSE COMPUTE_WH SUSPEND;

-- Resume
ALTER WAREHOUSE COMPUTE_WH RESUME;
```

### Warehouse Sizes & Cost
| Size | Credits/hour | Use case |
|---|---|---|
| X-Small | 1 | Dev, light queries |
| Small | 2 | Normal workloads |
| Medium | 4 | Heavy transforms |
| Large | 8 | Large dbt runs |
| X-Large | 16 | Massive data loads |

**Always set `AUTO_SUSPEND`** — a forgotten running warehouse burns credits fast.

---

## Database & Schema

```sql
CREATE DATABASE my_db;
CREATE SCHEMA my_db.my_schema;

USE DATABASE SNOWFLAKE_LEARNING_DB;
USE SCHEMA PUBLIC;

-- or set both at once
USE SNOWFLAKE_LEARNING_DB.PUBLIC;

-- show what's available
SHOW DATABASES;
SHOW SCHEMAS;
SHOW TABLES;
SHOW VIEWS;
```

---

## Tables

```sql
-- Create
CREATE TABLE orders (
    order_id        VARCHAR(50)     NOT NULL,
    customer_id     VARCHAR(50),
    order_status    VARCHAR(20),
    order_date      TIMESTAMP,
    total_amount    FLOAT
);

-- Create from query
CREATE TABLE fact_orders AS
SELECT * FROM staging_orders WHERE order_status = 'delivered';

-- Drop
DROP TABLE IF EXISTS fact_orders;

-- Truncate (delete all rows, keep structure)
TRUNCATE TABLE fact_orders;

-- Rename
ALTER TABLE old_name RENAME TO new_name;

-- Add column
ALTER TABLE orders ADD COLUMN delivery_days INT;
```

---

## Querying

```sql
-- Basic
SELECT * FROM orders LIMIT 10;

-- Filter
SELECT order_id, order_status
FROM orders
WHERE order_status = 'delivered'
  AND order_date >= '2018-01-01';

-- Aggregation
SELECT
    order_status,
    COUNT(*)            AS total_orders,
    SUM(total_amount)   AS revenue,
    AVG(total_amount)   AS avg_order_value
FROM orders
GROUP BY order_status
ORDER BY revenue DESC;

-- Date functions
SELECT
    DATE_PART('year', order_date)       AS order_year,
    DATE_TRUNC('month', order_date)     AS order_month,
    DATEDIFF('day', order_date, delivered_date) AS delivery_days
FROM orders;

-- Conditional
SELECT
    order_id,
    IFF(delivery_days > 15, TRUE, FALSE)    AS is_late,
    CASE
        WHEN delivery_days <= 5  THEN 'fast'
        WHEN delivery_days <= 15 THEN 'normal'
        ELSE 'slow'
    END AS delivery_bucket
FROM orders;
```

---

## Joins

```sql
-- Inner join
SELECT o.order_id, c.customer_city
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;

-- Left join (keep all orders even if no customer match)
SELECT o.order_id, p.product_category_name
FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.product_id;

-- Filtering NULLs from left join
SELECT *
FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.product_id
WHERE p.product_category_name IS NOT NULL;
```

---

## Window Functions

```sql
-- Row number per partition
SELECT
    order_id,
    customer_id,
    order_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_seq

-- Running total
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue

-- Lag/Lead (compare to previous/next row)
SELECT
    order_date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY order_date)  AS prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY order_date) AS next_day_revenue
FROM daily_sales;
```

---

## Data Types

| Type | Examples |
|---|---|
| `VARCHAR(n)` / `TEXT` | Strings — use VARCHAR for fixed max, TEXT for unbounded |
| `NUMBER(p,s)` / `FLOAT` | Numeric — NUMBER(10,2) for money, FLOAT for metrics |
| `BOOLEAN` | TRUE / FALSE |
| `DATE` | `2024-01-15` |
| `TIMESTAMP` | `2024-01-15 08:30:00` |
| `VARIANT` | Semi-structured JSON/XML |
| `ARRAY` | List of values |
| `OBJECT` | Key-value pairs |

### Type Casting
```sql
CAST(column AS FLOAT)
column::FLOAT          -- Snowflake shorthand
column::TIMESTAMP
column::DATE
column::VARCHAR
```

---

## Loading Data (COPY INTO)

```sql
-- From internal stage
CREATE STAGE my_stage;
PUT file:///local/path/data.csv @my_stage;

COPY INTO my_table
FROM @my_stage/data.csv
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

-- From S3
COPY INTO my_table
FROM 's3://my-bucket/data/'
CREDENTIALS = (AWS_KEY_ID='...' AWS_SECRET_KEY='...')
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);

-- Check load history
SELECT * FROM information_schema.load_history
WHERE table_name = 'MY_TABLE'
ORDER BY last_load_time DESC;
```

---

## Views

```sql
-- Standard view (no data stored, recomputed every query)
CREATE OR REPLACE VIEW v_delivered_orders AS
SELECT * FROM orders WHERE order_status = 'delivered';

-- Materialized view (data stored, auto-refreshed)
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT DATE_TRUNC('day', order_date) AS day, SUM(total_amount) AS revenue
FROM orders
GROUP BY 1;

-- Drop
DROP VIEW IF EXISTS v_delivered_orders;
```

---

## Roles & Access Control

```sql
-- Common roles (built-in)
ACCOUNTADMIN    -- full control, use sparingly
SYSADMIN        -- manages databases and warehouses
USERADMIN       -- manages users and roles
PUBLIC          -- default role every user has

-- Grant usage
GRANT USAGE ON DATABASE my_db TO ROLE analyst;
GRANT USAGE ON SCHEMA my_db.public TO ROLE analyst;
GRANT SELECT ON TABLE my_db.public.orders TO ROLE analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA my_db.public TO ROLE analyst;

-- Switch role in session
USE ROLE SYSADMIN;
```

---

## Time Travel

Snowflake retains historical versions of data (default 1 day, up to 90 days on Enterprise).

```sql
-- Query data as it was 1 hour ago
SELECT * FROM orders AT (OFFSET => -3600);

-- Query data as it was at a specific timestamp
SELECT * FROM orders AT (TIMESTAMP => '2024-01-15 08:00:00'::TIMESTAMP);

-- Restore a dropped table
UNDROP TABLE orders;

-- Clone a table at a point in time
CREATE TABLE orders_backup CLONE orders AT (OFFSET => -3600);
```

---

## Zero-Copy Cloning

Creates an instant copy without duplicating data — cloned object shares storage until rows diverge.

```sql
-- Clone table
CREATE TABLE orders_dev CLONE orders;

-- Clone schema
CREATE SCHEMA dev CLONE production;

-- Clone database
CREATE DATABASE dev_db CLONE prod_db;
```

Useful for: dev/test environments, pre-migration snapshots, sandboxing.

---

## Useful System Queries

```sql
-- Check currently running queries
SELECT * FROM table(information_schema.query_history_by_session())
ORDER BY start_time DESC LIMIT 10;

-- Check warehouse credit usage
SELECT * FROM table(information_schema.warehouse_metering_history(
    DATE_RANGE_START => DATEADD('day', -7, CURRENT_DATE)
));

-- List all tables in schema
SELECT table_name, row_count, bytes
FROM information_schema.tables
WHERE table_schema = 'PUBLIC'
ORDER BY bytes DESC;

-- Check column names and types
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'OLIST_ORDERS_DATASET'
ORDER BY ordinal_position;

-- Count rows
SELECT COUNT(*) FROM olist_orders_dataset;
```

---

## Snowflake + dbt Connection

```yaml
# profiles.yml
my_snowflake_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your_account_id>       # from Snowflake URL: <account>.snowflakecomputing.com
      user: <your_username>
      password: your_password
      role: ACCOUNTADMIN
      database: SNOWFLAKE_LEARNING_DB
      schema: PUBLIC                  # dbt creates models here
      warehouse: COMPUTE_WH
      threads: 1                      # parallel model runs
```

dbt models land in the specified `database.schema`. Each model becomes a table or view with the model filename as the object name.

---

## Python Connection

Two libraries — pick one based on the use case.

### SQLAlchemy (pandas-friendly)
```python
from sqlalchemy import create_engine

url = "snowflake://USER:PASSWORD@ACCOUNT/DATABASE/SCHEMA?warehouse=COMPUTE_WH&role=ACCOUNTADMIN"
engine = create_engine(url)

# Load a DataFrame into Snowflake
df.columns = [c.upper() for c in df.columns]   # Snowflake prefers uppercase
df.to_sql(name="my_table", con=engine, if_exists="replace", index=False, method="multi")
```

### snowflake-connector (SQL-first)
```python
import snowflake.connector

conn = snowflake.connector.connect(
    user="USER", password="PASSWORD", account="ACCOUNT",
    warehouse="COMPUTE_WH", database="SNOWFLAKE_LEARNING_DB",
    schema="PUBLIC", role="ACCOUNTADMIN"
)
cur = conn.cursor()
cur.execute("SELECT COUNT(*) FROM olist_orders_dataset")
print(cur.fetchone())
```

### Required packages
```bash
# SQLAlchemy approach
pip install snowflake-sqlalchemy sqlalchemy pandas

# Connector approach
pip install snowflake-connector-python
```

---

## Chunked Loading (Large Files)

Prevents memory issues by reading and writing in batches.

```python
first_chunk = True
for chunk in pd.read_csv("large_file.csv", chunksize=50000):
    chunk.columns = [c.upper() for c in chunk.columns]
    chunk.to_sql(
        name="my_table",
        con=engine,
        if_exists="replace" if first_chunk else "append",
        index=False,
        method="multi",
    )
    first_chunk = False
```

---

## Idempotent Load Pattern (Control Table)

Prevents reloading files that are already loaded — safe to re-trigger.

```sql
-- Create once
CREATE TABLE IF NOT EXISTS LOADED_FILES (
    FILE_NAME  STRING PRIMARY KEY,
    LOAD_TIMESTAMP TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Check before loading
SELECT COUNT(*) FROM LOADED_FILES WHERE FILE_NAME = 'olist_orders_dataset.csv';

-- Register after successful load
INSERT INTO LOADED_FILES (FILE_NAME) VALUES ('olist_orders_dataset.csv');
```

In Python:
```python
for file_name in files:
    result = conn.execute(text("SELECT COUNT(*) FROM LOADED_FILES WHERE FILE_NAME=:f"), {"f": file_name}).scalar()
    if result > 0:
        print(f"Skipping {file_name}, already loaded.")
        continue
    # ... load the file
```

---

## Loading via Stage (PUT + COPY INTO)

Best for large files — uploads to Snowflake internal stage, then bulk loads.

```python
# PUT from Python (snowflake-connector)
cur.execute("CREATE OR REPLACE STAGE my_stage")
cur.execute(f"PUT 'file:///local/path/data.csv' @my_stage AUTO_COMPRESS=TRUE")

cur.execute("""
    COPY INTO my_table
    FROM @my_stage/data.csv.gz
    FILE_FORMAT = (
        TYPE = CSV
        SKIP_HEADER = 1
        FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    )
    ON_ERROR = 'CONTINUE'
""")
```

`ON_ERROR = 'CONTINUE'` — skips bad rows instead of failing the entire load. Use `ON_ERROR = 'ABORT_STATEMENT'` in production to catch data quality issues.

---

## Gotchas

- **Identifiers are UPPERCASE by default** — `select order_id` works, but `SELECT * FROM "order_id"` (quoted) is case-sensitive
- **Warehouse must be running to execute queries** — use `AUTO_RESUME = TRUE` so it starts automatically
- **`AUTO_SUSPEND` is critical for cost control** — set to 60s for dev, 300s for prod
- **Time Travel retention = 1 day on Standard tier** — don't rely on it for long-term recovery
- **COPY INTO is not idempotent by default** — it tracks loaded files and skips re-loads; use `FORCE = TRUE` to reload
- **NULL in GROUP BY** — Snowflake groups all NULLs together as one row; filter them if unwanted (as we did in `fact_product_performance`)
- **Clones share storage** — cheap to create but modifying a clone starts diverging storage costs
