# AWS Well-Architected Framework Cheatsheet

---

## Core Concepts

- Set of best practices/design principles for secure, reliable, efficient, cost-effective workloads
- Used to **review architectures**, find gaps, and make informed trade-offs
- Evaluate your design with the **Well-Architected Tool** (Console) — choose Data Analytics/ML workload type

---

## The Six Pillars

| # | Pillar | Focus |
| --- | --- | --- |
| 1 | **Operational Excellence** | Run, monitor, automate, improve |
| 2 | **Security** | Protect data/systems, least privilege, encryption |
| 3 | **Reliability** | Recover from failure, meet continuity goals |
| 4 | **Performance Efficiency** | Use resources efficiently, right-size, scale |
| 5 | **Cost Optimization** | Avoid unnecessary spend, align to value |
| 6 | **Sustainability** | Minimize environmental impact / waste |

---

## Data Engineering Best Practices per Pillar

| Pillar | Do this |
| --- | --- |
| Operational Excellence | Automate ETL (Glue/Step Functions), schedule with EventBridge, monitor with CloudWatch |
| Security | IAM least privilege, KMS/S3 SSE encryption, mask PII (Glue/Lake Formation), audit via CloudTrail |
| Reliability | S3 versioning, AWS Backup for RDS/Redshift, **idempotent** pipelines, failure alarms |
| Performance | Partition + compress (Parquet), auto-scaling EMR, tune Spark, Redshift Spectrum |
| Cost | S3 lifecycle → Glacier, cost tags + Budgets, compress to cut Athena scan cost, consolidate jobs |
| Sustainability | Prefer serverless (Athena/Glue/Lambda) over always-on EC2, S3 Intelligent-Tiering |

---

## Example — Data Lake Pipeline Review

| Step | Service | Pillar | Best practice |
| --- | --- | --- | --- |
| Ingestion | Kinesis Firehose | Reliability | Multi-AZ delivery, retry policies |
| Storage | S3 | Security, Cost | Encryption, lifecycle rules |
| Processing | Glue ETL | Operational Excellence | Job monitoring, retries |
| Querying | Athena | Performance | Partitioned Parquet |
| Visualization | QuickSight | Cost, Sustainability | SPICE caching |

---

## Benefits

- Trustworthy, cost-effective pipelines; better compliance/governance
- Continuous improvement via measurable benchmarks; less operational risk/tech debt