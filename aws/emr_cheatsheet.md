# Amazon EMR Cheatsheet

---

## Core Concepts

- EMR (Elastic MapReduce) = **managed big-data platform** for Spark, Hadoop, Hive, Presto, HBase
- AWS handles provisioning, scaling, tuning of clusters — you focus on processing
- In short: **managed Spark/Hadoop clusters on AWS**

```
Raw Data (S3) → EMR (Spark/Hadoop/Hive) → Processed Data (S3, Redshift, DynamoDB, OpenSearch)
```

---

## Cluster Components

| Component | Role |
| --- | --- |
| **Master node** | Coordinates cluster, schedules jobs (YARN ResourceManager) |
| **Core nodes** | Run tasks + store data in HDFS |
| **Task nodes** | Compute only, no storage (great for Spot) |
| **HDFS** | Temporary distributed storage during processing |
| **EMRFS** | Lets EMR read/write directly from **S3** (persistent) |
| **YARN** | Resource allocation across nodes |

---

## Frameworks

| Framework | Use |
| --- | --- |
| **Spark** | In-memory distributed ETL/analytics (most common today) |
| **Hadoop** | Traditional MapReduce batch |
| **Hive** | SQL over HDFS/S3 |
| **Presto/Trino** | Interactive low-latency SQL across sources |
| **HBase** | NoSQL columnar real-time store |

- Modern default: **Spark + S3** on EMR

---

## EMR vs Glue

| | EMR | Glue |
| --- | --- | --- |
| Model | Clusters (or EMR Serverless) | Fully serverless jobs |
| Control | Full Spark/Hadoop config | Managed, less tuning |
| Best for | Large/complex, custom frameworks | Standard ETL, quick pipelines |

---

## Cost Optimization

- Use **Spot Instances** for task nodes
- Enable **auto-termination** when idle
- **EMR Serverless** = pay-per-job, most Free-Tier-friendly for demos/small workloads
- Store data in **S3** (persistent) not HDFS (ephemeral)

---

## Key Points

- Scales dynamically; supports batch and real-time
- Integrates with S3, Glue Data Catalog, CloudWatch, IAM
- Lower TCO than on-prem Hadoop
- Well-Architected: use auto-scaling clusters + Spot for large ETL