# Amazon EFS Cheatsheet

---

## Core Concepts

- EFS (Elastic File System) = fully managed, **shared** POSIX file system for AWS
- Multiple EC2/ECS/Lambda can read/write the **same directory** simultaneously
- Serverless, auto-scales to petabytes, pay for storage used
- Acts like a shared home directory for compute nodes (ETL staging, ML preprocessing)

---

## Key Facts

| Property | Value |
| --- | --- |
| Protocol | NFS v4.1 (**port 2049**) |
| Durability | Replicated across multiple AZs |
| Performance modes | General Purpose (default, recommended), Max I/O (legacy, higher latency) |
| Throughput modes | **Bursting** (default, scales with size), **Elastic** (auto-scales, pay-per-use), **Provisioned** (fixed) |
| Access | Mount targets per subnet/AZ |

- ⚠️ vs S3: EFS = mountable file system (POSIX, shared live access); S3 = object store (API access)
- ⚠️ vs EBS: EFS = shared across many instances; EBS = attached to one instance
- **Storage classes**: Standard, **Infrequent Access (IA)**, **Archive** — lifecycle management auto-tiers cold files to cheaper classes

---

## Setup

1. **Create file system** in the same VPC; enable mount targets in **2 subnets** (different AZs)
2. **Security group**: inbound **TCP 2049** from the EC2 security group
3. Attach SG to mount targets; attach EC2 SG to instances

### Mount on EC2
```bash
sudo yum install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs <fs-id>:/ /mnt/efs
```

---

## Data Engineering Use Case

Shared staging between workers:

```
Worker 1: write CSV → /mnt/efs/incoming/
Worker 2: read CSV → convert to Parquet → upload to S3 → Athena/CTAS
```

- Multiple EC2 instances collaborate on the same files in real time
- Good for ETL scratch space, ML preprocessing, shared config/scripts

---

## Cleanup

- Unmount, terminate EC2, delete mount targets, then delete the file system