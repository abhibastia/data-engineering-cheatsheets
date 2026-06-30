# S3 Cheatsheet

---

## Core Concepts

- Bucket names are **globally unique** across all AWS accounts
- S3 is **object storage**, not block or file storage — no random writes, no partial updates
- Keys are flat strings — `folder/file.csv` is not a real folder, just a prefix in the key name
- Objects can be up to **5 TB**; single PUT limit is **5 GB** (use multipart above 100 MB)
- S3 is **region-scoped** — data does not leave the region unless you configure replication
- S3 is **strongly consistent** (since Dec 2020) — reads after PUT, DELETE, or overwrite return the latest version immediately

---

## Storage Classes

| Class | Retrieval | Min Storage | Key Signal |
| --- | --- | --- | --- |
| Standard | ms | None | Default, active data |
| Intelligent-Tiering | ms | None | Unknown access pattern |
| Standard-IA | ms | 30 days | Monthly access |
| One Zone-IA | ms | 30 days | Reproducible data only |
| Glacier Instant | ms | 90 days | Archive + rare instant access |
| Glacier Flexible | min–hours | 90 days | Archive, delay OK |
| Glacier Deep Archive | up to 12h | 180 days | Compliance, years |

- ⚠️ You pay minimum storage duration even if you delete early
- ⚠️ One Zone-IA has no cross-AZ redundancy — data loss possible if AZ fails

---

## Security

- **Bucket policy** — controls who accesses the bucket; attached to the bucket; can include AWS services and other accounts as principals
- **IAM policy** — controls what an identity can do; attached to user/role/group
- **ACLs** — legacy; disable with S3 Object Ownership = Bucket Owner Enforced
- ⚠️ Cross-account access requires **both** the bucket policy (in account A) AND the IAM policy (in account B) to allow — one alone is not enough
- ⚠️ CloudTrail writing to S3 requires a **bucket policy** granting `cloudtrail.amazonaws.com` permission — your IAM policy is irrelevant for this
- **Block Public Access** is a safety override at the account or bucket level — it overrides any bucket policy that would make objects public
- ⚠️ Presigned URLs inherit the **creator's credentials** — if the signing role is revoked, the URL stops working immediately regardless of expiry time

---

## Encryption

- **SSE-S3** — AWS manages keys; AES-256; enabled by default since Jan 2023
- **SSE-KMS** — you control key policy in KMS; adds CloudTrail audit on key usage; requires `kms:GenerateDataKey` permission on the IAM role
- **SSE-C** — you provide the key with every request; AWS never stores it; console access not possible
- **Client-side** — encrypted before upload; AWS cannot read content
- ⚠️ SSE-KMS `AccessDenied` errors on upload usually mean missing `kms:GenerateDataKey` permission, not an S3 problem

---

## Versioning & Lifecycle

- Versioning cannot be **turned off**, only suspended
- Deleting a versioned object adds a **delete marker** — the data is not gone
- ⚠️ Always pair versioning with a lifecycle rule to expire noncurrent versions, or storage cost compounds
- Lifecycle transitions for objects **< 128 KB** cost more than they save — skip small files
- `NoncurrentVersionExpiration` must be configured separately when versioning is on

---

## Replication

- Versioning must be enabled on **both** source and destination
- Replication applies to **new objects only** — existing objects are not copied
- ⚠️ Delete markers are **not replicated by default** — enable explicitly
- CRR = cross-region (DR, compliance, latency); SRR = same-region (log aggregation, dev/prod sync)

---

## Performance

- 3,500 write / 5,500 read requests per second **per prefix**
- 🔧 Distribute writes across multiple prefixes for high-throughput pipelines
- Multipart upload: required >5 GB, recommended >100 MB; abort incomplete uploads via lifecycle rule
- Transfer Acceleration: CloudFront edge → AWS backbone; adds cost; only use for distant clients
- Byte-range fetches: download specific bytes in parallel; used internally by Athena/Spark on Parquet

---

## Athena + Data Lake

- Athena charges **per TB scanned** — format and partitioning directly control cost
- Parquet/ORC reduces scanned data vs CSV — columnar, compressed, Athena reads only needed columns
- Partition keys in key path: `year=2024/month=06/` → Athena prunes skipped partitions
- ⚠️ `SELECT *` on a large unpartitioned CSV is expensive — always filter on partitions
- 🔧 Standard layout: `raw/` → `processed/` → `curated/` with Parquet in processed/curated
- S3 Select: SQL on a single object inside Lambda without downloading the full file

---

## CLI

```bash
aws s3 ls                                          # list buckets
aws s3 ls s3://bucket/ --recursive --human-readable # list objects with sizes
aws s3 mb s3://bucket --region eu-central-1        # create bucket
aws s3 cp file.csv s3://bucket/prefix/             # upload
aws s3 cp s3://bucket/prefix/file.csv .            # download
aws s3 sync ./local/ s3://bucket/prefix/           # sync local → S3
aws s3 sync s3://bucket/prefix/ ./local/           # sync S3 → local
aws s3 sync ./local/ s3://bucket/prefix/ --delete  # mirror: removes destination files absent from source
aws s3 rm s3://bucket/prefix/file.csv              # delete object
aws s3 rm s3://bucket/prefix/ --recursive          # delete prefix
aws s3 presign s3://bucket/key --expires-in 3600   # presigned URL (1hr)
aws s3 cp file.csv s3://bucket/ --storage-class STANDARD_IA  # upload to IA
```
