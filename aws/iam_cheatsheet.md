# IAM Cheatsheet

---

## Core Concepts

- IAM controls authentication (who you are) and authorization (what you can do)
- Every AWS API call is checked against IAM — no bypass
- Default is **deny everything** — nothing works unless explicitly allowed
- Explicit Deny always wins over any Allow

---

## Identities

| Identity | Credentials | Use case |
| --- | --- | --- |
| User | Password + access keys (permanent) | Humans logging into console/CLI |
| Group | None (inherits member policies) | Manage permissions at team level |
| Role | Temporary STS tokens (auto-expire) | EC2, Lambda, services, cross-account |

- ⚠️ Long-lived access keys are the #1 source of AWS credential leaks — prefer roles
- ⚠️ Root user bypasses IAM — use only for billing/account setup, enable MFA
- ⚠️ `iam:PassRole` is required whenever a service (EC2, Glue) assumes a role at launch — its absence causes `AccessDenied` even when all other permissions are correct

---

## Policy Evaluation Order

1. Explicit **Deny** → always wins
2. Explicit **Allow** → granted if no deny
3. Implicit **Deny** → default if no allow

- ⚠️ Cross-account access requires Allow in **both** accounts — source IAM + destination bucket/role policy

---

## Policy Types

| Type | Attached to | Notes |
| --- | --- | --- |
| AWS Managed | Users/groups/roles | Broad; AWS maintains |
| Customer Managed | Users/groups/roles | Reusable; you maintain |
| Inline | One identity only | Deleted with identity; hard to audit at scale |
| Resource-based | S3, KMS, SQS, etc. | Cross-account; service-to-resource trust |
| Trust policy | Roles only | Defines who can assume the role |

---

## Common ARN Patterns

```
arn:aws:iam::123456789012:user/workshopuser
arn:aws:iam::123456789012:role/AirflowRole
arn:aws:s3:::my-bucket          ← bucket level (for s3:ListBucket)
arn:aws:s3:::my-bucket/*        ← object level (for s3:GetObject, s3:PutObject)
```

- ⚠️ `s3:ListBucket` must go on the bucket ARN (no `/*`); `s3:GetObject`/`s3:PutObject` must go on `/*`

---

## CLI

```bash
aws sts get-caller-identity                                          # who am I? (returns account ID, user/role ARN)
aws iam list-users                                                   # all IAM users in the account
aws iam list-groups                                                  # all groups
aws iam list-attached-group-policies --group-name G                 # managed policies attached to a group
aws iam get-group-policy --group-name G --policy-name P             # read an inline group policy document
aws iam list-roles                                                   # all roles, including AWS service roles
aws iam attach-role-policy --role-name R --policy-arn arn:...       # attach a managed policy to a role
```
