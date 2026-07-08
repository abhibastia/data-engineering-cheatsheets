# SageMaker Visual ETL / Data Wrangler Cheatsheet

---

## Core Concepts

- SageMaker **Visual ETL Flows** = visual, Spark-based ETL inside **SageMaker Unified Studio**
- Next-gen replacement for **Data Wrangler**
- Design pipelines visually, connect to S3/Redshift/Glue/Athena, run as Processing Jobs/Pipelines
- Aimed at preparing/exporting data for **machine learning**

---

## Flow Nodes

| Node | Role |
| --- | --- |
| **Source** | Read from S3, Glue, Redshift, Athena |
| **Transform** | Filter, Join, Aggregate, Rename, Select columns |
| **Destination** | Write output to S3 (CSV/Parquet) for ML |
| **Draft flow** | Editable mode before publishing |

- **Amazon Q Developer** — built-in AI assistant to query data / generate Spark code

---

## Typical Flow

```
S3 (raw csv) → Select columns → Filter nulls → Aggregate → S3 (cleaned)
```

Example (Titanic): keep `Pclass, Sex, Age, Fare, Survived` → `Age IS NOT NULL AND Fare IS NOT NULL` → group by `Sex, Survived` → export Parquet

---

## Setup (one-time)

1. SageMaker → **Domains** → Create domain → **Quick setup** (auto-creates role + S3 bucket)
2. Wait for **Active** → **Open Studio** (Unified Studio)
3. **Build → Visual ETL Flows → Create new flow**

---

## Notes

- Runs on SageMaker's managed Spark backend
- Early version = core Spark ops (filter/join/aggregate/select); advanced Data Wrangler transforms (imputation, encoding, normalization) arriving later
- Cleanup: stop session, delete intermediate S3 data, shut down notebooks