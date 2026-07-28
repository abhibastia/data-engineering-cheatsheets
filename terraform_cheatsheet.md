# Terraform Cheatsheet

> **Infrastructure as Code (IaC):** declare infra in `.tf` files (HCL); Terraform provisions it. Lifecycle: **Write → Plan → Apply → Destroy.** Terraform is declarative — you describe the desired state, it figures out the changes.

---

## 1. Core Concepts

| Concept | What it is |
|---|---|
| **Provider** | Plugin to talk to a system (AWS, Azure, local, random, …). |
| **Resource** | One infra component (S3 bucket, VM, file, DB) that Terraform **creates & manages**. |
| **Data Source** | A **read-only** lookup of existing infra (`data "..." {}`) — fetches attributes, changes nothing. |
| **State** | Terraform's memory of what it created (`terraform.tfstate`). |
| **Variable** | Input that makes configs reusable. |
| **Output** | Value Terraform prints / exposes to other tools. |
| **Module** | A folder of reusable `.tf` code — like a function. |
| **Backend** | Where state is stored (local file or remote, e.g. S3). |
| **Workspace** | An isolated copy of state → multiple envs from one codebase. |

---

## 2. Core Workflow Commands

```bash
terraform init        # download providers, prepare modules & backend
terraform validate    # check syntax/config validity
terraform plan        # preview changes (no changes made)
terraform apply       # provision infra (prompts for 'yes')
terraform show        # human-readable view of current state
terraform destroy     # tear down all managed infra
```

```bash
terraform plan  -var="bucket_name=my-bucket-99999"    # pass a variable
terraform apply -var="bucket_name=my-bucket-99999"
terraform apply -auto-approve                         # skip confirmation (CI)
terraform init  -input=false                          # non-interactive (CI)
```

```bash
# destroy takes the SAME flags as apply
terraform destroy -var="bucket_name=my-bucket-99999"     # supply required vars
terraform destroy -auto-approve                          # skip confirmation (CI)
terraform destroy -target=aws_s3_bucket.demo             # tear down ONE resource only
```

> `-target` is for **exceptional recovery**, not routine use (HashiCorp's own guidance) — it destroys the named resource plus anything depending on it and can leave state inconsistent. To permanently drop a resource, delete its block from code and run a normal `apply`/`destroy`.

---

## 3. Providers

A provider connects Terraform to a system. Declare required providers, then configure.

### Local provider (no cloud account needed)

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

provider "local" {}

resource "local_file" "demo" {
  content  = "Hello from Terraform"
  filename = "hello.txt"
}
```

### AWS provider

```hcl
provider "aws" {
  region = "eu-central-1"
}
```

`terraform init` installs the provider; `terraform apply` creates the resources.

---

## 4. Resources & Dependencies

Each `resource` block = one infra component. A config can hold many.

```hcl
provider "aws" {
  region = "eu-central-1"
}

resource "aws_s3_bucket" "logs" {
  bucket = "iac-logs-bucket-12345"
}

resource "aws_s3_bucket" "static" {
  bucket = "iac-static-bucket-12345"
}
```

### Implicit dependencies (via references)

Terraform reads references and orders creation automatically — **no manual ordering needed**.

```hcl
resource "aws_s3_bucket" "name" {
  bucket = "iac-demo-bucket-998877"
}

resource "aws_s3_object" "readme" {
  bucket  = aws_s3_bucket.name.bucket   # reference → bucket created first
  key     = "README.txt"
  content = "Provisioned by Terraform"
}
```

Reference syntax: `<resource_type>.<name>.<attribute>` (e.g. `aws_s3_bucket.name.id`, `.arn`, `.bucket`).

---

## 5. Data Sources (read existing infra)

A `resource` **creates** infra; a `data` source **reads** something that already exists (created elsewhere, by another team, or by AWS itself). It makes no changes — Terraform just fetches attributes at plan time.

```hcl
# Who am I / which account?
data "aws_caller_identity" "current" {}

# Look up the latest Amazon Linux 2 AMI instead of hardcoding an ID
data "aws_ami" "al2" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Reference an existing bucket you did NOT create in this config
data "aws_s3_bucket" "landing" {
  bucket = "company-datalake-landing"
}

resource "aws_instance" "worker" {
  ami = data.aws_ami.al2.id                     # data.<type>.<name>.<attr>
  tags = { Account = data.aws_caller_identity.current.account_id }
}
```

Reference syntax: `data.<type>.<name>.<attribute>` (note the leading `data.`, vs. a resource's `<type>.<name>.<attribute>`).

**`data` vs `resource`:** `resource` = "make this exist and manage it"; `data` = "find this and read it, don't touch it." Common DE uses: look up an existing VPC/subnet, the account ID, the newest AMI, a Secrets Manager secret, or a bucket another stack owns.

> **`data.tf`?** Filenames are a **convention only** — Terraform merges every `.tf` in the folder regardless of name. Teams commonly split by role: `main.tf` (resources), `variables.tf`, `outputs.tf`, and `data.tf` for data-source lookups. Putting data blocks in `main.tf` is equally valid; `data.tf` just keeps read-only lookups easy to find.

---

## 6. State

State is Terraform's source of truth — links `.tf` code to real cloud resources.

**Why it matters:** mapping code↔infra, faster plans, **drift detection**, team collaboration.

```bash
terraform show               # readable state
terraform state list         # list all resources tracked in state
cat terraform.tfstate        # raw JSON
```

```
iac-state/
├─ main.tf
├─ terraform.tfstate         # auto-generated
└─ terraform.tfstate.backup  # previous state
```

### Drift detection

Drift = infra changed **outside** Terraform (e.g. someone edits a tag in the AWS Console).

```bash
terraform plan
# ~ aws_s3_bucket.demo
#       tags.Env: "prod" => "dev"    ← Terraform wants to restore code's value
```

```bash
terraform apply    # reconciles: resets drifted infra back to match code
                   # recreates the resource if it was deleted
```

Terraform always enforces the **desired state defined in code**.

---

## 7. Variables & Outputs

Variables = inputs (reusability). Outputs = surfaced values (debugging / chaining).

**`variables.tf`**

```hcl
variable "bucket_name" {
  description = "Globally unique S3 bucket name (lowercase, no spaces)"
  type        = string
}

variable "enable_versioning" {
  type    = bool
  default = false
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

**`main.tf`**

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws" }
    random = { source = "hashicorp/random" }
  }
}

provider "aws" { region = "eu-central-1" }

resource "random_id" "suffix" {
  byte_length = 4                          # avoid name collisions
}

resource "aws_s3_bucket" "demo" {
  bucket = "${var.bucket_name}-${random_id.suffix.hex}"
  tags = {
    Name = var.bucket_name
    Env  = terraform.workspace
  }
}
```

**`outputs.tf`**

```hcl
output "bucket_name" {
  description = "The actual bucket name created"
  value       = aws_s3_bucket.demo.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.demo.arn
}
```

Ways to set a variable: `-var="name=value"`, `-var-file=x.tfvars`, `terraform.tfvars` file, `TF_VAR_name` env var, or a `default`.

---

## 8. Modules

A module is a reusable folder of `.tf` code. Give it inputs (variables), it creates resources, returns outputs. Every root project is already a module.

```
project/
├─ modules/
│  └─ s3_bucket/
│     ├─ main.tf
│     ├─ variables.tf
│     └─ outputs.tf
└─ main.tf            # calls the module
```

### Calling a module

```hcl
provider "aws" { region = "eu-central-1" }

module "mybucket" {
  source      = "./modules/s3_bucket"     # local path (or Git URL / Registry)
  bucket_name = "terraform-module-bucket-12345"
  tags = {
    owner = "you"
    env   = "dev"
  }
}

output "bucket_id" {
  value = module.mybucket.bucket_id        # consume module output
}
```

**Best practices:** keep modules small & focused (one for S3, one for VPC); put inputs in `variables.tf`, results in `outputs.tf`; `source` can be a local path, Git repo, or Terraform Registry.

---

## 9. Remote Backends (team-safe state)

Local state is risky for teams: **conflicts** (concurrent applies), **lost history**, **single point of failure**. Remote backends store state centrally.

### S3 backend + native locking (Terraform 1.11+, current)

As of Terraform **1.11**, the S3 backend locks state itself using an S3 lock file (`use_lockfile = true`) — **no DynamoDB table needed**. DynamoDB locking is now **deprecated** and slated for removal. Prefer this for new setups.

Create the bucket **once** (out of band):

```bash
# S3 bucket for state  (any region except us-east-1 needs LocationConstraint)
aws s3api create-bucket --bucket iac-remote-state-bucket --region eu-central-1 \
  --create-bucket-configuration LocationConstraint=eu-central-1

# Encrypt it
aws s3api put-bucket-encryption --bucket iac-remote-state-bucket \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Enable versioning (recover a corrupted/overwritten state file)
aws s3api put-bucket-versioning --bucket iac-remote-state-bucket \
  --versioning-configuration Status=Enabled
```

Configure the backend in `main.tf`:

```hcl
terraform {
  backend "s3" {
    bucket       = "iac-remote-state-bucket"
    key          = "terraform/terraform.tfstate"   # path inside bucket
    region       = "eu-central-1"
    use_lockfile = true                             # S3-native locking (1.11+)
    encrypt      = true
  }
}
```

**Effect:** every `apply` updates one shared state file; Terraform writes a `.tflock` object in S3 (via a conditional write) so a second `apply` waits until the first finishes.

### Legacy: DynamoDB locking (pre-1.11 / migration)

Still valid and widely deployed — the course examples use it. Create a lock table and reference it:

```bash
aws dynamodb create-table --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

```hcl
terraform {
  backend "s3" {
    bucket         = "iac-remote-state-bucket"
    key            = "terraform/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "terraform-locks"    # deprecated; migrate to use_lockfile
    encrypt        = true
  }
}
```

> You can set **both** `use_lockfile` and `dynamodb_table` during migration, then drop the DynamoDB line once every teammate is on Terraform ≥ 1.11.

**Best practices:** always use remote state in teams, enable locking, encrypt (state can hold secrets), separate state per environment.

---

## 10. Workspaces (multiple environments)

A workspace = an isolated copy of state. Reuse one codebase for `dev`/`staging`/`prod` without copying folders. Default workspace is `default`.

```bash
terraform workspace list          # list all
terraform workspace new dev       # create 'dev'
terraform workspace select dev    # switch
terraform workspace show          # current name
terraform workspace delete dev    # delete (careful!)
```

Use the current name in resources via `${terraform.workspace}`:

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "myapp-${terraform.workspace}-bucket-${random_id.suffix.hex}"
  tags = {
    Env = terraform.workspace     # "dev", "prod", ...
  }
}
```

```bash
terraform workspace new dev  && terraform apply    # myapp-dev-bucket
terraform workspace new prod && terraform apply    # myapp-prod-bucket
terraform workspace select dev && terraform destroy  # deletes only dev
```

**Best practice:** workspaces for *small* env variations; for big differences (regions, accounts, providers) use **separate configs/folders**. With remote backends, each workspace gets its own state path automatically.

---

## 11. Security / Hardening Snippets

Common S3 bucket hardening — block public access, versioning, encryption:

```hcl
resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

## 12. Terraform + GitHub Actions

Standard pattern: **PR → `terraform plan`**, **merge to main → `terraform apply`** (with approval gate).

### Env config with remote backend + workspace (`env/dev/main.tf`)

```hcl
terraform {
  backend "s3" {
    bucket       = "terraform-with-actions-bucket"
    key          = "env/${terraform.workspace}/terraform.tfstate"
    region       = "eu-central-1"
    use_lockfile = true                 # S3-native locking (Terraform 1.11+)
    encrypt      = true
  }
}

provider "aws" { region = "eu-central-1" }

module "app_bucket" {
  source            = "../../modules/s3_bucket"
  bucket_name       = "myapp-${terraform.workspace}-bucket"
  enable_versioning = false
  tags = { env = terraform.workspace }
}
```

### Workflow (`.github/workflows/terraform.yml`)

```yaml
name: Terraform CI

on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  plan:
    if: github.event_name == 'pull_request'   # PR → plan
    runs-on: ubuntu-latest
    permissions:
      id-token: write                          # OIDC — no long-lived keys
      contents: read
      pull-requests: write                     # to comment the plan on the PR
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-terraform
          aws-region: eu-central-1
      - run: terraform init -input=false
        working-directory: env/dev
      - run: terraform workspace select dev || terraform workspace new dev
        working-directory: env/dev
      - run: terraform plan -no-color
        working-directory: env/dev

  apply:
    if: github.ref == 'refs/heads/main'        # merge to main → apply
    needs: plan
    runs-on: ubuntu-latest
    environment: production                     # manual approval gate
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-terraform
          aws-region: eu-central-1
      - run: terraform init -input=false
        working-directory: env/dev
      - run: terraform workspace select dev || terraform workspace new dev
        working-directory: env/dev
      - run: terraform apply -auto-approve
        working-directory: env/dev
```

**OIDC (shown above) is preferred** over `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` secrets — GitHub exchanges a short-lived token for the IAM role, so there are no static keys to leak or rotate (needs `permissions: id-token: write` + an IAM role trusting GitHub's OIDC provider). If you must use keys, store them as environment secrets. With `environment: production`, approve at **Actions → Review deployments**.

---

## 13. Quick Reference

| Task | Command / snippet |
|---|---|
| Initialize project | `terraform init` |
| Format code | `terraform fmt` (current dir) · add `-recursive` for subfolders |
| Check formatting (CI) | `terraform fmt -check` (non-zero exit if unformatted) · `-diff` to show changes |
| Preview changes | `terraform plan` |
| Apply changes | `terraform apply` |
| Apply in CI | `terraform apply -auto-approve` |
| Destroy infra | `terraform destroy` (accepts `-var`, `-auto-approve`) |
| Destroy one resource | `terraform destroy -target=aws_s3_bucket.demo` (recovery only) |
| Pass a variable | `-var="name=value"` |
| Reference an attribute | `aws_s3_bucket.demo.arn` |
| Read existing infra | `data "aws_ami" "x" { ... }` → `data.aws_ami.x.id` |
| Current workspace | `${terraform.workspace}` |
| New workspace | `terraform workspace new dev` |
| Module output | `module.mybucket.bucket_id` |
| Call a module | `source = "./modules/s3_bucket"` |
| List state resources | `terraform state list` |
| Remote state | `backend "s3" { ... }` |
| S3-native lock | `use_lockfile = true` (1.11+; replaces `dynamodb_table`) |
| Detect drift | `terraform plan` |
