# GitHub Actions Cheatsheet

> CI/CD automation native to GitHub. Workflows are YAML files in `.github/workflows/`, triggered by repo **events**. Mental model: **If X happens in GitHub, then do Y.**

---

## 1. Core Concepts

| Term | What it is |
|---|---|
| **Workflow** | A YAML file in `.github/workflows/` defining an automation. Has one or more jobs. |
| **Event / Trigger** (`on:`) | *When* a workflow runs — `push`, `pull_request`, `schedule`, `workflow_dispatch`. |
| **Job** | A set of steps on one runner. Jobs run **in parallel by default**; sequence with `needs:`. |
| **Step** | A single task in a job — either `run:` (shell) or `uses:` (an Action). |
| **Runner** | The VM executing a job. GitHub-hosted (`ubuntu-latest`, `windows-latest`, `macos-latest`) or self-hosted. |
| **Action** | A reusable unit, e.g. `actions/checkout@v7`, `actions/setup-python@v7`. |

Execution model: **Jobs → Steps → Runners → Actions**.

---

## 2. Workflow Anatomy

```yaml
name: CI Pipeline                 # optional, shown in Actions tab

on:                               # triggers
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:              # manual "Run workflow" button

jobs:
  build:
    runs-on: ubuntu-latest        # runner OS
    steps:
      - name: Checkout code
        uses: actions/checkout@v7 # reuse an Action
      - name: Print message
        run: echo "Hello, GitHub Actions!"   # run a shell command
```

**Where to watch:** Repo → **Actions** tab → running workflows, logs, job details, annotations.

---

## 3. Triggers (`on:`)

```yaml
on:
  push:
    branches: [ main ]            # only pushes to main
  pull_request:
    branches: [ main ]            # PRs targeting main
  schedule:
    - cron: '0 2 * * *'           # nightly at 02:00 UTC
  workflow_dispatch:              # manual trigger from UI
  workflow_call:                  # callable by other workflows (reusable)
```

```yaml
on: [push, pull_request]          # shorthand: multiple events, default branches
```

### Path filters (only run when relevant files change)

Essential in a data monorepo — don't re-run the whole dbt/Airflow suite when only docs change:

```yaml
on:
  push:
    branches: [ main ]
    paths:
      - 'dags/**'
      - 'dbt/**'
      - 'requirements.txt'
    paths-ignore:
      - '**.md'
```

> **Version pinning:** `@v7` tracks the latest v7.x. For security-sensitive workflows, pin third-party actions to a full commit SHA (`uses: actions/checkout@<sha>  # v7.0.1`) — a tag can be moved to malicious code, a SHA cannot. Action majors move fast; the versions here are current as of mid-2026 — check the Marketplace before copying.

---

## 4. Jobs, Steps & Dependencies

### Parallel jobs (default)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building"
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing"       # runs at the same time as build
```

### Sequential jobs with `needs:`

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building"
  test:
    runs-on: ubuntu-latest
    needs: build                  # waits for build to succeed
    steps:
      - run: echo "Testing after build"
  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'   # only on main
    steps:
      - run: echo "Deploying"
```

> `needs:` on a matrix job waits for **all** matrix runs to finish.

### Concurrency (cancel superseded runs / serialize deploys)

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true          # new push cancels the in-flight PR run
```

For deploys, **don't** cancel — serialize instead so two applies never overlap:

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false         # queue; one deploy at a time
```

---

## 5. Environment Variables (`env:`)

Three scopes — workflow, job, step (narrowest wins):

```yaml
env:
  APP_ENV: development            # workflow-level (all jobs)

jobs:
  example:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: "job scope"        # job-level
    steps:
      - run: echo "Env=$APP_ENV, Job=$JOB_VAR"
        env:
          STEP_VAR: "step scope"  # step-level
```

---

## 6. Conditionals (`if:`)

```yaml
steps:
  - name: Deploy only on main
    if: github.ref == 'refs/heads/main'
    run: echo "Deploying to production"
```

Common contexts: `github.ref`, `github.event_name`, `github.actor`, `success()`, `failure()`, `always()`.

```yaml
plan:
  if: github.event_name == 'pull_request'   # run job only on PRs
apply:
  if: github.ref == 'refs/heads/main'        # run job only on main
```

---

## 7. Secrets

Encrypted env vars. GitHub **auto-masks** them in logs (shown as `***`). Set at three levels:

| Level | Scope | Use for |
|---|---|---|
| **Repository** | One repo | Project-specific keys |
| **Organization** | Many repos | Shared creds (e.g. DockerHub) — pick which repos get access |
| **Environment** | Per deploy stage (`staging`, `production`) | Stage-specific keys + approval gates |

**Add:** Settings → Secrets and variables → Actions → *New repository secret*.

### Using secrets

```yaml
# Inline on a step
steps:
  - run: echo "Hello $MY_SECRET"
    env:
      MY_SECRET: ${{ secrets.MY_SECRET }}

# Job-wide
jobs:
  example:
    runs-on: ubuntu-latest
    env:
      API_KEY: ${{ secrets.API_KEY }}
    steps:
      - run: echo "API key is set"

# As action inputs
steps:
  - uses: docker/login-action@v4
    with:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

**Do:** scope per environment, use org secrets for shared values, rotate often.
**Don't:** commit secrets, print them, or reuse one everywhere.

---

## 8. Protected Environments (approval gates)

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production            # add reviewers/branch rules in repo settings
      url: https://myapp.com
    steps:
      - run: echo "Deploying with $DEPLOY_TOKEN"
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

Requires manual approval before `production` runs → **Actions → Review deployments**.

---

## 9. Enterprise Security Patterns

### OIDC — cloud auth without stored secrets (short-lived tokens)

```yaml
jobs:
  aws-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write             # required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions
          aws-region: us-east-1
      - run: aws s3 ls s3://myorg-datalake/
```

### Fine-grained `GITHUB_TOKEN` permissions

```yaml
permissions:
  contents: read
  deployments: write
  packages: none
```

### Mask an arbitrary value in logs

```yaml
steps:
  - run: echo "::add-mask::my-super-secret"
```

### External secret managers

`hashicorp/vault-action@v4`, AWS Secrets Manager, Azure Key Vault, Google Secret Manager.

```yaml
steps:
  - uses: hashicorp/vault-action@v4
    with:
      url: ${{ secrets.VAULT_URL }}
      token: ${{ secrets.VAULT_TOKEN }}
      secrets: |
        secret/data/myapp API_KEY | API_KEY
```

---

## 10. Reusable Workflows

Define once, call from many places — like a function. Trigger must be `workflow_call`.

### Callable workflow (`.github/workflows/python-build.yml`)

```yaml
name: Python Build

on:
  workflow_call:                  # only runs when called by another workflow
    inputs:
      python-version:
        type: string
        default: "3.11"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: ${{ inputs.python-version }}
      - run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - run: pytest -q
```

### Caller workflow (`.github/workflows/ci.yml`)

```yaml
name: CI with E2E
on: [push, pull_request]

jobs:
  build:
    uses: ./.github/workflows/python-build.yml   # same repo
    with:
      python-version: "3.11"
  e2e:
    needs: build                  # only if build passes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - run: pytest tests/e2e -q
```

**Call sources:**

| Source | Syntax |
|---|---|
| Same repo | `./.github/workflows/python-build.yml` |
| Another repo | `owner/repo/.github/workflows/python-build.yml@main` |
| Pinned version | `owner/repo/.github/workflows/build.yml@v1` |

**Best practice:** keep them small & focused, **version-pin external ones** (`@v1`, not `@main`), validate inputs, pass only needed secrets.

---

## 11. Matrix Strategy

Run the *same job* across many inputs (versions, OSes) without duplicating YAML.

```yaml
jobs:
  build:
    strategy:
      matrix:
        python-version: ["3.10", "3.11"]   # one job per value
      fail-fast: false          # let all runs finish (good for debugging)
      max-parallel: 4           # cap concurrency
    uses: ./.github/workflows/python-build.yml
    with:
      python-version: ${{ matrix.python-version }}
```

| Key | Effect |
|---|---|
| `strategy.matrix` | Declares axes → one job per combination |
| `fail-fast` | `true` cancels remaining runs on first failure; `false` runs all |
| `max-parallel` | Max matrix jobs running at once |
| `include` | Add extra specific combinations |
| `exclude` | Remove specific combinations |

### NxM matrix (OS × Python)

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    python-version: ["3.10", "3.11"]
runs-on: ${{ matrix.os }}
name: Build (${{ matrix.os }} — py${{ matrix.python-version }})
```

### include / exclude

```yaml
strategy:
  matrix:
    python-version: [3.9, 3.10, 3.11]
    exclude:
      - python-version: 3.9
    include:
      - python-version: 3.12
        extra-label: "preview"
```

---

## 12. Caching & Artifacts

### Cache dependencies

```yaml
- uses: actions/cache@v6
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ matrix.python-version }}-${{ hashFiles('requirements.txt') }}
```

### Share build outputs between jobs

```yaml
# Producer job
- uses: actions/upload-artifact@v7
  with:
    name: dist-${{ inputs.python-version }}
    path: dist/*.whl

# Consumer job (needs the producer)
- uses: actions/download-artifact@v8
  with:
    name: dist-3.11
```

Each job runs on a **fresh runner** — re-`checkout` and re-install deps in every job, or pass state via artifacts.

---

## 13. Data Engineering Patterns

### Nightly dbt run against a warehouse

```yaml
name: dbt-nightly
on:
  schedule:
    - cron: '0 3 * * *'            # 03:00 UTC daily
  workflow_dispatch:

jobs:
  dbt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with: { python-version: "3.11" }
      - run: pip install dbt-snowflake
      - run: dbt deps
      - run: dbt build --fail-fast   # run + test models; stop on first failure
        env:
          DBT_PROFILES_DIR: ./
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
```

### Build & push a pipeline image to ECR (OIDC, no stored keys)

```yaml
jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-ecr
          aws-region: us-east-1
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - uses: docker/build-push-action@v7
        with:
          push: true
          tags: ${{ steps.ecr.outputs.registry }}/etl-job:${{ github.sha }}
```

### Trigger an external pipeline (e.g. Airflow/Databricks) after merge

```yaml
- name: Trigger Airflow DAG
  run: |
    curl -X POST "$AIRFLOW_URL/api/v1/dags/etl_main/dagRuns" \
      -H "Authorization: Bearer $AIRFLOW_TOKEN" \
      -H "Content-Type: application/json" -d '{"conf": {}}'
  env:
    AIRFLOW_URL: ${{ secrets.AIRFLOW_URL }}
    AIRFLOW_TOKEN: ${{ secrets.AIRFLOW_TOKEN }}
```

> **Data CI checklist:** lint SQL (`sqlfluff`), validate DAGs import (`python -c "import dag"`), run `dbt build` on a PR against a scratch schema, and gate `main` merges on green. Keep warehouse creds in **environment** secrets with approval, not repo-wide.

---

## 14. Quick Reference

| Task | Snippet |
|---|---|
| Manual trigger | `on: workflow_dispatch` |
| Nightly run | `on: { schedule: [{ cron: '0 2 * * *' }] }` |
| Run job only on main | `if: github.ref == 'refs/heads/main'` |
| Run job only on PR | `if: github.event_name == 'pull_request'` |
| Sequence jobs | `needs: build` |
| Checkout repo | `uses: actions/checkout@v7` |
| Set up Python | `uses: actions/setup-python@v7` |
| Use a secret | `${{ secrets.NAME }}` |
| Read a matrix value | `${{ matrix.python-version }}` |
| Call reusable workflow | `uses: ./.github/workflows/build.yml` |
| Mask a value | `echo "::add-mask::value"` |
