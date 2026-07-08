# Spark (PySpark) Cheatsheet

Apache Spark is a distributed, in-memory data processing engine. Unlike Pandas (single machine), Spark **parallelizes** computation across a cluster and only executes when an **action** is called (lazy evaluation).

> **Version baseline:** examples target **Spark 3.5** (LTS, security fixes to Nov 2027) — the runtime used by AWS Glue 5.0. Spark **4.x** is the current release line; the DataFrame/SQL APIs shown here carry forward unchanged. Version gates are noted inline (e.g. `3.4+`).

## Core Concepts

| Concept | What it is |
|---|---|
| **RDD** | Resilient Distributed Dataset — low-level, immutable, distributed collection. Fault-tolerant, no schema. |
| **DataFrame** | Table-like abstraction with named columns + schema. Optimized by the Catalyst engine. Prefer this. |
| **Dataset** | Typed DataFrame (Scala/Java only — PySpark has no Dataset). |
| **Spark SQL** | Query DataFrames with SQL syntax via temp views. |
| **SparkSession** | Entry point for DataFrame/SQL API (`spark`). |
| **SparkContext** | Entry point for the low-level RDD API (`sc`). |
| **Transformation** | Lazy op that returns a new RDD/DataFrame (`map`, `filter`, `select`). Nothing runs yet. |
| **Action** | Eager op that triggers execution and returns a result (`count`, `collect`, `show`, `save`). |
| **Lazy evaluation** | Spark builds a DAG of transformations, executes only when an action fires. |
| **Partition** | A chunk of the data processed in parallel by one task. |
| **Shuffle** | Redistributing data across partitions (e.g. `groupBy`, `join`) — expensive, crosses the network. |

---

## Architecture

```text
User Code
   │
   ▼
 Driver ── builds DAG, splits into stages, schedules tasks
   │
   ▼
Cluster Manager ── allocates resources (YARN / K8s / Standalone / local)
   │
   ▼
Executors ── run tasks on data partitions, cache data in memory
   │
   ▼
Results collected by Driver
```

| Component | Role |
|---|---|
| **Driver** | Orchestrates the app, builds the execution plan, schedules work. |
| **Executor** | Runs on a worker node; executes tasks, holds cached data. |
| **Cluster Manager** | Allocates executors (YARN, Kubernetes, Standalone, or `local[*]`). |
| **DAG** | Logical plan of transformations, optimized into stages. |
| **Job → Stage → Task** | Action triggers a **Job**; split into **Stages** at shuffle boundaries; each stage = **Tasks** (one per partition). |

---

## Setup (Docker + Jupyter)

```bash
# jupyter/pyspark-notebook image — Spark + PySpark preinstalled
docker compose up -d
# Open Jupyter at http://localhost:8888 (use the printed token)
# Spark UI (per running app) at http://localhost:4040
```

```yaml
# docker-compose.yml (minimal)
services:
  spark-jupyter:
    image: jupyter/pyspark-notebook:latest
    ports:
      - "8888:8888"   # Jupyter
      - "4040:4040"   # Spark UI
    volumes:
      - ./notebooks:/home/jovyan/work
```

---

## Entry Points

```python
# SparkSession — for DataFrames & SQL (the modern default)
from pyspark.sql import SparkSession

spark = (SparkSession.builder
         .appName("MyApp")
         .master("local[*]")                      # all local cores
         .config("spark.default.parallelism", "4")
         .getOrCreate())

print(spark.version)

# SparkContext — for RDDs (created automatically by the session)
sc = spark.sparkContext
# or standalone:
from pyspark import SparkContext
sc = SparkContext.getOrCreate()

spark.stop()   # release resources when done
```

`getOrCreate()` reuses an existing session/context instead of erroring — safe to call repeatedly in notebooks.

---

## RDD API

Use RDDs for **unstructured/raw data** or fine-grained control. Otherwise prefer DataFrames.

```python
rdd = sc.parallelize([1, 2, 3, 4, 5, 6])   # from a Python collection
rdd = sc.textFile("data.txt")              # from a file (one line = one element)
```

### Transformations (lazy — return a new RDD)
```python
rdd.map(lambda x: x * 2)                    # 1-to-1
rdd.filter(lambda x: x % 2 == 0)            # keep matching
rdd.flatMap(lambda line: line.split(" "))   # 1-to-many, flattened
rdd.distinct()                              # dedupe
rdd.reduceByKey(lambda a, b: a + b)         # aggregate per key (on (k, v) pairs)
rdd.groupByKey()                            # group values per key (heavier shuffle)
rdd.sortBy(lambda x: x)
rdd1.union(rdd2)
rdd1.cogroup(rdd2)                          # group both RDDs by shared key
```

### Actions (eager — trigger execution)
```python
rdd.collect()          # ALL elements to driver — avoid on big data
rdd.count()
rdd.first()
rdd.take(3)            # first n
rdd.reduce(lambda a, b: a + b)
rdd.saveAsTextFile("output/")   # one file per partition
```

### Partitions
```python
rdd.getNumPartitions()                 # inspect
sc.textFile("data.txt", minPartitions=4)
rdd.repartition(8)                     # full shuffle, increase/decrease
rdd.coalesce(2)                        # decrease only, avoids full shuffle
```
- Small local files → typically **2 partitions**; HDFS → ~1 partition per 128 MB block.
- `saveAsTextFile()` writes **one output file per partition**.

### Word Count (the "Hello World" of Spark)
```python
counts = (sc.textFile("data.txt")
          .flatMap(lambda line: line.split(" "))
          .map(lambda word: (word, 1))
          .reduceByKey(lambda a, b: a + b))
counts.collect()   # [('hello', 2), ('world', 1)]
```

---

## DataFrame API

### Create
```python
# From a Python collection
df = spark.createDataFrame([("Alice", 34), ("Bob", 45)], ["name", "age"])

# From files — shortcut readers
df = spark.read.csv("path.csv", header=True, inferSchema=True)
df = spark.read.json("path.json")
df = spark.read.parquet("path.parquet")

# Generic reader — .format(...).option(...).load(...)  (works for any source)
df = (spark.read
      .format("csv")                       # csv | json | parquet | orc | jdbc | delta | iceberg ...
      .option("header", "true")
      .option("inferSchema", "true")
      .load("path.csv"))

# Read a whole folder or a glob — Spark reads all matching files in parallel
df = spark.read.parquet("s3a://bucket/events/")
df = spark.read.json("data/2024-*/*.json")

# JDBC (databases)
df = (spark.read.format("jdbc")
      .option("url", "jdbc:postgresql://host:5432/db")
      .option("dbtable", "public.employees")
      .option("user", "u").option("password", "p")
      .load())

# From an RDD
df = sc.parallelize([("Alice", 34)]).toDF(["name", "age"])
```

### Explicit schema (faster & safer than `inferSchema`)
```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("name", StringType(), True),
    StructField("age",  IntegerType(), True),
])
df = spark.createDataFrame(data, schema)
```

### Inspect
```python
df.show(5)          # print first 5 rows
df.printSchema()    # column names + types
df.columns          # list of names
df.count()
df.describe().show()
```

### Transform
```python
from pyspark.sql import functions as F

df.select("name", "salary")
df.filter(df.age > 25)                          # or df.where(...)
df.filter((df.age > 25) & (df.dept == "eng"))   # & | ~, parenthesize each condition
df.withColumn("bonus", df.salary * 0.1)         # add/replace column
df.withColumnRenamed("dept", "department")
df.drop("temp_col")
df.orderBy(F.desc("salary"))
df.dropDuplicates()
df.na.drop()                                    # drop rows with nulls
df.na.fill(0)                                   # fill nulls
df.withColumn("age", df.age.cast("integer"))    # change type

# F.expr() — write a column as a SQL expression string
df.withColumn("bonus", F.expr("salary * 0.1"))
df.withColumn("tag", F.expr("CASE WHEN age > 30 THEN 'senior' ELSE 'junior' END"))
df.filter(F.expr("age > 25 AND dept = 'eng'"))

# selectExpr() — select with SQL expressions in one shot
df.selectExpr("name", "salary * 12 AS annual", "upper(dept) AS dept")

# when / otherwise — conditional column (the DataFrame-native CASE WHEN)
df.withColumn("level", F.when(df.age > 30, "senior")
                        .when(df.age > 20, "mid")
                        .otherwise("junior"))

# withColumns / withColumnsRenamed — batch versions (Spark 3.3+/3.4+)
df.withColumns({"a": F.lit(1), "b": F.col("salary") * 2})
df.withColumnsRenamed({"dept": "department", "age": "years"})

# Chain transformations (df.transform lets you compose reusable functions)
df.filter(df.age > 25).select("name", "salary").show()
df.transform(lambda d: d.filter(d.age > 25)).transform(lambda d: d.dropDuplicates())
```

### Aggregate (KPIs)
```python
from pyspark.sql import functions as F

df.groupBy("dept").agg(F.sum("salary").alias("total_salary")).show()
df.groupBy("dept").agg(F.avg("age").alias("avg_age")).show()
df.groupBy("dept").count().show()
df.groupBy().agg(F.avg("salary")).show()        # global aggregate (no grouping)
```

Common `F` functions: `sum, avg, count, min, max, countDistinct, round, upper, lower, trim, when, col, lit, expr, concat, split, explode, coalesce, to_date, year, date_format, regexp_replace, broadcast`.

### Write
```python
# Shortcut writers
df.write.mode("overwrite").parquet("out/")       # modes: append, overwrite, ignore, error(default)
df.write.mode("overwrite").csv("out/", header=True)
df.write.partitionBy("dept").parquet("out/")     # partitioned output

# Generic writer — .format(...).option(...).save(...)  (mirror of read)
(df.write
   .format("parquet")                            # parquet | csv | json | orc | jdbc | delta ...
   .mode("overwrite")
   .option("compression", "snappy")
   .save("out/"))

# Save as a managed/metastore table (queryable via spark.sql)
df.write.mode("overwrite").saveAsTable("db.employees")

# Control the number of output files
df.coalesce(1).write.csv("out/", header=True)    # single file (small data only)

# JDBC sink
(df.write.format("jdbc")
   .option("url", "jdbc:postgresql://host:5432/db")
   .option("dbtable", "public.results")
   .option("user", "u").option("password", "p")
   .mode("append").save())
```

---

## Spark SQL

Query DataFrames with SQL by registering a **temporary view** (session-scoped, no disk write).

```python
df.createOrReplaceTempView("employees")

spark.sql("SELECT name, age FROM employees WHERE age > 30").show()
spark.sql("SELECT dept, AVG(salary) AS avg_sal FROM employees GROUP BY dept").show()

spark.catalog.dropTempView("employees")          # remove explicitly
df.createGlobalTempView("emp")                    # survives across sessions (global_temp.emp)
```

`spark.sql(...)` returns a DataFrame — you can chain the DataFrame API onto it.

### Joins
```python
# DataFrame API
df.join(other, "key", "inner")                   # join on shared column name
df.join(other, df.id == other.emp_id, "left")    # explicit condition

# Spark SQL
spark.sql("""
  SELECT a.name, b.dept
  FROM employees a
  INNER JOIN departments b ON a.dept_id = b.id
""").show()
```

| Join type | Returns |
|---|---|
| `inner` | Only matching keys in both. |
| `left` / `right` | All rows from that side + matches from the other (nulls if none). |
| `outer` (full) | All rows from both; nulls where no match. |
| `cross` | Cartesian product (every combination). |
| `semi` | Left rows that HAVE a match (left columns only). |
| `anti` | Left rows that have NO match. |

### Window Functions
Row-wise calculations across partitions **without collapsing rows** (unlike `groupBy`).

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import rank, row_number, lag, lead, sum as _sum, desc

w = Window.partitionBy("dept").orderBy(desc("salary"))
df.withColumn("salary_rank", rank().over(w)).show()

# SQL equivalent
spark.sql("""
  SELECT name, salary,
         RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS salary_rank
  FROM employees
""").show()
```

Common window functions: `RANK(), DENSE_RANK(), ROW_NUMBER(), LAG(), LEAD(), SUM(), AVG()`.

---

## ETL Pattern (Extract → Transform → Load)

```python
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName("ETLProject").getOrCreate()

# EXTRACT
df = spark.read.csv("tracks.csv", header=True, inferSchema=True)
df.printSchema()                                 # validate types before proceeding
df.show(5)

# TRANSFORM
df = (df.na.drop()                               # remove nulls
        .dropDuplicates()                        # remove duplicates
        .withColumn("duration_min", F.col("duration_ms") / 60000))

kpis = df.groupBy("artists").agg(
    F.count("*").alias("track_count"),
    F.avg("popularity").alias("avg_popularity"),
)

# LOAD
kpis.write.mode("overwrite").parquet("output/kpis/")

spark.stop()
```

---

## More Column & DataFrame Ops

```python
from pyspark.sql import functions as F

# Combine DataFrames
df1.union(df2)                       # by position — schemas must line up
df1.unionByName(df2, allowMissingColumns=True)   # by column name (safer)

# Pivot / unpivot
df.groupBy("dept").pivot("year").agg(F.sum("salary"))
df.unpivot(["id"], ["jan", "feb"], "month", "value")   # melt() is an alias (3.4+)

# Arrays & structs
df.withColumn("words", F.split("text", " "))         # string -> array
df.withColumn("word", F.explode("words"))            # array -> one row per element
df.select("addr.city")                               # dot access into a struct
df.withColumn("arr", F.array("a", "b"))
df.withColumn("s", F.struct("a", "b"))

# Sampling / limiting
df.limit(100)
df.sample(fraction=0.1, seed=42)
df.distinct()

# Dates & timestamps
df.withColumn("d", F.to_date("ts"))
df.withColumn("y", F.year("ts"))
df.withColumn("now", F.current_timestamp())
```

---

## UDFs

Prefer built-in `F.*` functions — they run inside the JVM and are optimizable. Reach for a UDF only when no built-in fits.

```python
from pyspark.sql import functions as F
from pyspark.sql.types import IntegerType

# Regular Python UDF (row-at-a-time — slowest; serializes to Python)
@F.udf(returnType=IntegerType())
def add_one(x):
    return x + 1
df.withColumn("plus1", add_one("age"))

# Pandas UDF (vectorized — much faster, operates on batches via Arrow)
import pandas as pd
@F.pandas_udf("double")
def pct(s: pd.Series) -> pd.Series:
    return s / s.sum()
df.withColumn("share", pct("salary"))

# applyInPandas — whole-group as a pandas DataFrame (grouped map)
def normalize(pdf: pd.DataFrame) -> pd.DataFrame:
    pdf["z"] = (pdf.salary - pdf.salary.mean()) / pdf.salary.std()
    return pdf
df.groupBy("dept").applyInPandas(normalize, schema="... z double")
```

---

## Spark Connect (introduced 3.4)

Thin client/server architecture — your code talks to a remote Spark cluster over gRPC, no local JVM needed. The DataFrame API is unchanged; only how you build the session differs.

- **3.4**: PySpark DataFrame API coverage over Connect
- **3.5**: Scala client reaches GA; Structured Streaming + Pandas-API parity over Connect

```python
spark = SparkSession.builder.remote("sc://localhost:15002").getOrCreate()
```

---

## Performance & Caching

```python
df.cache()                    # persist in memory across actions (avoids recompute)
df.persist()                  # like cache, with configurable storage level
df.unpersist()                # release it

df.explain()                  # inspect the physical plan (spot shuffles/scans)
df.explain("formatted")       # more readable plan

# Broadcast a small table into a join — avoids the big shuffle
from pyspark.sql.functions import broadcast
big.join(broadcast(small), "key")

# Key runtime configs
spark.conf.set("spark.sql.shuffle.partitions", 200)      # partitions after a shuffle (default 200)
spark.conf.set("spark.sql.adaptive.enabled", True)       # AQE — on by default since 3.0
```

- **Adaptive Query Execution (AQE)** is on by default (3.0+): re-optimizes the plan at runtime — coalesces shuffle partitions, switches join strategies, and handles skew. Usually leave it on.
- **Broadcast join** when one side is small (< ~10 MB by default) — ships it to every executor, skipping the shuffle. AQE also triggers this automatically.
- **`spark.sql.shuffle.partitions`** (default 200) governs partition count after shuffles — too high = many tiny tasks, too low = huge tasks. AQE auto-tunes it, but override for very small or very large jobs.
- **Cache** only when you reuse a DataFrame/RDD multiple times — caching once-used data wastes memory.
- **Avoid `collect()`** on large data — it pulls everything to the driver and can OOM. Use `take(n)` / `show()`.
- **Minimize shuffles**: `reduceByKey` beats `groupByKey`; filter early, select only needed columns (column pruning).
- **Prefer DataFrames over RDDs** — the Catalyst optimizer + Tungsten give big speedups RDDs don't get.
- **Parquet over CSV** for storage — columnar, compressed, carries schema, supports predicate pushdown.
- Watch the **Spark UI** (`:4040`) to see the DAG, stages, and shuffle sizes.

---

## When to Use What

| Use | For |
|---|---|
| **RDD** | Unstructured/raw data, custom low-level logic, fine partition control. |
| **DataFrame** | Structured data, best performance (Catalyst), most everyday work. **Default choice.** |
| **Spark SQL** | Analysts comfortable with SQL; complex joins/aggregations read more clearly as SQL. |

DataFrame API and Spark SQL compile to the **same optimized plan** — pick whichever reads better; there's no performance difference.