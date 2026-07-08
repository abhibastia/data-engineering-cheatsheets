# EventBridge Cheatsheet

---

## Core Concepts

- EventBridge = **serverless event bus** connecting AWS services via JSON events
- An **event** describes a change (S3 upload, Glue job finished, scheduled time)
- Routes events to targets: **Lambda, Glue, Step Functions, SNS**, etc.
- Two rule types: **Schedule** (cron/rate) and **Event pattern** (react to service events)
- Related newer features: **EventBridge Scheduler** (dedicated, scalable scheduler — now preferred over scheduled rules for cron/rate jobs) and **EventBridge Pipes** (point-to-point source→target integrations with filtering/enrichment)

---

## Rule Types

| Type | Use |
| --- | --- |
| **Schedule → rate** | Run every N minutes/hours (`rate(5 minutes)`) |
| **Schedule → cron** | Specific times (`cron(0 10 * * ? *)` = daily 10:00 UTC) |
| **Event pattern** | Match a service event (e.g. Glue `SUCCEEDED`, S3 `PutObject`) |

---

## Schedule Expressions

```
rate(5 minutes)          # every 5 minutes
rate(1 hour)             # hourly
cron(0 10 * * ? *)       # every day at 10:00 AM UTC
cron(0/15 * * * ? *)     # every 15 minutes
```

- Cron format: `cron(min hour day-of-month month day-of-week year)` — note the `?` placeholder

---

## Targets

- **AWS service** → pick service + API (e.g. Glue → `StartJobRun` with `JobName`)
- **Lambda function** → invoke with optional constant JSON input
- Enable **Retry policy + Dead Letter Queue** under additional settings for failure handling

---

## Typical Automated Pipeline

```
EventBridge (schedule/event) → Lambda → Glue ETL Job
```

Replaces manual triggering with time-based or event-driven automation.

---

## IAM — Trust Relationship

When EventBridge (scheduler) assumes a role to call a target, add the principals:

```json
{ "Effect": "Allow",
  "Principal": { "Service": ["lambda.amazonaws.com", "scheduler.amazonaws.com"] },
  "Action": "sts:AssumeRole" }
```

---

## Verify & Debug

- Rule status should show **Enabled**
- Check **EventBridge → Rules → Metrics / invocation history**
- Confirm downstream target ran: Lambda CloudWatch logs, Glue job run history
- ⚠️ Rule not firing → disabled rule or bad schedule; target fails → target permissions
- Exhausted retries → inspect the **DLQ**