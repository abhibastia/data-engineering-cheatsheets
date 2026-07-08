# AWS DMS (Database Migration Service) Cheatsheet

---

## Core Concepts

- DMS = fully managed service to **migrate and replicate databases** with minimal downtime
- Source stays online during migration; uses **Change Data Capture (CDC)** for real-time sync
- Homogeneous (MySQL→MySQL) and heterogeneous (Oracle→PostgreSQL) migrations

---

## Components

| Component | Role |
| --- | --- |
| **Replication instance** | Managed EC2 running the DMS engine (extract → transform → load) |
| **Source endpoint** | Connection to the source DB |
| **Target endpoint** | Connection to the destination |
| **Migration task** | Defines what/how to migrate (full load, CDC, or both) |
| **CDC** | Captures ongoing INSERT/UPDATE/DELETE in near real time |

```
Source DB → DMS Replication Instance (Full Load + CDC) → Target DB
```

---

## Migration Types

| Type | Description | Use |
| --- | --- | --- |
| **Full Load** | Copy existing data once | Small/initial loads |
| **CDC** | Replicate changes after load | Continuous, near-zero downtime |
| **Full Load + CDC** | Both phases | Production migrations |

---

## Schema Conversion Tool (SCT)

- Converts schema across **different** engines (heterogeneous), e.g. Oracle → PostgreSQL
- Pair with DMS: SCT converts schema, DMS moves the data

---

## Data Engineering Relevance

```
On-prem MySQL → DMS → S3 → Glue ETL → Redshift / QuickSight
```

- Ingest operational data into data lakes (S3) or warehouses (Redshift)
- Use **CDC for incremental ingestion** instead of full refreshes
- Cross-region replication and disaster recovery

---

## Cost / Free Tier

- Priced on replication instance hours + storage + data transfer
- Use `dms.t3.micro` for small workloads; delete replication instances + RDS after labs
- Note: **DMS Serverless** auto-provisions/scales capacity (DCUs) instead of a fixed replication instance
- New accounts (after 2025-07-15) use the credit-based Free Tier, not the old 12-month model