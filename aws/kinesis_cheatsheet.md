# Kinesis Cheatsheet

---

## Core Concepts

- Kinesis = AWS real-time streaming platform. Two services you'll mix up:
  - **Kinesis Data Streams (KDS)** — custom real-time processing platform (you write consumers)
  - **Amazon Data Firehose** — managed delivery to S3/Redshift/OpenSearch (no code)
- ⚠️ **Kinesis Data Firehose was renamed "Amazon Data Firehose" (Feb 2024)** — APIs, CLI (`firehose`), IAM actions, and CloudWatch metrics are unchanged; only the name/console differs. Now supports 40+ source/destination connectors (incl. Snowflake).

---

## KDS vs Firehose

| Feature | Data Streams (KDS) | Firehose (KDF) |
| --- | --- | --- |
| Purpose | Custom real-time processing | Managed delivery / loading |
| Latency | Real-time (~ms) | Near real-time (buffered, secs–mins) |
| Consumer | You write code (e.g. Lambda) | Fully managed, no code |
| Destinations | Anywhere your code sends | S3, Redshift, OpenSearch, Splunk, HTTP |
| Scaling | Shards (provisioned/on-demand) | Auto, fully serverless |
| Retention / replay | Up to 365 days, replayable | None — delivered then purged |

- 🔧 **KDS** for complex logic, multiple consumers, or replay; **Firehose** for simple load into S3/Redshift

---

## Key Terms (KDS)

| Term | Meaning |
| --- | --- |
| **Shard** | Unit of capacity/parallelism (1 MB/s in, 2 MB/s out) |
| **Partition key** | Determines which shard a record lands on |
| **Record** | Data blob + partition key |
| **Consumer** | App reading from shards (Lambda, KCL, custom) |

---

## Producer (boto3 → KDS)

```python
import boto3, json
kinesis = boto3.client("kinesis", region_name="us-east-1")

kinesis.put_record(
    StreamName="wikimedia-stream",
    Data=json.dumps({"wiki": "enwiki", "title": "Python", "type": "edit"}),
    PartitionKey="enwiki"          # groups related records onto the same shard
)
```

---

## Firehose Delivery Stream (to S3)

- **Source**: Kinesis Data Stream (or Direct PUT)
- **Destination**: S3 (bucket + prefix, e.g. `wikimedia/raw/`)
- **Buffering hints**: size (e.g. 1 MB) OR interval (e.g. 60 s) — whichever hits first flushes
- **Compression**: GZIP (optional); **Encryption**: SSE-S3
- Optional: transform with Lambda, or convert to Parquet/ORC (format conversion)

Output objects land partitioned by time: `2025/10/12/19/wikimedia-firehose-....gz`

---

## Firehose → Redshift

- Firehose stages to S3, then issues `COPY` into Redshift automatically
- Provide staging bucket + Redshift connection + COPY options (`json 'auto'` or CSV)

---

## Monitoring

| Service | Watch |
| --- | --- |
| KDS | `IncomingRecords`, `PutRecords.Success` |
| Firehose | `DeliveryToS3.Success`, `DeliveryToS3.Records`, buffering metrics |
| Logs | `/aws/kinesisfirehose/<stream>` (if enabled) |

- ⚠️ Firehose buffers — expect a delay (up to buffer interval) before data appears in S3