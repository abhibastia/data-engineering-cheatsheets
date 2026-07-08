# RDS & Aurora Cheatsheet

---

## Core Concepts

- RDS = managed **relational** database — AWS handles backups, patching, HA, scaling
- OLTP workload (transactions), row-based storage — **not** for analytics (use Redshift for OLAP)
- Aurora = AWS-built engine, MySQL/PostgreSQL-compatible, cloud-optimized
- You still manage: schema, queries, indexes, and (for RDS) instance sizing

---

## RDS vs Aurora

| Feature | RDS | Aurora |
| --- | --- | --- |
| Engines | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server | MySQL- & PostgreSQL-compatible only |
| Performance | Standard engine | 3–5× faster than standard |
| Storage | Fixed per instance type | Auto-scales up to **256 TiB** (raised from 128 TiB in 2025) |
| Read replicas | Limited | Up to 15, <10 ms lag |
| Global / multi-region | Manual setup | Aurora Global Database |
| Cost | Lower per instance | Higher, better performance |

- 🔧 Use **RDS** for traditional apps tied to a specific engine; **Aurora** for high-throughput/global cloud-native workloads

---

## High Availability & Scaling

| Feature | What it does |
| --- | --- |
| **Multi-AZ** | Synchronous standby in another AZ; automatic failover. HA, **not** read scaling |
| **Read Replicas** | Async copies for read scaling; can promote to standalone |
| **Automated Backups** | Daily snapshot + transaction logs → point-in-time recovery |
| **Manual Snapshots** | Retained until you delete them |

- ⚠️ Multi-AZ standby is **not** readable — it's for failover only; use read replicas to scale reads

---

## Data Engineering Relevance

- Common **source** for ingestion pipelines: RDS → DMS/Glue → S3/Redshift
- Common **target** for DMS migrations (e.g. Oracle → Aurora PostgreSQL)
- Use CDC (via DMS) for incremental ingestion instead of full table refreshes

---

## Free Tier / Cost

- Priced on instance-hours + storage + I/O + data transfer
- **Legacy 12-month Free Tier** (accounts created before **2025-07-15**): 750 hrs/month `db.t3.micro` + 20 GB
- **New Free Tier** (accounts after 2025-07-15): credit-based (up to $200 credits), not the 750-hour model — RDS usage draws down credits
- ⚠️ Always delete lab instances and snapshots after use

---

## CLI

```bash
aws rds describe-db-instances --query "DBInstances[*].{Id:DBInstanceIdentifier,Engine:Engine,Status:DBInstanceStatus}" --output table
aws rds create-db-snapshot --db-instance-identifier mydb --db-snapshot-identifier mydb-snap
aws rds describe-db-snapshots --db-instance-identifier mydb
aws rds delete-db-instance --db-instance-identifier mydb --skip-final-snapshot
```