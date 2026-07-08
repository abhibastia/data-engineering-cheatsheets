# Redshift Cheatsheet

---

## Core Concepts

- Redshift = fully managed, **columnar** cloud data warehouse for **analytics (OLAP)**
- PostgreSQL-compatible SQL; scales to petabytes
- **MPP** (Massively Parallel Processing): queries split across compute nodes
- Not a replacement for RDS — OLAP (aggregations, BI) not OLTP (transactions)

---

## Architecture

| Component | Role |
| --- | --- |
| **Leader node** | Parses queries, builds plan, distributes work, aggregates results |
| **Compute nodes** | Store data + run query fragments in parallel |
| **Columnar storage** | Stores by column → faster scans, better compression |
| **Slices** | Each compute node is split into slices for parallelism |

---

## Provisioned vs Serverless

| | Provisioned Cluster | Serverless |
| --- | --- | --- |
| Capacity | You size nodes | Measured in **RPUs**, auto-managed |
| WLM | Manual **WLM queues** (memory/concurrency) | Automatic WLM |
| Cost control | Node count | Base/Max RPUs + Usage Limits + QMRs |

---

## Serverless Workload Management

- No manual queues — control cost/perf with three levers:
  - **Base / Max capacity (RPUs)** — compute floor/ceiling. Range **4–512 RPU** (4-RPU minimum added in 2025, was 8); **default 128 RPU** → drop to 8 (or 4) for labs
  - **Query Monitoring Rules (QMRs)** — abort/limit queries (e.g. queue time > 60s)
  - **Usage Limits** — cap RPU-hours daily/weekly/monthly → alert or turn off queries
- ⚠️ Forgetting to right-size RPUs burns credits fast — the 128-RPU default is expensive; always lower it + set a Usage Limit in labs
- Note: 4-RPU base supports up to 32 TB managed storage; above 32 TB the minimum is 8 RPU

---

## Loading Data — COPY from S3

```sql
COPY schema.table
FROM 's3://bucket/path/'
IAM_ROLE 'arn:aws:iam::<acct>:role/RedshiftRole'
CSV IGNOREHEADER 1
REGION 'us-east-1';

-- Parquet
COPY schema.table
FROM 's3://bucket/curated/'
IAM_ROLE 'arn:aws:iam::<acct>:role/RedshiftRole'
FORMAT AS PARQUET
REGION 'us-east-1';
```

- 🔧 `COPY` is the fast, parallel bulk loader — never use row-by-row `INSERT` for bulk

---

## Redshift Spectrum — Query S3 Without Loading

```sql
-- 1. Link to Glue Data Catalog via external schema
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'glue_db'
IAM_ROLE 'arn:aws:iam::<acct>:role/RedshiftSpectrumRole'
REGION 'us-east-1';

-- 2. Query S3 data directly (or define an external table)
SELECT * FROM spectrum_schema.demo_ext LIMIT 10;

-- 3. Join warehouse tables with S3 data in one query
SELECT d.id, e.value
FROM demo d
JOIN spectrum_schema.demo_ext e ON d.id = e.id;
```

- Spectrum role needs **S3 read + Glue Catalog** access (`glue:GetTable`, `glue:GetPartitions`)
- ⚠️ `TABLE_NOT_FOUND` → Glue DB/table not in same region, or crawler didn't run

---

## Performance Tips

- **Distribution key** — spreads rows across slices; pick a common join key to co-locate data
- **Sort key** — orders data on disk; speeds range/filter scans
- Analyze slow queries with `STL_QUERY`, `SVL_QUERYREPORT`
- Prefer Parquet + partitioned S3 for Spectrum scans

---

## Integrations

- Sources/targets: S3 (Spectrum + COPY), Glue Data Catalog, Kinesis Firehose, DMS
- BI: QuickSight, Tableau, Power BI, Looker