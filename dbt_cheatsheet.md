# dbt Cheatsheet

## CLI Commands

| Command | What it does |
|---|---|
| `dbt deps` | Installs packages from `packages.yml` (like `pip install`) |
| `dbt clean` | Deletes `target/` and `dbt_packages/` — fresh slate |
| `dbt compile` | Resolves Jinja → plain SQL in `target/compiled/`. No Snowflake writes |
| `dbt run` | Builds all models in dependency order |
| `dbt test` | Runs all tests defined in `schema.yml` and `tests/` |
| `dbt build` | `run` + `test` in one command (dependency-aware) |
| `dbt debug` | Checks connection to the warehouse |
| `dbt docs generate` | Generates docs site data |
| `dbt docs serve` | Serves docs at `http://localhost:8080` |

### Useful Flags
```bash
dbt run --select stg_orders                  # run a single model
dbt run --select staging.*                   # run all staging models
dbt run --select +fact_product_performance   # run model + all upstream
dbt run --select fact_product_performance+   # run model + all downstream
dbt test --select fact_product_performance   # test a single model
dbt run --full-refresh                       # force full rebuild (incremental models)
dbt run --profiles-dir .                     # use profiles.yml from current dir
dbt run --project-dir /path/to/project       # run from a different directory
```

---

## Project Structure

```
my_project/
├── dbt_project.yml          # project config, materializations, vars
├── profiles.yml             # warehouse connection (Snowflake, Postgres, etc.)
├── packages.yml             # external package dependencies
├── models/
│   ├── sources.yml          # declares raw source tables
│   ├── staging/             # 1:1 with source tables, light cleaning
│   ├── intermediate/        # joins, enrichment, business logic
│   └── marts/               # final aggregated tables for consumers
├── tests/                   # custom SQL tests
├── macros/                  # reusable Jinja functions
├── seeds/                   # static CSV files loaded as tables
└── target/                  # compiled SQL output (gitignore this)
```

---

## Materializations

| Type | What it creates | When to use |
|---|---|---|
| `view` | SQL view — no data stored | Staging, intermediate models |
| `table` | Physical table — full drop+recreate on every run | Marts, final outputs |
| `incremental` | Appends/merges only new rows | Large tables where full refresh is expensive |
| `ephemeral` | CTE inlined into downstream model — no object created | Lightweight intermediate logic |

Set in `dbt_project.yml`:
```yaml
models:
  my_project:
    staging:
      +materialized: view
    intermediate:
      +materialized: view
    marts:
      +materialized: table
```

Override in a specific model file (only when diverging from project default):
```sql
{{ config(materialized='incremental') }}
```

---

## sources.yml

Declares raw tables that dbt reads but does not create.

```yaml
version: 2

sources:
  - name: olist
    database: SNOWFLAKE_LEARNING_DB
    schema: PUBLIC
    tables:
      - name: OLIST_ORDERS_DATASET
      - name: OLIST_ORDER_ITEMS_DATASET
      - name: OLIST_PRODUCTS_DATASET
```

Reference in SQL:
```sql
select * from {{ source('olist', 'OLIST_ORDERS_DATASET') }}
```

---

## ref() vs source()

| | When to use |
|---|---|
| `{{ source('name', 'TABLE') }}` | Referencing raw source tables declared in `sources.yml` |
| `{{ ref('model_name') }}` | Referencing another dbt model |

`ref()` builds the DAG — dbt uses it to determine execution order.

---

## schema.yml — Generic Tests

```yaml
version: 2

models:
  - name: fact_product_performance
    description: "Aggregated product performance metrics by category."   # shown in dbt docs
    columns:
      - name: product_category_name
        description: "Product category name from Olist products dataset."
        tests:
          - unique
          - not_null
      - name: total_sales
        description: "Sum of price + freight_value for the category."
        tests:
          - not_null
      - name: order_status
        tests:
          - accepted_values:
              values: ['delivered', 'shipped', 'canceled']
      - name: customer_id
        tests:
          - relationships:
              to: ref('stg_customers')
              field: customer_id
```

Built-in generic tests: `unique`, `not_null`, `accepted_values`, `relationships`

---

## Custom SQL Tests

Place in `tests/` folder. Test passes if query returns **0 rows** (any row returned = failure).

```sql
-- tests/test_non_negative_sales.sql
select *
from {{ ref('fact_seller_performance') }}
where total_sales_value < 0
   or order_count < 0
```

---

## Idempotency

| Materialization | Rerun behaviour |
|---|---|
| `view` | `CREATE OR REPLACE VIEW` — always safe |
| `table` | `DROP + CREATE` — always safe, same result |
| `incremental` | Appends new rows — use `--full-refresh` to reset |

---

## profiles.yml (Snowflake)

```yaml
my_snowflake_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: ICKJVKI-CR66247
      user: YOUR_USER
      password: YOUR_PASSWORD
      role: ACCOUNTADMIN
      database: SNOWFLAKE_LEARNING_DB
      schema: PUBLIC
      warehouse: COMPUTE_WH
      threads: 1
```

Default location: `~/.dbt/profiles.yml`
Override: `dbt run --profiles-dir /path/to/dir`

---

## Airflow + dbt Pattern (BashOperator)

```python
DBT_PROJECT_DIR = "/opt/airflow/my_snowflake_project"
DBT_EXECUTABLE  = "/home/airflow/.local/bin/dbt"
DBT_FLAGS       = f"--project-dir {DBT_PROJECT_DIR} --profiles-dir {DBT_PROJECT_DIR}"

deps    = BashOperator(task_id="dbt_deps",     bash_command=f"{DBT_EXECUTABLE} deps {DBT_FLAGS}")
clean   = BashOperator(task_id="dbt_clean",    bash_command=f"{DBT_EXECUTABLE} clean {DBT_FLAGS}")
compile = BashOperator(task_id="dbt_compile",  bash_command=f"{DBT_EXECUTABLE} compile {DBT_FLAGS}")
run     = BashOperator(task_id="dbt_run",      bash_command=f"{DBT_EXECUTABLE} run {DBT_FLAGS}")
test    = BashOperator(task_id="dbt_test",     bash_command=f"{DBT_EXECUTABLE} test {DBT_FLAGS}")

deps >> clean >> compile >> run >> test
```

---

## Model Layers — What Goes Where

| Layer | Prefix | Rule |
|---|---|---|
| Staging | `stg_` | 1:1 with source, rename columns, cast types, no joins |
| Intermediate | `int_` | Joins, enrichment, delivery_days calculations |
| Marts | `fact_` / `dim_` | Final aggregations for business consumers |

---

## Documentation & Lineage

```bash
dbt docs generate            # builds docs from schema.yml descriptions + model SQL
dbt docs serve               # serves docs UI at http://localhost:8080
dbt docs serve --port 8085   # use a different port (useful if 8080 is taken by Airflow)
```

- Lineage graph shows the full DAG: sources → staging → intermediate → marts
- Descriptions from `schema.yml` appear as hover tooltips in the UI
- Run `dbt docs generate` after every `dbt run` to keep docs in sync

---

## Test Failure Behaviour

- dbt **does not roll back** tables on test failure
- Data stays in Snowflake, test results printed to stdout
- Exit code is non-zero — Airflow marks the task as failed
- Find failing rows in `target/compiled/.../schema.yml/<test_name>.sql`
