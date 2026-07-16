# MongoDB Cheatsheet

MongoDB is a document database that stores data as **BSON** (binary JSON) documents inside **collections**. Unlike a relational table, a collection does not enforce a fixed schema — documents in the same collection can have different fields. It scales horizontally through sharding and provides high availability through replica sets.

> **Version baseline:** examples target **MongoDB 8.x** (server) with **`mongosh`** (the modern shell, not the removed legacy `mongo`) and **PyMongo 4.x** (Python driver). The CRUD, indexing, and aggregation APIs shown here are stable across 6.0/7.0/8.0. Avoid `mongo:6.0` in new work — the 6.0 series is end-of-life; pin `mongo:8.0` (or newer) instead.

## Contents

1. [Core Concepts](#core-concepts)
2. [NoSQL vs RDBMS](#nosql-vs-rdbms)
3. [CAP Theorem & MongoDB](#cap-theorem--mongodb)
4. [BSON Data Types](#bson-data-types)
5. [Setup with Docker](#setup-with-docker)
6. [Connecting](#connecting)
7. [CRUD — Shell & PyMongo](#crud--shell--pymongo)
8. [Query Operators](#query-operators)
9. [Update Operators](#update-operators)
10. [Indexing](#indexing)
11. [Aggregation Framework](#aggregation-framework)
12. [Schema Design](#schema-design-embedding-vs-referencing)
13. [Write Concern & Read Preference](#write-concern--read-preference)
14. [Import / Export / Backup](#import--export--backup)
15. [Advanced Features](#advanced-features)
16. [Gotchas](#gotchas)

---

## Core Concepts

| Concept | What it is |
|---|---|
| **Document** | A single record — a BSON object like `{"name": "Alice", "age": 25}`. Max size **16 MB**. |
| **Collection** | A group of documents (≈ a table). Schema-less — documents need not share fields. |
| **Database** | A container for collections. |
| **`_id`** | Primary key, unique per collection. Auto-generated as an `ObjectId` if you don't supply one. |
| **Replica set** | A group of `mongod` nodes holding the same data: one **primary** (all writes) + **secondaries** (replicate the oplog). Gives failover + durability. |
| **Sharding** | Horizontal partitioning of a collection across shards by a **shard key**, for datasets/throughput beyond one machine. |
| **WiredTiger** | Default storage engine (since 3.2) — document-level concurrency, compression (snappy default), journaling, checkpoints. |
| **`mongosh`** | The MongoDB Shell — a JavaScript REPL for queries and admin. Replaced the legacy `mongo` shell. |
| **Driver** | Language library (PyMongo, Node, Java…) for using MongoDB from application code. |

---

## NoSQL vs RDBMS

| | RDBMS (e.g. PostgreSQL) | MongoDB (document store) |
|---|---|---|
| **Structure** | Rows/columns, fixed schema | Flexible BSON documents |
| **Schema** | Enforced up front (`CREATE TABLE`) | Optional (enforce with `$jsonSchema` validation if wanted) |
| **Joins** | First-class (`JOIN`) | `$lookup` in aggregation; often modeled by embedding instead |
| **Scaling** | Primarily vertical | Horizontal (sharding) built in |
| **Transactions** | ACID, mature | ACID multi-document transactions since 4.0 (replica set) / 4.2 (sharded) |
| **Best for** | Highly relational data, complex ad-hoc joins | Evolving schemas, nested/hierarchical data, high write throughput |

---

## CAP Theorem & MongoDB

The **CAP theorem** says that during a network **partition**, a distributed system can guarantee only **Consistency** *or* **Availability**, not both. (Partition tolerance is not optional for a distributed database — networks fail — so the real choice is C vs A when a partition happens.)

- **Consistency** — every read sees the most recent write.
- **Availability** — every request gets a (non-error) response, even if possibly stale.
- **Partition tolerance** — the system keeps working despite dropped/delayed messages between nodes.

**Where MongoDB sits:** MongoDB is a **CP** system by default. Each replica set has a single primary that takes all writes; if the primary is partitioned away from the majority, it steps down and **rejects writes** until a new primary is elected — choosing consistency over availability. It is **tunable**: write concern and read preference/read concern let you trade toward availability (e.g. reading from secondaries) or toward stronger consistency (`w: "majority"`, `readConcern: "majority"`).

> ⚠️ A common oversimplification (including in some tutorials) labels MongoDB "AP." That's inaccurate for the default configuration — the single-primary design makes it **CP**. See MongoDB's own [CAP explainer](https://www.mongodb.com/resources/basics/databases/cap-theorem).

---

## BSON Data Types

BSON extends JSON with extra types. Common ones:

| Type | Shell literal | Use case |
|---|---|---|
| **ObjectId** | `ObjectId()` | Auto `_id`; encodes a creation timestamp |
| **String** | `"David"` | Text (UTF-8) |
| **Int32 / Int64** | `NumberInt(25)` / `NumberLong(25)` | Whole numbers |
| **Double** | `3.14` | Floating point (default JS number) |
| **Decimal128** | `NumberDecimal("75000.50")` | Exact decimals — money, no float rounding |
| **Boolean** | `true` / `false` | Flags |
| **Date** | `ISODate("1990-01-01")` / `new Date()` | Timestamps (ms since epoch, UTC) |
| **Array** | `["Python", "SQL"]` | Lists / one-to-many within a doc |
| **Embedded doc** | `{"city": "Tokyo", "zip": "100-0001"}` | Nested structure |
| **Binary** | `BinData(0, ...)` | Files, hashes, UUIDs |
| **Null** | `null` | Missing/unknown |

```javascript
db.users.insertOne({
  name: "David",
  birthdate: ISODate("1990-01-01"),
  skills: ["Python", "SQL"],
  address: { city: "Tokyo", zip: "100-0001" },
  salary: NumberDecimal("75000.50"),   // exact, not a float
  age: NumberInt(30)
})
```

**Use `NumberDecimal` for currency** — plain JS numbers are 64-bit floats and lose precision (`0.1 + 0.2 !== 0.3`).

---

## Setup with Docker

```yaml
# docker-compose.yml
services:
  mongodb:
    image: mongo:8.0                 # pin a supported major; avoid EOL 6.0
    container_name: mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: rootpassword
    volumes:
      - ./nosql-data:/data/db        # persist data across restarts
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.runCommand('ping').ok", "--quiet"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
```

```bash
docker compose up -d                 # start
docker ps                            # verify container is healthy
docker compose down                  # stop & remove containers (data persists in ./nosql-data)

# open a shell inside the container
docker exec -it mongodb mongosh "mongodb://root:rootpassword@localhost:27017/nosql_db?authSource=admin"
```

`authSource=admin` is required — the root user is created in the `admin` database, so auth must be checked there even when you connect to `nosql_db`.

---

## Connecting

### mongosh
```javascript
use nosql_db          // switch to (and lazily create) the database
db                    // show current database name
show collections
```

### PyMongo
```python
# pip install pymongo
from pymongo import MongoClient

client = MongoClient("mongodb://root:rootpassword@localhost:27017/?authSource=admin")
db = client["nosql_db"]                     # DB created lazily on first write

print("Connected to:", db.name)
print("Collections:", db.list_collection_names())
```

**Connection string notes:**
- From another Docker container (e.g. a Jupyter container), use `host.docker.internal` instead of `localhost`.
- The database and a collection are **created lazily** — they don't exist until you insert the first document.
- Reuse one `MongoClient` for the app's lifetime; it's thread-safe and pools connections. Don't create one per request.

---

## CRUD — Shell & PyMongo

| Operation | mongosh | PyMongo |
|---|---|---|
| Insert one | `db.users.insertOne({...})` | `db.users.insert_one({...})` |
| Insert many | `db.users.insertMany([...])` | `db.users.insert_many([...])` |
| Find many | `db.users.find({...})` | `db.users.find({...})` |
| Find one | `db.users.findOne({...})` | `db.users.find_one({...})` |
| Update one | `db.users.updateOne(f, u)` | `db.users.update_one(f, u)` |
| Update many | `db.users.updateMany(f, u)` | `db.users.update_many(f, u)` |
| Replace | `db.users.replaceOne(f, doc)` | `db.users.replace_one(f, doc)` |
| Delete one | `db.users.deleteOne({...})` | `db.users.delete_one({...})` |
| Delete many | `db.users.deleteMany({...})` | `db.users.delete_many({...})` |
| Count | `db.users.countDocuments({...})` | `db.users.count_documents({...})` |

### mongosh
```javascript
use nosql_db

db.users.insertOne({ name: "Eve", age: 28, email: "eve@example.com" })
db.users.insertMany([
  { name: "Grace", age: 28, city: "Paris" },
  { name: "Henry", age: 35, city: "Tokyo" }
])

db.users.find({ city: "Tokyo" })                      // filter
db.users.find({}, { name: 1, _id: 0 })                // projection: name only
db.users.find().sort({ age: -1 }).limit(5)            // sort desc, cap at 5

db.users.updateOne({ name: "Eve" }, { $set: { age: 29 } })
db.users.updateMany({ city: "Tokyo" }, { $inc: { age: 1 } })

db.users.deleteOne({ email: "eve@example.com" })
db.users.countDocuments({ city: "Tokyo" })
```

### PyMongo
```python
result = db.users.insert_one({"name": "Frank", "age": 30, "city": "Berlin"})
print("Inserted ID:", result.inserted_id)

db.users.insert_many([
    {"name": "Grace", "age": 28, "city": "Paris"},
    {"name": "Henry", "age": 35, "city": "Tokyo"},
])

# find returns a lazy cursor — iterate it
for doc in db.users.find({"city": "Berlin"}, {"name": 1, "_id": 0}).sort("age", -1):
    print(doc)

# find_one by _id
from bson import ObjectId
db.users.find_one({"_id": ObjectId("68d15dfd81aaa17b26f449a9")})

db.users.update_one({"name": "Henry"}, {"$set": {"age": 36}})
db.users.update_many({"age": {"$gt": 25}}, {"$set": {"status": "active"}})

db.users.delete_one({"name": "Frank"})
db.users.delete_many({"city": "Paris"})

count = db.users.count_documents({"city": "Berlin"})
```

**Projection:** `{"name": 1, "_id": 0}` includes `name`, excludes `_id`. You can't mix includes and excludes in one projection (except explicitly excluding `_id`).

**Upsert** — insert if no match exists:
```python
db.users.update_one({"name": "Ivy"}, {"$set": {"age": 22}}, upsert=True)
```

---

## Query Operators

```python
# Comparison
db.users.find({"age": {"$gt": 25}})          # $gt $gte $lt $lte $ne $eq
db.users.find({"city": {"$in": ["Berlin", "Paris"]}})   # $in / $nin

# Logical
db.users.find({"$or": [{"name": "Bob"}, {"name": "Amit"}]})   # $and $or $not $nor

# Existence & type
db.users.find({"email": {"$exists": True}})
db.users.find({"age": {"$type": "int"}})

# Arrays
db.users.find({"hobbies": "coding"})         # array contains value
db.users.find({"hobbies": {"$all": ["coding", "gaming"]}})    # contains all
db.users.find({"hobbies": {"$size": 3}})     # exactly 3 elements

# Embedded documents — dot notation
db.users.find({"address.city": "London"})

# Regex
db.users.find({"name": {"$regex": "^A", "$options": "i"}})

# $expr — compare fields within the same document
db.users.find({"$expr": {"$gt": ["$spent", "$budget"]}})
```

---

## Update Operators

```python
db.users.update_one({"name": "Eve"}, {"$set":   {"age": 29}})       # set field
db.users.update_one({"name": "Eve"}, {"$unset": {"temp": ""}})      # remove field
db.users.update_one({"name": "Eve"}, {"$inc":   {"age": 1}})        # increment
db.users.update_one({"name": "Eve"}, {"$rename": {"city": "town"}}) # rename field

# Array operators
db.users.update_one({"name": "Amit"}, {"$push":     {"hobbies": "reading"}})   # append
db.users.update_one({"name": "Amit"}, {"$pull":     {"hobbies": "reading"}})   # remove matching
db.users.update_one({"name": "Amit"}, {"$addToSet": {"hobbies": "coding"}})    # append if absent
db.users.update_one({"name": "Amit"}, {"$pop":      {"hobbies": 1}})           # remove last (-1 = first)
```

### Bulk writes
Batch mixed operations into one round trip:
```python
import pymongo
db.users.bulk_write([
    pymongo.InsertOne({"name": "Bob", "age": 27, "city": "Chennai"}),
    pymongo.UpdateOne({"name": "Amit"}, {"$set": {"age": 26}}),
    pymongo.DeleteOne({"name": "Old"}),
])
```
```javascript
db.users.bulkWrite([
  { insertOne: { document: { name: "Bob", age: 27, city: "Chennai" } } },
  { updateOne: { filter: { name: "Amit" }, update: { $set: { age: 26 } } } }
])
```

---

## Indexing

Without an index, a query scans **every** document (`COLLSCAN`). An index is a B-tree that lets MongoDB jump straight to matching documents. Verify usage with `.explain("executionStats")`.

```python
# Basic (single field): 1 = ascending, -1 = descending
db.users.create_index([("age", 1)])

# Compound — order matters; supports a query's prefix
db.users.create_index([("city", 1), ("age", 1)])
#   serves: {city}  and {city, age}  and sort by city then age
#   does NOT serve: a query on {age} alone

# Unique — reject duplicate values
db.users.create_index([("email", 1)], unique=True)

# Partial — index only a subset (smaller, cheaper)
db.users.create_index([("age", 1)], partialFilterExpression={"age": {"$gt": 30}})

# TTL — auto-delete documents N seconds after their date field
db.logs.create_index([("createdAt", 1)], expireAfterSeconds=3600)

# Geospatial (GeoJSON) — needed for $near / $geoWithin
db.users.create_index([("location", "2dsphere")])

# Text — full-text keyword search
db.users.create_index([("bio", "text")])

# Inspect / drop
list(db.users.list_indexes())
db.users.drop_index("age_1")
```

```javascript
db.users.createIndex({ age: 1 })
db.users.createIndex({ city: 1, age: 1 })
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ age: 1 }, { partialFilterExpression: { age: { $gt: 30 } } })
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })
db.users.createIndex({ location: "2dsphere" })
db.users.createIndex({ bio: "text" })
db.users.getIndexes()
```

**Index rules to remember:**
- **Compound index prefix rule** — an index on `{a, b, c}` serves queries on `{a}`, `{a,b}`, `{a,b,c}`, but *not* on `{b}` or `{c}` alone.
- **TTL indexes:** single field only (never compound, never `_id`); the field must be a **Date** (or array of dates); the sweeper runs **~every 60 s**, so deletion isn't instant; on a replica set, only the **primary** deletes.
- **Text index:** at most **one per collection** (it may cover multiple fields). For anything beyond keyword matching, MongoDB now recommends **Atlas Search** over `$text`.
- Every index speeds reads but **slows writes** and uses storage/RAM. Drop unused ones.

### Geospatial & text queries
```python
# $near needs a 2dsphere index; results come back sorted by distance
db.users.find({
    "location": {
        "$near": {
            "$geometry": {"type": "Point", "coordinates": [77.2167, 28.6667]},
            "$maxDistance": 1000   # metres
        }
    }
})

# $text needs a text index
db.users.find({"$text": {"$search": "coding"}})
```
> `$near` is a query operator and **cannot** be used in an aggregation `$match`. Inside a pipeline, use the **`$geoNear`** stage (which must be the first stage) instead.

---

## Aggregation Framework

A pipeline passes documents through ordered **stages**, each transforming the stream. Think of it as MongoDB's `GROUP BY` + `JOIN` + ETL.

| Stage | Does | SQL equivalent |
|---|---|---|
| `$match` | Filter (put early — uses indexes when first) | `WHERE` |
| `$group` | Group by `_id`, compute accumulators (`$sum`, `$avg`, `$max`, `$push`…) | `GROUP BY` + aggregates |
| `$project` | Reshape: include/exclude/compute fields | `SELECT col1, col2, expr` |
| `$addFields` / `$set` | Add or overwrite fields, keep the rest | `SELECT *, expr AS new` |
| `$sort` | Order documents | `ORDER BY` |
| `$limit` / `$skip` | Paginate | `LIMIT` / `OFFSET` |
| `$unwind` | One output doc per array element (flatten) | `CROSS JOIN UNNEST` (explode array) |
| `$lookup` | Left-outer join another collection | `LEFT JOIN` |
| `$out` / `$merge` | Write results to a collection | `CREATE TABLE AS` / `MERGE` |
| `$count` | Count documents in the stream | `COUNT(*)` |
| `$match` after `$group` | Filter on aggregated values | `HAVING` |

```python
# Average age per city, most-populous first
pipeline = [
    {"$match": {"age": {"$gt": 25}}},                          # filter first
    {"$group": {"_id": "$city", "avg_age": {"$avg": "$age"},
                "count": {"$sum": 1}}},
    {"$sort": {"count": -1}},
]
for doc in db.users.aggregate(pipeline):
    print(doc)
```

```javascript
db.users.aggregate([
  { $match: { age: { $gt: 25 } } },
  { $group: { _id: "$city", avg_age: { $avg: "$age" }, count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

### `$lookup` + `$unwind` (join)
```python
pipeline = [
    {"$lookup": {
        "from": "posts",
        "localField": "_id",
        "foreignField": "user_id",
        "as": "user_posts"
    }},
    {"$unwind": "$user_posts"},                       # one row per post
    {"$project": {"name": 1, "user_posts.title": 1}}
]
db.users.aggregate(pipeline)
```

### Write results out
```python
pipeline = [
    {"$match": {"age": {"$gt": 30}}},
    {"$group": {"_id": "$city", "count": {"$sum": 1}}},
    {"$out": "city_report"},        # replaces the whole target collection
]
db.users.aggregate(pipeline)
```
`$out` overwrites the target collection; `$merge` (4.2+, more flexible) can insert/update/keep matching docs instead of replacing everything.

> In `mongosh`, `db.coll.aggregate(...)` returns a cursor that auto-prints the first 20 docs — **don't chain `.pretty()`** on it (it's a no-op / error on aggregation cursors; `find()` cursors already print readably).

---

## Schema Design: Embedding vs Referencing

The central modeling decision. There are no joins by default, so you choose where related data lives.

### Embedding — nest related data in one document
```javascript
db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  posts: [
    { title: "My First Post",  content: "Hello, world!" },
    { title: "My Second Post", content: "MongoDB is fun!" }
  ]
})
// read everything in a single query: db.users.findOne({ name: "Alice" })
```
**Use when:** data is read together, the "many" side is bounded, and it doesn't change independently. Fast reads, no joins.
**Risk:** unbounded growth toward the 16 MB document cap; duplicated data.

### Referencing — link collections by id
```javascript
const user = db.users.insertOne({ name: "Alice", email: "alice@example.com" })
db.posts.insertMany([
  { title: "My First Post",  content: "Hello, world!", user_id: user.insertedId },
  { title: "My Second Post", content: "MongoDB is fun!", user_id: user.insertedId }
])
// join at read time with $lookup
```
**Use when:** the related set is large or unbounded, updated independently, or shared across many parents. Costs a `$lookup` to read together.

**Rule of thumb:** *"Data that is accessed together should be stored together."* Embed by default for one-to-few read-heavy data; reference for one-to-many/many-to-many or independently mutable data.

---

## Write Concern & Read Preference

**Write concern** — how many nodes must acknowledge a write before it's considered successful (durability vs speed):

| `w` value | Meaning |
|---|---|
| `w: 1` | Primary acknowledged (can roll back if primary fails before replicating) |
| `w: "majority"` | A majority of data-bearing voting nodes acknowledged — **the default**, durable |
| `j: true` | Also require the write be flushed to the on-disk journal |

```python
from pymongo.write_concern import WriteConcern
coll = db.users.with_options(write_concern=WriteConcern(w="majority", wtimeout=5000))
coll.insert_one({"name": "Grace"})
```

**Read preference** — which replica-set members serve reads:

| Preference | Reads from |
|---|---|
| `primary` (default) | Primary only — freshest |
| `primaryPreferred` | Primary, else secondary |
| `secondary` | Secondaries only |
| `secondaryPreferred` | Secondary, else primary |
| `nearest` | Lowest latency member |

```python
from pymongo import ReadPreference
for u in db.users.with_options(read_preference=ReadPreference.SECONDARY_PREFERRED).find():
    print(u)
```

**Use:** `w: "majority"` for critical writes (payments); route heavy analytics reads to `secondaryPreferred` to spare the primary — accepting possibly-stale data.

---

## Import / Export / Backup

Command-line tools (run inside the container via `docker exec`):

| Tool | Purpose |
|---|---|
| `mongoexport` | Export one collection to JSON/CSV (human-readable) |
| `mongoimport` | Import JSON/CSV into a collection |
| `mongodump` | Binary (BSON) backup of a database/collection |
| `mongorestore` | Restore from a `mongodump` backup |

```bash
# Export a collection to JSON (file lands inside the container)
docker exec mongodb mongoexport --db=nosql_db --collection=movies \
  --out=/tmp/movies.json --username=root --password=rootpassword \
  --authenticationDatabase=admin
docker cp mongodb:/tmp/movies.json ./movies.json     # copy out to host

# Import a JSON file into a collection
docker cp ./movies.json mongodb:/tmp/movies.json
docker exec mongodb mongoimport --db=nosql_db --collection=new_movies \
  --file=/tmp/movies.json --username=root --password=rootpassword \
  --authenticationDatabase=admin

# Full backup / restore (BSON)
docker exec mongodb mongodump --db=nosql_db --out=/tmp/backup \
  --username=root --password=rootpassword --authenticationDatabase=admin
docker exec mongodb mongorestore --username=root --password=rootpassword \
  --authenticationDatabase=admin /tmp/backup
```

**export vs dump:** `mongoexport` = readable JSON/CSV for sharing/analysis (loses some BSON type fidelity); `mongodump` = exact binary backup for disaster recovery. Use dump/restore for real backups.

---

## Advanced Features

- **Sharding** — splits a collection across shards by a **shard key**. Hashed key = even writes, no range scans; ranged key = range scans but a monotonic key (timestamp/auto-id) creates a **hot shard**. Pick high-cardinality, evenly-accessed.
- **Capped collections** — fixed size, insertion-ordered; oldest overwritten when full. Good for rolling logs / recent-N buffers.
- **Time-series collections** (5.0+) — for timestamped data (IoT, metrics). Declare `timeField` + optional `metaField`; auto-bucketed for storage/query wins. In 8.0, sharding on `timeField` is deprecated — use `metaField`.
- **Transactions** (4.0+ replica sets) — ACID across documents/collections. Carry overhead; good modeling that keeps related data together avoids most need.
- **Security** — enable auth (never expose an unauthenticated `mongod`); RBAC with least privilege; TLS in transit; field-level encryption for sensitive fields.

---

## Gotchas

- **`mongo:6.0` is end-of-life** — pin `mongo:8.0` (or another supported major) for new deployments.
- **The legacy `mongo` shell is gone** — use `mongosh`. Some old snippets/`.pretty()` habits differ.
- **`_id` is immutable** — you can't change it after insert; to "change" it you delete and re-insert.
- **`count()` is deprecated** in both the shell and PyMongo 4.x — use `countDocuments()` (accurate) or `estimatedDocumentCount()` (fast metadata estimate).
- **PyMongo 4 removed legacy methods** — `insert()`, `update()`, `remove()`, `count()`, `find_and_modify()`, `map_reduce()`, `ensure_index()`. Use `insert_one`/`insert_many`, `update_one`/`update_many`, `delete_*`, `count_documents`, `create_index`.
- **`find()` returns a lazy cursor** — it doesn't hit the DB until you iterate. A cursor is consumed once; re-query to iterate again.
- **Dates are UTC** — a BSON `Date` has no timezone. Store UTC, convert on display. Use timezone-aware `datetime` in Python.
- **Floats vs `Decimal128`** — use `NumberDecimal` for money; JS/BSON doubles round (`0.1 + 0.2 ≠ 0.3`).
- **MongoDB is CP, not AP** — the default single-primary setup rejects writes rather than serve a partitioned primary. Don't assume it stays writable during a partition.
- **TTL deletion lags** — the background sweeper runs ~every 60 s, so expired docs linger briefly; don't rely on TTL for exact-time deletion.
- **Compound index order matters** — an index on `{a, b}` won't help a query filtering only on `b`.
- **`$near` isn't allowed in aggregation** — use the `$geoNear` stage (first in the pipeline) instead.
- **`$out` overwrites the whole target collection** — use `$merge` if you need to upsert into an existing collection instead of replacing it.
- **Document size cap is 16 MB** — unbounded embedded arrays eventually break inserts; reference instead when a child set can grow without limit.
- **Missing field ≠ null** — `{field: null}` also matches documents where the field is absent; use `$exists` to distinguish.
```
