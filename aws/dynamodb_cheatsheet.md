# DynamoDB Cheatsheet

---

## Core Concepts

- DynamoDB = **serverless NoSQL** key-value / document database — single-digit-ms at any scale
- Schema-less items (except the key); no joins, no servers to manage
- Design for your **access patterns first** — the opposite of relational normalization

---

## Structure

| Term | Relational analogy |
| --- | --- |
| **Table** | Table |
| **Item** | Row (a JSON object) |
| **Attribute** | Column (but per-item, not fixed) |
| **Primary key** | Uniquely identifies an item |

---

## Primary Keys

| Key type | Made of | Notes |
| --- | --- | --- |
| **Partition key** | 1 attribute | Hashes to a partition; must be unique alone |
| **Composite** | Partition key + **Sort key** | Uniqueness = pair; enables range queries within a partition |

- ⚠️ You **cannot change** partition/sort key after table creation

---

## Capacity Modes

| Mode | Use |
| --- | --- |
| **On-demand** | Unpredictable traffic, labs — pay per request, auto-scales |
| **Provisioned** | Steady traffic — set RCUs/WCUs, cheaper if predictable |

---

## Query vs Scan

| | Query | Scan |
| --- | --- | --- |
| How | By **partition key** (+ optional sort condition) | Reads **entire table**, then filters |
| Cost | Efficient | Expensive |
| Use | Known key lookups | Last resort / full exports |

- 🔧 Filters on non-key attributes are **post-filters** — they still read everything first

---

## Global Secondary Index (GSI)

- Alternate key for querying by **non-primary** attributes (e.g. query by `Country` + `Age`)
- Has its own partition/sort key + projected attributes (`ALL` / `INCLUDE` / `KEYS_ONLY`)
- Can be created after the table; while `CREATING` you can't query it yet
- ⚠️ GSIs cost extra storage + write capacity

---

## Item Format (JSON, typed)

```json
{
  "UserId":  { "S": "U1001" },
  "Email":   { "S": "alice@example.com" },
  "Age":     { "N": "29" },
  "Country": { "S": "India" }
}
```

Type codes: `S` string, `N` number, `B` binary, `BOOL`, `L` list, `M` map, `SS`/`NS` sets.

---

## boto3 Access

```python
import boto3
table = boto3.resource("dynamodb", region_name="us-east-1").Table("UsersTable")

table.put_item(Item={"UserId": "U1001", "Email": "a@x.com", "Age": 29})
table.get_item(Key={"UserId": "U1001", "Email": "a@x.com"})
table.update_item(Key={"UserId": "U1002", "Email": "b@x.com"},
                  UpdateExpression="SET Age = :a", ExpressionAttributeValues={":a": 30})
table.delete_item(Key={"UserId": "U1001", "Email": "a@x.com"})
```

---

## Data Modeling Principles

1. Design for access patterns first
2. Composite keys model one-to-many relationships
3. Store related data together (avoid joins)
4. Use GSIs for alternate query paths
5. Keep items lean — avoid huge attributes

---

## Extras

- **DynamoDB Streams** — change log; trigger Lambda on insert/update/delete
- **TTL** — auto-expire items by timestamp attribute
- Monitor: read/write throughput, **throttled requests** in the table Monitor tab