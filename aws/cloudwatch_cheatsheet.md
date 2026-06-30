# CloudWatch Cheatsheet

---

## Core Concepts

- CloudWatch = metrics + logs + alarms + dashboards for AWS and custom apps
- Metrics are time-series data points; grouped by namespace (`AWS/EC2`, `AWS/S3`)
- Logs are text event streams; stored in log groups → log streams
- Alarms monitor one metric and trigger actions (SNS, auto-scaling, EC2 action)

---

## Key Metrics to Know

| Service | Metric | Gotcha |
| --- | --- | --- |
| EC2 | `CPUUtilization` | % — set alarm at 80%+ sustained |
| S3 | `BucketSizeBytes` | Updates once per day — not real-time |
| S3 | `NumberOfObjects` | Updates once per day |
| Lambda | `Errors` | Alarm on > 0 for silent failure detection |
| Billing | `EstimatedCharges` | Only available in **us-east-1** |

---

## Alarm States

| State | Meaning |
| --- | --- |
| OK | Metric within threshold |
| ALARM | Threshold breached |
| INSUFFICIENT_DATA | Not enough data points |

---

## Logs

- EC2 does **not** auto-ship logs — install CloudWatch agent
- Lambda auto-ships to `/aws/lambda/[function-name]`
- Default retention = **never expire** — set a retention policy or storage accumulates
- Log Insights = SQL-like query language for searching log content

---

## CLI

```bash
aws cloudwatch list-metrics --namespace AWS/EC2                        # browse available EC2 metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=$ID \
  --start-time $(date -u -v-1H '+%Y-%m-%dT%H:%M:%SZ') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
  --period 300 --statistics Average --output table                     # CPU avg last 1h in 5-min buckets

aws cloudwatch describe-alarms \
  --query "MetricAlarms[*].{Name:AlarmName,State:StateValue}" --output table  # alarm names and current states

aws logs put-retention-policy --log-group-name /my/app --retention-in-days 30  # prevent unbounded log storage cost
```

---

## Pricing Gotchas

- Basic monitoring (5-min metrics) = free; detailed (1-min) = ~$0.01/metric/month
- Logs ingestion = ~$0.50/GB; storage = ~$0.03/GB/month
- ⚠️ No retention policy = logs stored forever = unbounded cost
- Dashboards: first 3 free, then ~$3/dashboard/month
- `EstimatedCharges` billing metric only in `us-east-1`
