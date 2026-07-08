# Amazon OpenSearch Cheatsheet

---

## Core Concepts

- OpenSearch Service = managed **search + analytics engine** (successor to Elasticsearch Service)
- Store, index, and analyze large volumes of semi-structured JSON data in real time
- Powers **log analytics / observability** and **full-text search** applications
- Includes **OpenSearch Dashboards** (formerly Kibana) for visualization

---

## Key Concepts

| Term | Meaning |
| --- | --- |
| **Cluster** | Group of nodes working together |
| **Node** | Single server (data / master / UltraWarm) |
| **Index** | Namespace of documents (~ a table) |
| **Document** | JSON object with searchable fields (~ a row) |
| **Shard** | Partition of an index for parallelism |
| **Replica** | Copy of a shard for HA |

---

## Typical Ingestion Pipeline

```
Data sources (logs, metrics, CSV)
   → Ingestion (Kinesis / Firehose / Logstash / S3)
   → OpenSearch cluster (index + query)
   → OpenSearch Dashboards (visualize)
```

Log analytics example: CloudWatch Logs → Firehose → OpenSearch → Dashboards

---

## Querying (REST / Dashboards Dev Tools)

```json
GET logs-index/_search
{ "query": { "match": { "message": "error" } } }
```

```json
GET logs-index/_search
{ "aggs": { "by_status": { "terms": { "field": "status_code" } } } }
```

---

## Integrations

| Source | Path |
| --- | --- |
| S3 | S3 → Firehose → OpenSearch |
| Glue ETL | Glue job → OpenSearch |
| EMR/Spark | Spark → OpenSearch connector |
| CloudWatch | Logs → subscription filter → OpenSearch |

---

## Monitoring, Security, Cost

- Monitor: `CPUUtilization`, `FreeStorageSpace`, `ClusterStatus.yellow/red`
- Prefer **VPC access** over public; enable encryption at rest + in transit; IAM/fine-grained access control
- Use **UltraWarm** for cheap long-term log retention
- Legacy Free Tier (accounts before 2025-07-15): 750 hrs/month `t3.small.search` + 10 GB EBS; newer accounts use the credit-based Free Tier — delete the domain after labs either way
- Note: **OpenSearch Serverless** exists for auto-scaling collections without managing nodes