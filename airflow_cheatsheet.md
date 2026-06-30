# Airflow Cheatsheet

## Core Concepts

| Concept | What it is |
|---|---|
| **DAG** | Directed Acyclic Graph — a pipeline definition. Nodes = tasks, edges = dependencies |
| **Task** | A single unit of work inside a DAG (run dbt, upload file, send email) |
| **Operator** | The type of task — BashOperator, PythonOperator, etc. |
| **TaskInstance** | One specific run of a task at a specific time |
| **DAG Run** | One execution of the full DAG |
| **Scheduler** | Watches the clock, triggers DAG runs on schedule |
| **Executor** | Decides how tasks are run (locally, on workers, on Kubernetes) |
| **Worker** | The process that actually executes a task |
| **XCom** | Cross-communication — pass small values between tasks |

---

## DAG Anatomy

```python
import pendulum
from airflow.sdk import dag
from airflow.providers.standard.operators.bash import BashOperator

@dag(
    dag_id="my_pipeline",
    start_date=pendulum.datetime(2025, 1, 1, tz="UTC"),
    schedule="0 6 * * *",        # cron: every day at 6am
    catchup=False,                # don't backfill past runs
    tags=["dbt", "snowflake"],
    description="My pipeline",
)
def my_pipeline():

    task_a = BashOperator(task_id="task_a", bash_command="echo A")
    task_b = BashOperator(task_id="task_b", bash_command="echo B")
    task_c = BashOperator(task_id="task_c", bash_command="echo C")

    task_a >> task_b >> task_c    # A → B → C


my_pipeline()
```

---

## TaskFlow API (Modern Style)

The `@task` decorator is the modern Airflow 3.x way to write Python tasks. Return values are automatically passed as XCom — no push/pull needed.

```python
import pendulum
from airflow.decorators import dag, task

@dag(
    dag_id="etl_pipeline",
    start_date=pendulum.datetime(2025, 1, 1, tz="UTC"),
    schedule=None,
    catchup=False,
)
def etl_pipeline():

    @task
    def extract() -> dict:
        return {"rows": 1000, "source": "api"}

    @task
    def transform(raw: dict) -> dict:
        return {"rows": raw["rows"], "cleaned": True}

    @task
    def load(data: dict):
        print(f"Loading {data['rows']} rows")

    # Dependency is inferred from function chaining — no >> needed
    load(transform(extract()))

etl_pipeline()
```

**TaskFlow vs Operators — when to use which:**

| | TaskFlow `@task` | BashOperator / PythonOperator |
|---|---|---|
| Python logic | Preferred | Works but more verbose |
| Shell commands (dbt, etc.) | Not suitable | Use BashOperator |
| Data passing between tasks | Automatic via return value | Manual XCom push/pull |
| Code style | Cleaner, function-based | Explicit, class-based |

---

## Task Dependencies

```python
# Sequential
task_a >> task_b >> task_c

# Fan out (parallel)
task_a >> [task_b, task_c]

# Fan in
[task_b, task_c] >> task_d

# Explicit set_upstream / set_downstream
task_b.set_upstream(task_a)
task_b.set_downstream(task_c)
```

---

## Common Operators

| Operator | Use case | Import |
|---|---|---|
| `BashOperator` | Run any shell/CLI command | `airflow.providers.standard.operators.bash` |
| `PythonOperator` | Run a Python function | `airflow.providers.standard.operators.python` |
| `EmptyOperator` | Placeholder / grouping anchor | `airflow.providers.standard.operators.empty` |
| `SnowflakeOperator` | Run SQL on Snowflake | `airflow.providers.snowflake.operators.snowflake` |
| `S3ToSnowflakeOperator` | Load S3 file into Snowflake | `airflow.providers.snowflake.transfers.s3_to_snowflake` |
| `TriggerDagRunOperator` | Trigger another DAG | `airflow.operators.trigger_dagrun` |

### BashOperator
```python
from airflow.providers.standard.operators.bash import BashOperator

task = BashOperator(
    task_id="run_script",
    bash_command="python /opt/airflow/scripts/etl.py",
)
```

### PythonOperator
```python
from airflow.providers.standard.operators.python import PythonOperator

def my_function(**context):
    print("Hello from Python")

task = PythonOperator(
    task_id="run_python",
    python_callable=my_function,
)
```

---

## Schedule Syntax (Cron)

```
┌─ minute (0-59)
│  ┌─ hour (0-23)
│  │  ┌─ day of month (1-31)
│  │  │  ┌─ month (1-12)
│  │  │  │  ┌─ day of week (0=Sun, 6=Sat)
│  │  │  │  │
*  *  *  *  *
```

| Schedule | Cron |
|---|---|
| Every day at midnight | `"0 0 * * *"` |
| Every day at 6am | `"0 6 * * *"` |
| Every hour | `"0 * * * *"` |
| Every Monday at 9am | `"0 9 * * 1"` |
| Every 15 minutes | `"*/15 * * * *"` |
| Manual trigger only | `None` |

Presets: `"@daily"`, `"@hourly"`, `"@weekly"`, `"@monthly"`

---

## catchup

```python
catchup=False   # only run from now forward — almost always what you want
catchup=True    # backfill all missed runs since start_date — use carefully
```

If `start_date` is 30 days ago and `catchup=True`, Airflow queues 30 DAG runs immediately on first deploy.

---

## XCom — Passing Values Between Tasks

```python
# Push a value
def push(**context):
    context['ti'].xcom_push(key='row_count', value=1234)

# Pull in downstream task
def pull(**context):
    count = context['ti'].xcom_pull(task_ids='push_task', key='row_count')
    print(count)
```

XCom is for small values (strings, numbers, short dicts). Never push DataFrames or large files through XCom.

---

## Task States

| State | Meaning |
|---|---|
| `queued` | Waiting for a worker to pick it up |
| `running` | Actively executing |
| `success` | Completed without error |
| `failed` | Threw an exception or non-zero exit code |
| `skipped` | Upstream branching skipped this task |
| `upstream_failed` | An upstream task failed, this was never attempted |
| `retry` | Failed, waiting to retry |

---

## Retries

```python
task = BashOperator(
    task_id="flaky_task",
    bash_command="curl https://api.example.com",
    retries=3,                              # retry up to 3 times
    retry_delay=timedelta(minutes=5),       # wait 5 min between retries
)
```

---

## Docker Compose Setup (Airflow 3.x)

Key services:
| Service | Role |
|---|---|
| `airflow-apiserver` | REST API + UI — accessible at `localhost:8080` |
| `airflow-scheduler` | Monitors DAGs, triggers runs |
| `airflow-dag-processor` | Parses DAG files |
| `airflow-worker` | Executes tasks (CeleryExecutor) |
| `postgres` | Airflow's metadata database |
| `redis` | Celery task broker |

### Common Docker Commands
```bash
docker compose up -d              # start all services in background
docker compose down               # stop all services
docker compose build              # rebuild image (after .env or Dockerfile change)
docker compose logs -f            # stream all logs
docker compose logs airflow-worker -f   # stream worker logs only
docker compose ps                 # check service health
```

---

## Volumes — How Files Get Into Airflow

```yaml
volumes:
  - ./dags:/opt/airflow/dags                          # DAG files
  - ./logs:/opt/airflow/logs                          # task logs
  - ./my_snowflake_project:/opt/airflow/my_snowflake_project  # dbt project
```

Any file in `./dags` on your laptop is immediately visible inside the container at `/opt/airflow/dags`. No rebuild needed.

---

## Environment Variables / .env

```bash
# .env
FERNET_KEY=your_fernet_key
AIRFLOW_UID=50000
_PIP_ADDITIONAL_REQUIREMENTS=dbt-snowflake==1.10.0
```

`_PIP_ADDITIONAL_REQUIREMENTS` installs packages at container startup — slow but convenient for quick checks. For production, bake packages into a custom Docker image instead.

---

## Airflow + dbt Pattern

```
deps → clean → compile → run → test
```

- `deps` — install dbt packages (fresh container has none)
- `clean` — remove stale compiled artifacts
- `compile` — validate all refs/sources before hitting Snowflake
- `run` — build models
- `test` — validate data quality

Why not just `dbt build`? Separate tasks give you granular failure visibility in the Airflow UI — you can see exactly which step failed without parsing logs.

---

## Executors

| Executor | How it runs tasks | When to use |
|---|---|---|
| `SequentialExecutor` | One task at a time, same process | Local dev only |
| `LocalExecutor` | Parallel tasks, same machine | Small setups |
| `CeleryExecutor` | Distributed workers via Redis/RabbitMQ | Production, this bootcamp setup |
| `KubernetesExecutor` | Each task in its own pod | Large-scale production |

---

## Useful UI Actions

| Action | Where |
|---|---|
| Trigger a DAG manually | DAGs list → play button |
| View task logs | DAG Run → click task → Logs tab |
| Retry a failed task | DAG Run → click task → Mark as → retry |
| Clear a task (re-run) | DAG Run → click task → Clear |
| Pause a DAG | DAGs list → toggle off |

---

## Gotchas

- **`start_date` must be in the past** — Airflow won't schedule a DAG with a future `start_date`
- **DAG file must call the decorated function** — `my_pipeline()` at the bottom is required
- **`catchup=False` is almost always what you want** — set it explicitly, don't rely on the default
- **XCom has a size limit** — store large outputs in S3/GCS/Snowflake, pass only the reference
- **Container restart does not re-parse DAGs** — the dag-processor watches the folder; changes appear within 30s without restart
- **`.env` changes require `docker compose down && docker compose up -d`** to take effect
