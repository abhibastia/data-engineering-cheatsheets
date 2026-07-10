# Spark (PySpark) Cheatsheet

Apache Spark is a distributed, in-memory data processing engine. Unlike Pandas (single machine), Spark **parallelizes** computation across a cluster and only executes when an **action** is called (lazy evaluation).

> **Version baseline:** examples target **Spark 3.5** (LTS, security fixes to Nov 2027) — the runtime used by AWS Glue 5.0. Spark **4.x** is the current release line; the DataFrame/SQL APIs shown here carry forward unchanged. Version gates are noted inline (e.g. `3.4+`).

## Contents

1. [Core Concepts](#core-concepts)
2. [Architecture](#architecture)
3. [Setup with Docker and Jupyter](#setup-with-docker-and-jupyter)
4. [Entry Points](#entry-points)
5. [RDD API](#rdd-api)
6. [DataFrame API](#dataframe-api)
7. [Spark SQL](#spark-sql) — joins, join strategies, window functions
8. [ETL Pattern](#etl-pattern-extract-transform-load)
9. [More Column and DataFrame Ops](#more-column-and-dataframe-ops)
10. [UDFs](#udfs)
11. [Spark Connect](#spark-connect-introduced-34)
12. [Reading Query Plans (`explain`)](#reading-query-plans-explain)
13. [Performance and Caching](#performance-and-caching)
14. [When to Use What](#when-to-use-what)

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

## Setup with Docker and Jupyter

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

#### `select` vs `withColumn`

Both add/derive columns and both are lazy transformations producing the same optimized plan for equivalent output — the difference is **shape** and **scale**.

| | `select` / `selectExpr` | `withColumn` |
|---|---|---|
| Output columns | Exactly what you list (a projection) | All existing columns **+** the new/replaced one |
| Adds many cols | One call, list them all | One call **per** column |
| Drops/reorders | Yes — you choose the full set | No — keeps everything |
| Replace a column | List it with the new expr | `withColumn("x", ...)` with the same name |

```python
# Same result, two styles:
df.withColumn("bonus", F.col("salary") * 0.1)                 # keep all cols + bonus
df.select("*", (F.col("salary") * 0.1).alias("bonus"))       # explicit equivalent

# Adding several derived columns — select does it in ONE projection:
df.select(
    "id", "name",
    (F.col("salary") * 12).alias("annual"),
    F.upper("dept").alias("dept"),
)
# vs. chaining withColumn (readable, but see the gotcha below):
(df.withColumn("annual", F.col("salary") * 12)
   .withColumn("dept", F.upper("dept")))
```

**The gotcha:** each `withColumn` wraps the plan in another `Project` node. Chaining **hundreds** of them (e.g. in a loop) builds a deeply nested logical plan that makes Catalyst analysis slow — and can blow the stack. For many columns, use a single `select`/`withColumns` (3.3+) instead of a long `withColumn` chain.

```python
# BAD at scale — N nested projections
for c in many_cols:
    df = df.withColumn(c, F.col(c).cast("double"))

# GOOD — one projection
df = df.select(*[F.col(c).cast("double").alias(c) if c in many_cols else c
                 for c in df.columns])
# or (Spark 3.3+): df = df.withColumns({c: F.col(c).cast("double") for c in many_cols})
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

Aliases Spark accepts for the `how` argument: `inner`; `left`/`left_outer`; `right`/`right_outer`; `outer`/`full`/`fullouter`/`full_outer`; `cross`; `leftsemi`/`semi`; `leftanti`/`anti`.

**Join gotchas**
```python
# Ambiguous columns after a join on a condition (both sides keep their key col).
# Join on a STRING key name instead — Spark keeps a single merged key column:
df.join(other, "id", "inner")                    # one `id` column in the result
df.join(other, ["id", "region"], "inner")        # multi-column equi-join, merged

# Null keys never match in an equi-join (SQL semantics). Use eqNullSafe for null==null:
df.join(other, df.k.eqNullSafe(other.k))         # <=> operator

# Duplicate rows explode a join — dedupe/aggregate the smaller side first.
```

### Join Strategies (physical join algorithms)

The join *type* (inner/left/…) is **what** to return; the join *strategy* is **how** Spark computes it. Catalyst (+ AQE) picks one based on table sizes, join keys, and hints. Read it off `df.explain()`.

| Strategy | When Spark picks it | Cost | Notes |
|---|---|---|---|
| **Broadcast Hash Join (BHJ)** | One side ≤ `autoBroadcastJoinThreshold` (default **10 MB**), equi-join. | Cheapest — **no shuffle**. | Small side is broadcast to every executor, built into a hash table. AQE can trigger it at runtime once it knows real sizes. |
| **Shuffle Sort-Merge Join (SMJ)** | Both sides large, equi-join on sortable keys. **Default** for big-vs-big. | Shuffles + sorts both sides. | Robust and memory-stable; the workhorse for large joins. |
| **Shuffle Hash Join (SHJ)** | One side fits in memory per partition after shuffle, equi-join. | Shuffles both; builds a hash table (no sort). | Off by default vs SMJ unless `spark.sql.join.preferSortMergeJoin=false`. Can OOM if the build side is underestimated. |
| **Broadcast Nested Loop Join (BNLJ)** | Non-equi joins (`<`, `>`, `between`) or no join key, with one small side. | O(n·m) but broadcasts the small side. | Fallback for range/inequality joins. |
| **Cartesian (Shuffle-Replicate NL)** | Cross join / non-equi with no broadcastable side. | O(n·m), fully shuffled. | Almost always a mistake on big data — check for a missing join key. |

```python
# Force a strategy with a hint (overrides the optimizer). SQL names in the hint:
#   BROADCAST | MERGE | SHUFFLE_HASH | SHUFFLE_REPLICATE_NL
from pyspark.sql.functions import broadcast
big.join(broadcast(small), "key")                        # force BHJ (DataFrame API)

df.hint("broadcast").join(other, "key")                  # hint form
spark.sql("SELECT /*+ BROADCAST(d) */ * FROM emp e JOIN dept d ON e.dept_id = d.id")
spark.sql("SELECT /*+ MERGE(a, b) */ * FROM a JOIN b ON a.k = b.k")   # force SMJ

# Broadcast threshold — raise to broadcast bigger dims, set -1 to disable BHJ:
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 50 * 1024 * 1024)   # 50 MB
```

- **Only equi-joins** (`=` / `eqNullSafe`) can use BHJ, SMJ, or SHJ. Any inequality falls back to BNLJ or Cartesian — expensive.
- Broadcasting a side that's *not* actually small risks a driver/executor OOM. Trust AQE's runtime sizing over a hardcoded `broadcast()` when the size is uncertain.

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

## ETL Pattern (Extract, Transform, Load)

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

## More Column and DataFrame Ops

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

## Reading Query Plans (`explain`)

`explain()` shows how Catalyst will actually run your query — use it to spot shuffles (`Exchange`), the chosen join strategy, scans, and whether filters/columns were pushed into the source.

```python
df.explain()                  # physical plan only (default)
df.explain(True)              # ALL phases: parsed → analyzed → optimized → physical
df.explain("formatted")       # physical plan, tree + per-node detail (most readable, 3.0+)
df.explain("cost")            # optimized logical plan WITH row/size stats
df.explain("extended")        # same as explain(True)
```

**The four plan phases** (`explain(True)` prints all of them — this *is* the Catalyst pipeline):

| Phase | What it is |
|---|---|
| **Parsed Logical Plan** | Raw plan from your code/SQL — column/table names not yet resolved. |
| **Analyzed Logical Plan** | Names + types resolved against the catalog (fails here on typos / missing cols). |
| **Optimized Logical Plan** | Catalyst rules applied: predicate/projection pushdown, constant folding, filter reordering. |
| **Physical Plan** | Executable operators chosen (join strategy, `Exchange` shuffles, scans). The one Spark runs. |

**What to look for in the physical plan:**

```text
== Physical Plan ==
*(2) BroadcastHashJoin ...        # ← join strategy actually used
+- *(2) Filter (age > 25)         # ← was the filter pushed down?
   +- Exchange hashpartitioning   # ← a SHUFFLE — the expensive part
      +- FileScan parquet ...
            PushedFilters: [GreaterThan(age,25)]   # ← predicate pushdown working
            DataFilters / PartitionFilters: [...]  # ← partition pruning working
```
- `Exchange` = a shuffle (network + disk). Fewer is better.
- `*(n)` = **whole-stage codegen** — operators fused into one generated Java method (fast). Missing `*` = falling back to interpreted (e.g. some UDFs).
- `BroadcastHashJoin` vs `SortMergeJoin` tells you which join algorithm Catalyst chose (see Join Strategies).
- `PushedFilters` / `PartitionFilters` present = Spark reads less data at the source.

---

## Performance and Caching

```python
df.cache()                    # persist in memory across actions (avoids recompute)
df.persist()                  # like cache, with configurable storage level
df.unpersist()                # release it

from pyspark import StorageLevel
df.persist(StorageLevel.MEMORY_AND_DISK)   # spill to disk if it won't fit in RAM
# levels: MEMORY_ONLY, MEMORY_AND_DISK (default for DataFrames), DISK_ONLY,
#         MEMORY_ONLY_SER, MEMORY_AND_DISK_SER, *_2 (replicated to 2 nodes)

# Broadcast a small table into a join — avoids the big shuffle
from pyspark.sql.functions import broadcast
big.join(broadcast(small), "key")

# Key runtime configs
spark.conf.set("spark.sql.shuffle.partitions", 200)      # partitions after a shuffle (default 200)
spark.conf.set("spark.sql.adaptive.enabled", True)       # AQE — on by default since 3.2
```

### Repartition vs Coalesce
```python
df.repartition(200)                  # full shuffle; increase OR decrease; even sizes
df.repartition("dept")               # hash-partition by column (co-locates keys)
df.repartition(200, "dept")          # both count and key
df.coalesce(10)                      # decrease only; NO shuffle (merges partitions) — may skew
```
- Use `coalesce` to shrink partition count cheaply before a write (e.g. avoid thousands of tiny files).
- Use `repartition` to grow parallelism, rebalance skew, or pre-partition by a join/group key.

### Handling Data Skew
One hot key sends most rows to a single task — that straggler dominates the stage.
```python
# 1. Let AQE fix it (default on in 3.2+): detects + splits skewed partitions automatically.
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", True)

# 2. Salting — spread a hot key across N buckets, then join on (key, salt) and aggregate up.
from pyspark.sql import functions as F
N = 16
big  = big.withColumn("salt", (F.rand() * N).cast("int"))
small = small.crossJoin(spark.range(N).withColumnRenamed("id", "salt"))
big.join(small, ["key", "salt"])
```

### Bucketing (pre-shuffled tables)
```python
# Write data pre-hashed by a key so future joins/aggregations on it skip the shuffle.
(df.write.bucketBy(50, "user_id").sortBy("user_id")
   .mode("overwrite").saveAsTable("events_bucketed"))
```
Joining two tables bucketed the same way on the same key → no `Exchange`.

### Optimization checklist
- **Adaptive Query Execution (AQE)** — on by default since **3.2** (existed but *off* in 3.0/3.1). Re-optimizes at runtime: coalesces shuffle partitions, converts SMJ→BHJ once real sizes are known, and splits skewed joins. Usually leave it on.
- **Broadcast join** when one side is small (< ~10 MB by default) — ships it to every executor, skipping the shuffle. AQE also triggers this automatically.
- **`spark.sql.shuffle.partitions`** (default 200) governs partition count after shuffles — too high = many tiny tasks, too low = huge tasks. AQE auto-tunes it, but override for very small or very large jobs.
- **Predicate pushdown** — `filter()`/`where()` early so Parquet/ORC/JDBC skip rows at the source. Verify with `PushedFilters` in `explain()`.
- **Partition pruning** — filter on a `partitionBy` column and Spark reads only those directories (`PartitionFilters` in the plan). Dynamic partition pruning (3.0+) prunes based on a joined dimension.
- **Column pruning** — `select()` only the columns you need; columnar formats then read fewer bytes.
- **Cache** only when you reuse a DataFrame/RDD multiple times — caching once-used data wastes memory.
- **Avoid `collect()`** on large data — it pulls everything to the driver and can OOM. Use `take(n)` / `show()`.
- **Minimize shuffles**: `reduceByKey` beats `groupByKey`; filter early, select only needed columns.
- **Prefer DataFrames over RDDs** — the Catalyst optimizer + Tungsten (off-heap memory, whole-stage codegen) give big speedups RDDs don't get.
- **Parquet/ORC over CSV/JSON** — columnar, compressed, carries schema, supports predicate pushdown and column pruning.
- **Avoid Python UDFs** where a built-in `F.*` exists — UDFs break codegen and serialize row-by-row to Python; prefer `pandas_udf` if you must.
- Watch the **Spark UI** (`:4040`) — DAG, stages, shuffle read/write sizes, and skewed/straggler tasks.

---

## When to Use What

| Use | For |
|---|---|
| **RDD** | Unstructured/raw data, custom low-level logic, fine partition control. |
| **DataFrame** | Structured data, best performance (Catalyst), most everyday work. **Default choice.** |
| **Spark SQL** | Analysts comfortable with SQL; complex joins/aggregations read more clearly as SQL. |

DataFrame API and Spark SQL compile to the **same optimized plan** — pick whichever reads better; there's no performance difference.