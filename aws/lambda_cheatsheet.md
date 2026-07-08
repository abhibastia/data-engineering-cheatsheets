# AWS Lambda Cheatsheet

---

## Core Concepts

- Lambda = **serverless compute** — run code in response to events, no servers to manage
- **Event-driven**, **stateless** (each invocation independent), pay per invocation + duration
- Common in data pipelines as the **glue/trigger** between services

---

## Common Triggers

| Source | Example |
| --- | --- |
| **S3** | New object upload → validate / start Glue job |
| **EventBridge** | Scheduled (cron) or event-pattern invocation |
| **DynamoDB Streams** | React to item changes |
| **API Gateway** | HTTP request → function |
| **MSK / Kinesis** | Consume streaming records in batches |

---

## Handler Basics (Python)

```python
import json

def lambda_handler(event, context):
    print("Received event:", json.dumps(event, indent=2))
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key    = record['s3']['object']['key']
        print(f"New object: s3://{bucket}/{key}")
    return {"status": "success", "processed": len(event['Records'])}
```

- `event` = trigger payload (shape varies by source); `context` = runtime metadata
- Logs go automatically to **CloudWatch** → `/aws/lambda/<function-name>`

---

## Trigger a Glue Job from Lambda

```python
import boto3
glue = boto3.client('glue', region_name='us-east-1')

def lambda_handler(event, context):
    resp = glue.start_job_run(JobName='ny_taxi_csv_to_parquet')
    return {'JobRunId': resp['JobRunId']}
```

Required IAM on the Lambda role:

```json
{ "Effect": "Allow",
  "Action": ["glue:StartJobRun", "glue:GetJobRun", "glue:GetJob"],
  "Resource": "*" }
```

---

## S3 Trigger Setup

- Function overview → **Add trigger** → S3
- Configure: bucket, event type (`All object create events`), optional **prefix** (`raw/`) / suffix (`.csv`)
- ⚠️ Don't let a function write back into the same prefix that triggers it → infinite loop

---

## Common Issues

| Symptom | Cause / Fix |
| --- | --- |
| `AccessDenied` on S3/Glue | Add `s3:GetObject`, `s3:ListBucket`, `glue:StartJobRun` to execution role |
| Function timeout | Raise timeout, or push long work to Glue (Lambda max 15 min) |
| Missing dependencies | Package as a **Lambda layer** or container image |
| Continuous retries | Configure a **Dead Letter Queue (DLQ)** to stop retry storms |

---

## Best Practices

- Keep functions small and single-purpose; offload heavy compute to Glue/EMR
- Least-privilege execution role, scoped to specific ARNs/prefixes
- Use environment variables for config (job names, buckets, region)
- Watch invocation count + `Errors` metric in CloudWatch; alarm on `Errors > 0`