# CloudTrail Cheatsheet

---

## Core Concepts

- CloudTrail records every AWS API call — Console clicks, CLI commands, SDK calls, service-to-service
- Recording starts the moment an account is created — no setup required
- Event History = always on, 90 days, management events only, free
- Trail = long-term delivery to S3; must be created; first trail per region is free
- ⚠️ CloudTrail is **not** real-time — events have up to 15 minutes of delay before appearing in `lookup-events`

---

## Event Types

| Type | Default | Cost | Examples |
| --- | --- | --- | --- |
| Management | On (90 days) | Free (1st trail) | `RunInstances`, `CreateBucket`, `PutBucketPolicy` |
| Data | Off | ~$0.10/100K | S3 `GetObject`/`PutObject`, Lambda `Invoke` |
| Insights | Off | ~$0.35/100K analysed | Anomalous API spikes |

- ⚠️ Data events off by default — high volume pipeline generates millions/day, each billed

---

## Trail vs Event History

| | Event History | Trail |
| --- | --- | --- |
| Retention | 90 days | Unlimited (S3) |
| Setup | None | Required |
| Event types | Management only | All types |
| Query | Console + CLI | S3 JSON.gz + Athena |

---

## Bucket Policy Requirement

- Via **Console**: AWS adds the bucket policy automatically
- Via **CLI**: you must add it manually before `create-trail` — otherwise trail creates but logs are never delivered
- ⚠️ `InsufficientS3BucketPolicyException` = forgot to apply bucket policy before CLI trail creation

---

## CLI

```bash
aws cloudtrail create-trail --name my-trail --s3-bucket-name $BUCKET  # create trail; logging not started yet
aws cloudtrail start-logging --name my-trail                           # begin delivering events to S3
aws cloudtrail stop-logging --name my-trail                            # pause delivery; trail still exists
aws cloudtrail delete-trail --name my-trail                            # remove trail entirely
aws cloudtrail get-trail-status --name my-trail \
  --query "{Logging:IsLogging,LastDelivery:LatestDeliveryTime}"        # check if logging is active and last delivery time

# Lookup events (90-day window, management events only)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=workshopuser \
  --max-results 10 --output table                                      # all actions by a specific user

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 5 --output table                                       # who launched EC2 instances and when
```

---

## Log Path in S3

```
s3://bucket/AWSLogs/[account-id]/CloudTrail/[region]/[year]/[month]/[day]/filename.json.gz
```
