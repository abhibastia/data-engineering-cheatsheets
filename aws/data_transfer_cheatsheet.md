# AWS Data Transfer Services Cheatsheet

---

## Core Concepts

- Multiple ways to move data in/out of AWS — pick by **volume, speed, connectivity, security**
- When the network is too slow/expensive, ship data physically (Snow family)

---

## Service Selection

| If you need to… | Use |
| --- | --- |
| Move **TB–PB** offline (network too slow) | **Snowball Edge** / **Snowcone** |
| Automate ongoing sync on-prem ↔ AWS | **DataSync** |
| Provide **FTP/SFTP/FTPS** access to S3/EFS | **Transfer Family** |

- ⚠️ **AWS Snowmobile (the 45-ft truck) was retired in 2024** — no longer offered. AWS now points exabyte-scale offline transfers at multiple **Snowball Edge** devices or **DataSync** over the network. Included below only for exam/historical awareness.

---

## Comparison

| Feature | Snowball Edge | DataSync | Transfer Family |
| --- | --- | --- | --- |
| Type | Physical device | Online service | Managed FTP/SFTP |
| Scale | TB–PB | GB–PB | MB–TB |
| Speed | Offline | 10× faster than rsync | Internet-speed |
| Connection | Shipping | Network (VPN/Direct Connect) | Internet |
| Encryption | AES-256 | TLS + AES | TLS |

---

## Snowball Edge

- Rugged appliance AWS ships to you (Storage Optimized, or Compute Optimized w/ EC2+Lambda on-device)
- Order → load via AWS OpsHub/Snowball client → ship back → auto-ingest to S3
- Use: data-center decommission, DR imports, edge collection (video/IoT)
- Note: the original standalone "Snowball" and Snowmobile are retired; the current Snow family is **Snowball Edge** and **Snowcone**

## DataSync

- Software agent (VM/EC2) → managed transfer to **S3 / EFS / FSx** (and AWS→AWS cross-region/account)
- Handles encryption, compression, validation; scheduled/incremental/bidirectional
- Use: continuous file sync, cloud backups, NAS migrations, exabyte-scale online transfers

## Transfer Family

- Managed **SFTP / FTPS / FTP** front-end for **S3 or EFS**
- Auth via IAM or Active Directory; no servers to run
- Use: migrate legacy FTP workloads, partner file exchange

---

## Security

- All transfers encrypted **in transit and at rest**; KMS keys for Snow/DataSync
- Snow devices are tamper-resistant with chain-of-custody tracking
- Transfer Family access controlled via IAM