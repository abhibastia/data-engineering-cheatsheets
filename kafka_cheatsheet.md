# Apache Kafka Cheatsheet

Kafka is a distributed, append-only **commit log** for streaming events. Producers write records to **topics** (split into **partitions**); consumers read them at their own pace. Records are retained by time/size, so multiple consumers can replay independently.

> **Version baseline:** targets **Kafka 4.x** (released March 2025). Kafka 4.0 runs **KRaft only** — ZooKeeper is fully removed, so there is no `zookeeper` service to run anymore. CLI tools ship as `kafka-topics.sh` in the Apache distribution; Confluent Docker images drop the `.sh` (`kafka-topics`). Examples use those bare names.

## Contents

1. [Core Concepts](#core-concepts)
2. [Architecture](#architecture)
3. [Setup with Docker (KRaft)](#setup-with-docker-kraft)
4. [Topic CLI](#topic-cli)
5. [Console Producer / Consumer](#console-producer--consumer)
6. [Consumer Groups & Offsets](#consumer-groups--offsets)
7. [Python — Producer & Consumer](#python--producer--consumer)
8. [Delivery Guarantees](#delivery-guarantees)
9. [Key Configs](#key-configs)
10. [Retention & Cleanup](#retention--cleanup)
11. [Kafka Connect](#kafka-connect)
12. [ksqlDB](#ksqldb)
13. [Gotchas](#gotchas)

---

## Core Concepts

| Term | What it is |
|---|---|
| **Broker** | A Kafka server storing data and serving clients. A **cluster** = many brokers. |
| **Topic** | Named category of messages (≈ a log file). Producers write, consumers read. |
| **Partition** | A topic is split into ordered partitions — the unit of parallelism & ordering. |
| **Offset** | Sequential id of a record within a partition. Consumers track their position by offset. |
| **Producer** | Client that publishes records to a topic. |
| **Consumer** | Client that reads records from a topic. |
| **Consumer group** | Consumers sharing a `group.id`; each partition goes to exactly one member — scales reads. |
| **Replication** | Each partition has 1 leader + N-1 followers on other brokers for fault tolerance. |
| **ISR** | In-Sync Replicas — followers caught up with the leader; eligible to become leader. |
| **Controller** | KRaft broker(s) managing cluster metadata (replaces ZooKeeper). |
| **Retention** | How long records are kept (by time or size) before deletion — not by consumption. |

---

## Architecture

```text
Producers ──▶ [ Topic: orders ]                    Consumer Group A
              ├─ Partition 0 (leader b1, replicas b2,b3) ──▶ consumer 1
              ├─ Partition 1 (leader b2, replicas b1,b3) ──▶ consumer 2
              └─ Partition 2 (leader b3, replicas b1,b2) ──▶ consumer 1
```

- **Ordering** is guaranteed **within a partition only**, never across a topic.
- Records with the **same key** always land in the same partition (key hash) → per-key ordering.
- Partition count sets the **max parallelism**: more consumers than partitions ⇒ some sit idle.

---

## Setup with Docker (KRaft)

```yaml
# docker-compose.yml — single-broker KRaft, no ZooKeeper
services:
  kafka:
    image: apache/kafka:latest       # official image, KRaft by default
    container_name: kafka
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

```bash
docker compose up -d
docker exec -it kafka bash        # get a shell to run CLI tools
```

---

## Topic CLI

```bash
# Create — partitions & replication-factor are fixed at creation (RF cannot exceed broker count)
kafka-topics --create --topic orders --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1

kafka-topics --list --bootstrap-server localhost:9092
kafka-topics --describe --topic orders --bootstrap-server localhost:9092

# Add partitions (can only increase, never decrease — and it breaks key ordering)
kafka-topics --alter --topic orders --bootstrap-server localhost:9092 --partitions 6

kafka-topics --delete --topic orders --bootstrap-server localhost:9092
```

---

## Console Producer / Consumer

```bash
# Produce (type messages, Ctrl+C to stop). key:value needs the two parse flags
kafka-console-producer --topic orders --bootstrap-server localhost:9092 \
  --property parse.key=true --property key.separator=:

# Consume from the beginning, showing keys
kafka-console-consumer --topic orders --bootstrap-server localhost:9092 \
  --from-beginning --property print.key=true
```

---

## Consumer Groups & Offsets

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --list

# LAG = how far behind a group is (log-end-offset − current-offset). Key health metric.
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group my-group

# Reset offsets (group must be inactive). --to-earliest | --to-latest | --to-datetime ...
kafka-consumer-groups --bootstrap-server localhost:9092 --group my-group \
  --topic orders --reset-offsets --to-earliest --execute
```

- **Offsets are committed per group** and stored in the internal `__consumer_offsets` topic.
- A new group with `auto.offset.reset=earliest` replays all retained data; `latest` reads only new records.

---

## Python — Producer & Consumer

Two clients: **`confluent-kafka`** (librdkafka-backed, fastest, officially maintained — prefer for production) and **`kafka-python`** (pure Python, simple). `confluent-kafka` shown first.

```python
# pip install confluent-kafka
import json
from confluent_kafka import Producer

p = Producer({"bootstrap.servers": "localhost:9092"})

def ack(err, msg):
    print("failed:", err) if err else print("delivered:", msg.topic(), msg.partition())

# value/key must be str or bytes — serialize dicts to JSON yourself
p.produce("orders", key="order123", value=json.dumps({"status": "placed"}), callback=ack)
p.flush()                       # block until all messages are delivered
```

```python
from confluent_kafka import Consumer

c = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "my-group",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,        # commit manually for at-least-once control
})
c.subscribe(["orders"])

try:
    while True:
        msg = c.poll(1.0)               # seconds
        if msg is None:
            continue
        if msg.error():
            print("error:", msg.error()); continue
        print(msg.key(), msg.value())
        c.commit(msg)                   # commit AFTER processing
finally:
    c.close()
```

```python
# kafka-python equivalent (pip install kafka-python)
from kafka import KafkaProducer, KafkaConsumer
KafkaProducer(bootstrap_servers="localhost:9092").send("orders", key=b"k", value=b"v")
KafkaConsumer("orders", bootstrap_servers="localhost:9092", auto_offset_reset="earliest")
```

---

## Delivery Guarantees

| Guarantee | Meaning | How |
|---|---|---|
| **At-most-once** | May lose, never duplicate | Commit offset *before* processing; `acks=0/1` |
| **At-least-once** | Never lose, may duplicate | Commit offset *after* processing (default mindset) |
| **Exactly-once** (EOS) | No loss, no duplicates | Idempotent producer + transactions + `read_committed` |

```python
# Exactly-once-ish producer settings (confluent-kafka)
Producer({
    "bootstrap.servers": "localhost:9092",
    "enable.idempotence": True,    # default true since Kafka 3.0 — dedupes retries
    "acks": "all",                 # implied by idempotence; wait for all ISR
})
```

---

## Key Configs

**Producer**

| Config | Effect |
|---|---|
| `acks` | `0` fire-and-forget · `1` leader only · `all` all ISR (durable, default with idempotence) |
| `enable.idempotence` | Dedupe retried sends (default `true` since 3.0) |
| `linger.ms` / `batch.size` | Wait to batch records → higher throughput, slight latency |
| `compression.type` | `none`/`gzip`/`snappy`/`lz4`/`zstd` — shrinks network & disk |

**Consumer**

| Config | Effect |
|---|---|
| `group.id` | Consumer group name — required for coordinated consumption |
| `auto.offset.reset` | `earliest` / `latest` when no committed offset exists |
| `enable.auto.commit` | Auto-commit offsets every `auto.commit.interval.ms` (risk: commit before processing) |
| `max.poll.records` | Max records per `poll()` |
| `group.protocol` | `consumer` opts into the KIP-848 rebalance protocol (GA in 4.0, still opt-in); `classic` is the old default |

**Broker / topic durability**

| Config | Effect |
|---|---|
| `replication.factor` | Copies per partition (3 typical in prod) |
| `min.insync.replicas` | With `acks=all`, min replicas that must ack or the write fails |

---

## Retention & Cleanup

```bash
# See a topic's effective (non-default) configs
kafka-configs --bootstrap-server localhost:9092 --describe --topic orders

# Time-based: keep 7 days
kafka-configs --bootstrap-server localhost:9092 --alter --topic orders \
  --add-config retention.ms=604800000

# Log compaction: keep only the latest value per key (great for changelogs/state)
kafka-configs --bootstrap-server localhost:9092 --alter --topic orders \
  --add-config cleanup.policy=compact

# Revert a config back to its default
kafka-configs --bootstrap-server localhost:9092 --alter --topic orders \
  --delete-config retention.ms
```

- `cleanup.policy=delete` (default) drops old segments by `retention.ms` / `retention.bytes`.
- `cleanup.policy=compact` retains the last record per key forever — used for state/changelog topics.

---

## Kafka Connect

Framework to move data in/out of Kafka with **no code** — configure connectors via REST (default port `8083`).

- **Source connector** — pulls external data *into* Kafka (DB → topic).
- **Sink connector** — pushes topic data *out* to an external system (topic → MongoDB/S3).

```bash
# List and create connectors via the REST API
curl http://localhost:8083/connectors

curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
  "name": "mongo-sink",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
    "topics": "orders",
    "connection.uri": "mongodb://root:rootpassword@mongodb:27017",
    "database": "nosql_db",
    "collection": "orders"
  }
}'

curl http://localhost:8083/connectors/mongo-sink/status
```

---

## ksqlDB

SQL over Kafka streams — filter, transform, join, aggregate continuously.

- **Stream** — append-only sequence of events (facts). Ordered, immutable.
- **Table** — latest value per key (evolving state), built by aggregating a stream.

```sql
-- Stream over a topic
CREATE STREAM orders (id STRING, amount INT)
  WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='JSON');

-- Continuous filter (EMIT CHANGES = push query, runs forever)
SELECT * FROM orders WHERE amount > 100 EMIT CHANGES;

-- Aggregate into a table
CREATE TABLE order_counts AS
  SELECT id, COUNT(*) AS cnt FROM orders GROUP BY id EMIT CHANGES;

-- Tumbling window: fixed, non-overlapping time buckets
CREATE TABLE per_minute AS
  SELECT id, COUNT(*) AS cnt FROM orders
  WINDOW TUMBLING (SIZE 1 MINUTE) GROUP BY id EMIT CHANGES;
```

`PRINT 'orders' FROM BEGINNING;` dumps raw topic contents · `SHOW STREAMS;` / `SHOW TABLES;` list them.

---

## Gotchas

- **ZooKeeper is gone in 4.0** — KRaft only. Old `--zookeeper` flags and `zookeeper` compose services are dead; use `--bootstrap-server`. No direct upgrade from ZK mode → migrate to KRaft on 3.9 first.
- **Ordering is per-partition, not per-topic** — need global order? Use one partition (kills parallelism).
- **Partitions only increase** — you can't shrink; and increasing them **remaps key→partition**, breaking existing per-key ordering.
- **Replication factor is capped by broker count** — RF=3 needs ≥3 brokers, else topic creation fails.
- **`acks=all` alone isn't enough** — pair with `min.insync.replicas` ≥2, or a single live replica still accepts writes you can lose.
- **Auto-commit can lose messages** — it commits on a timer regardless of processing success; commit manually after processing for at-least-once.
- **Retention ≠ consumption** — records expire on schedule even if unread; a slow consumer with lag past retention **misses data permanently**.
- **`kafka-python` lags on newer features** — prefer `confluent-kafka` for KIP-848 consumer protocol, transactions, and performance.
- **Consumer count > partitions** — extra consumers in a group idle; scale partitions to scale consumers.
- **Rebalances pause consumption** — adding/removing group members triggers a rebalance. The KIP-848 protocol (GA in 4.0) makes these incremental and far less disruptive, but you must opt in with `group.protocol=consumer`; it isn't the default yet.
