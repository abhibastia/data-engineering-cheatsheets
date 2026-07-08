# Amazon QuickSight Cheatsheet

---

## Core Concepts

- QuickSight = **serverless BI / visualization** service — AWS-native alternative to Tableau/Power BI
- Build interactive dashboards, KPIs, and ad-hoc analysis; share securely
- Integrates directly with S3, Athena, Redshift, RDS
- Note: QuickSight is now part of **Amazon Quick Suite** (AWS's rebranded BI/agentic suite); the BI service and APIs are unchanged

---

## SPICE vs Direct Query

| Mode | How | When |
| --- | --- | --- |
| **SPICE** (Super-fast Parallel In-memory Calculation Engine) | Imports data into fast in-memory cache | Small–medium datasets, sub-second dashboards |
| **Direct Query** | Queries the source live via SQL | Large datasets that must stay in RDS/Redshift/Athena |

- **10 GB SPICE** included per provisioned Author; SPICE billed ~$0.38/GB/month beyond that
- Each SPICE dataset supports up to ~1 billion rows / 1 TB

---

## Data Sources

- AWS: S3, Athena, Redshift, RDS, Timestream, OpenSearch
- External: MySQL, PostgreSQL, Snowflake, Salesforce
- Files: CSV, JSON, XLSX

---

## Common Pipeline

```
S3 (data lake) → Athena (serverless SQL) → QuickSight (dashboards)
```

A staple of serverless data-engineering stacks.

---

## Build a Dashboard

1. Sign up (Enterprise = 30-day free) in the same region as your data
2. **New dataset** → S3 / Athena / upload file
3. Prepare data (rename, fix types, e.g. `Survived` → Boolean)
4. **Analysis** → drag fields to axes; pick chart (bar, pie, histogram, KPI, map)
5. **Calculated field** for derived metrics, e.g. `ifelse(Survived = 1, 1, 0)`
6. **Share → Publish dashboard** (email invite, embed, PDF/CSV export)

---

## Editions & Cost

| Edition | Notes | Price (approx, 2025) |
| --- | --- | --- |
| Standard | Basic viz; single region; IAM auth; 10 GB SPICE/author | ~$9/user/mo (annual) |
| Enterprise | + SSO, VPC, row-level security, ML insights, Amazon Q, embedding | Author ~$24, Reader ~$3/session-capped |

- **ML Insights** (Enterprise): anomaly detection, forecasting, key drivers
- 🔧 Use SPICE for speed on small data; Direct Query to avoid moving large data
- ⚠️ New AWS accounts (after 2025-07-15) use the **credit-based Free Tier**, not the old standing QuickSight free allowance — verify current pricing before relying on "free"
- Cleanup: delete datasets/dashboards; unsubscribe account to stop SPICE charges