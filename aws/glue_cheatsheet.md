# AWS Glue Cheatsheet

---

## Core Concepts

- Glue = **serverless** data integration — discover, catalog, and transform data (no clusters to manage)
- ETL jobs run on a managed **Apache Spark** (or Python Shell) environment
- Sits between storage (S3) and analytics (Athena, Redshift Spectrum, QuickSight)

---

## Components

| Component | What it does |
| --- | --- |
| **Crawler** | Scans S3, infers schema, creates/updates catalog tables |
| **Data Catalog** | Central metadata store (databases, tables, columns) — shared by Athena, Redshift, EMR |
| **ETL Job** | Extract → Transform → Load, in PySpark (visual or code) |
| **Python Shell job** | Lightweight non-Spark job for small tasks / API calls |
| **Trigger** | Starts jobs on schedule, event, or job-completion |

- Tables created by a crawler or in Athena are visible everywhere the Catalog is used

---

## Crawler → Catalog → Query Flow

```
S3 (raw) → Glue Crawler → Data Catalog → Athena / Redshift Spectrum
                                       → Glue ETL Job → S3 (curated, Parquet)
```

1. Create database → 2. Add crawler pointing at S3 path → 3. Run → 4. Table appears → 5. Query in Athena

---

## ETL Job — CSV → Parquet (PySpark)

```python
import sys
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext
from pyspark.sql.functions import to_timestamp, year, month, col

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext(); glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext); job.init(args['JOB_NAME'], args)

df = spark.read.option("header", "true").csv("s3://bucket/raw/")
df = df.withColumn("pickup_ts", to_timestamp(col("tpep_pickup_datetime"))) \
       .withColumn("year", year("pickup_ts")).withColumn("month", month("pickup_ts"))

(df.write.mode("overwrite")
   .option("compression", "snappy")     # Glue/Spark default for Parquet
   .partitionBy("year", "month")
   .parquet("s3://bucket/curated/"))

job.commit()
```

- Output layout: `curated/year=2016/month=1/part-0000.snappy.parquet`

---

## Job Tuning

| Parameter | Recommendation |
| --- | --- |
| **Worker type (G)** | `G.1X`/`G.2X` standard ETL; `G.4X`/`G.8X` heavy jobs; `G.025X` streaming/Python-shell |
| **Worker type (R)** | `R.1X`–`R.8X` memory-optimized (Glue 4.0+); `G.12X`/`G.16X` for very large jobs |
| **# workers** | Start 2–5, scale gradually |
| **Job bookmarking** | Enable for **incremental** loads (skips already-processed data) |
| **Partitioning** | Partition on query filters (e.g. `year`, `month`) |
| **Compression** | Parquet + Snappy |
| **Glue version** | **5.0** (latest) = Spark 3.5.4, Python 3.11, Java 17; 4.0 = Spark 3.3; 3.0 = Spark 3.1 |

- 🔧 Match Spark features to the Glue version — Glue 5.0 runs Spark 3.5 (same as the PySpark cheatsheet)
- OOM / `ExecutorLostFailure` → bump worker type (`G.1X`→`G.2X`), use an R worker for memory-heavy jobs, reduce wide shuffles, add `--conf spark.executor.memoryOverhead=1024`
- For debugging incremental loads: `--job-bookmark-option disable`

---

## Common Errors

| Error | Fix |
| --- | --- |
| `AccessDeniedException` | Add `s3:GetObject`, `glue:GetDatabase/GetTable` to Glue role |
| Schema mismatch | Cast fields in job or fix Catalog schema |
| OOM / executor lost | Larger/more workers, simpler transforms |
| Job timeout | Extend timeout, reduce data volume |
| `Py4JJavaError` | Read CloudWatch driver/executor logs for the stack trace |

---

## Glue → DynamoDB (Python Shell)

```python
import boto3
table = boto3.resource("dynamodb", region_name="us-east-1").Table("Wikimedia_Changes")
table.put_item(Item={"wiki": "enwiki", "event_ts_id": "1716078945_123", "title": "..."})
```

- Python Shell job type = good for streaming-style ingest, small transforms, boto3 calls

---

## Monitoring & IAM

- Job runs → Glue Console → Runs tab → **View logs** (CloudWatch driver + executor logs)
- Enable **job metrics / observability** for driver/executor memory
- Roles: separate least-privilege roles for Glue, Lambda, Redshift; restrict to specific S3 prefixes
- Validate permissions with the **IAM Policy Simulator**