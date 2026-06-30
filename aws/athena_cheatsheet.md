# Athena Cheatsheet

---

## Core Concepts

- Athena = serverless SQL engine over S3; no cluster, no data loading
- Schema-on-read — data files unchanged; schema applied only at query time
- Billed per TB scanned ($5/TB) — format and partitioning directly control cost
- Uses Glue Data Catalog as metastore; tables visible to Glue ETL, EMR, Redshift Spectrum

---

## Format Comparison

| Format | Columnar | Performance | Use |
| --- | --- | --- | --- |
| CSV | No | Slowest | Raw ingest only |
| JSON | No | Slow | Semi-structured logs |
| Parquet | Yes | Fast | ✅ Standard for processed data |
| ORC | Yes | Fast | Alternative to Parquet |

- 🔧 Always convert raw CSV → Parquet before production queries; 10–100x less data scanned

---

## Partitioning

- S3 folder structure: `year=2024/month=06/day=07/`
- Athena skips partitions not matching `WHERE` clause
- `MSCK REPAIR TABLE` registers new partitions from S3 paths
- ⚠️ Filter on **partition columns** to prune; filtering on regular columns scans everything first
- ⚠️ No `MSCK REPAIR TABLE` = no results on new partitions

---

## Cost Rules

- `LIMIT 100` does **not** reduce data scanned — use `WHERE` with partition filters
- `SELECT *` on large unpartitioned CSV = expensive even for 10 rows
- Parquet + partitions + compression = maximum cost reduction
- Columnar projection (`SELECT col1, col2`) only works on Parquet/ORC

---

## Key SQL

```sql
-- Create external table (CSV)
CREATE EXTERNAL TABLE db.table (col STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://bucket/prefix/'
TBLPROPERTIES ('skip.header.line.count'='1');

-- Create partitioned Parquet table
CREATE EXTERNAL TABLE db.table (col STRING)
PARTITIONED BY (year STRING, month STRING)
STORED AS PARQUET
LOCATION 's3://bucket/prefix/';

-- Register partitions
MSCK REPAIR TABLE db.table;

-- CTAS: CSV → partitioned Parquet
CREATE TABLE db.new_table
WITH (format='PARQUET', partitioned_by=ARRAY['year','month'],
      external_location='s3://bucket/output/')
AS SELECT col1, col2, substr(date,1,4) AS year, substr(date,6,2) AS month
FROM db.old_table;
```

---

## Glue Integration

- Glue Crawler: scans S3 → infers schema → creates table in Glue catalog
- Tables created in Athena are visible in Glue and vice versa
- 🔧 Use crawler for initial discovery; use `MSCK REPAIR TABLE` for ongoing partition management
