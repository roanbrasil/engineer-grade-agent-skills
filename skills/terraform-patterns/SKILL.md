---
name: terraform-patterns
description: Write production-grade Terraform — module design, remote state, workspace vs directory structure, variable management, drift detection, Atlantis GitOps, tagging, and anti-patterns
---

# Production Terraform Patterns — Expert Reference

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                Terraform Workflow — GitOps with Atlantis              │
│                                                                        │
│  Engineer        GitHub PR         Atlantis        AWS / GCP / Azure  │
│  ┌────────┐      ┌─────────┐      ┌──────────┐    ┌───────────────┐  │
│  │ writes │─────►│ PR open │─────►│ plan     │───►│  Preview      │  │
│  │ .tf    │      │         │      │ comment  │    │  (no changes) │  │
│  │ files  │      │         │      │          │    │               │  │
│  │        │      │ PR merge│─────►│ apply    │───►│  Real changes │  │
│  └────────┘      └─────────┘      └──────────┘    └───────────────┘  │
│                                                                        │
│  State stored remotely (S3 + DynamoDB lock / GCS / Terraform Cloud)  │
│  Every plan and apply has a PR audit trail                            │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Terraform Fundamentals — Quick Reference

```hcl
# Providers declare which APIs to talk to
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"   # pessimistic constraint: >=5.40, <6.0
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.27"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Data sources — read existing infra (not managed by this config)
data "aws_vpc" "main" {
  id = var.vpc_id
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
  }
}

# Resources — the infrastructure you manage
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = data.aws_vpc.main.id

  tags = local.common_tags
}

# Locals — computed values, avoid repetition
locals {
  name_prefix = "${var.environment}-${var.project}"
  common_tags = {
    Name        = "${local.name_prefix}-web"
    Environment = var.environment
    Team        = var.team
    CostCenter  = var.cost_center
    ManagedBy   = "terraform"
  }
}

# Outputs — expose values to callers or other modules
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}
```

---

## Module Design

### Directory Structure

```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf          # required_providers pinned here
├── eks-cluster/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
└── rds-postgres/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── versions.tf

environments/
├── dev/
│   ├── main.tf              # calls modules
│   ├── variables.tf
│   ├── terraform.tfvars     # dev-specific values
│   └── backend.tf           # remote state config
├── staging/
│   ├── main.tf
│   ├── terraform.tfvars
│   └── backend.tf
└── prod/
    ├── main.tf
    ├── terraform.tfvars
    └── backend.tf
```

### Single Responsibility — One Module Does One Thing

```hcl
# GOOD: modules/rds-postgres/main.tf — only RDS, nothing else
resource "aws_db_subnet_group" "this" {
  name       = "${var.identifier}-subnet-group"
  subnet_ids = var.subnet_ids
  tags       = var.tags
}

resource "aws_db_instance" "this" {
  identifier        = var.identifier
  engine            = "postgres"
  engine_version    = var.engine_version   # "16.2"
  instance_class    = var.instance_class
  allocated_storage = var.allocated_storage
  db_name           = var.database_name
  username          = var.master_username
  password          = var.master_password
  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = var.backup_retention_days
  skip_final_snapshot     = var.environment != "prod"
  deletion_protection     = var.environment == "prod"

  tags = var.tags
}

# BAD: "platform" module that creates VPC + EKS + RDS + S3 + IAM + CloudFront
# - Hard to test; change to EKS also plans VPC changes
# - Cannot reuse individual pieces
# - Blast radius: one bad plan touches everything
```

### Input Variables — Typed and Validated

```hcl
# modules/rds-postgres/variables.tf

variable "identifier" {
  description = "Unique identifier for the RDS instance (e.g., 'orders-prod')"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{0,62}$", var.identifier))
    error_message = "Identifier must be lowercase alphanumeric with hyphens, max 63 chars."
  }
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t4g.medium"

  validation {
    condition     = startswith(var.instance_class, "db.")
    error_message = "instance_class must start with 'db.' (e.g., 'db.t4g.medium')."
  }
}

variable "engine_version" {
  description = "PostgreSQL version to use"
  type        = string
  default     = "16.2"
}

variable "environment" {
  description = "Deployment environment: dev, staging, prod"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "master_password" {
  description = "Master DB password — provide via secrets manager, not tfvars"
  type        = string
  sensitive   = true   # suppresses output in plans; value never logged
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

### Outputs — Expose Only What Callers Need

```hcl
# modules/rds-postgres/outputs.tf

output "endpoint" {
  description = "RDS endpoint (host:port)"
  value       = aws_db_instance.this.endpoint
}

output "port" {
  description = "RDS port"
  value       = aws_db_instance.this.port
}

output "database_name" {
  description = "Database name"
  value       = aws_db_instance.this.db_name
}

# NEVER expose secrets in outputs — they appear in state and plan output
# BAD:
# output "master_password" { value = var.master_password }  # secret in state!

# If you must reference a secret downstream, use a data source at point of use
```

### Calling Modules

```hcl
# environments/prod/main.tf
module "vpc" {
  source  = "../../modules/vpc"

  cidr_block       = "10.0.0.0/16"
  azs              = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets   = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  environment      = var.environment
  tags             = local.common_tags
}

module "orders_db" {
  source  = "../../modules/rds-postgres"

  identifier      = "orders-${var.environment}"
  instance_class  = "db.r6g.xlarge"
  engine_version  = "16.2"
  environment     = var.environment
  subnet_ids      = module.vpc.private_subnet_ids
  master_username = "orders_admin"
  master_password = data.aws_secretsmanager_secret_version.db_password.secret_string
  tags            = local.common_tags
}

# Cross-module references: use outputs from one module as inputs to another
module "eks" {
  source  = "../../modules/eks-cluster"
  vpc_id  = module.vpc.vpc_id          # reference vpc module output
  subnet_ids = module.vpc.private_subnet_ids
  # ...
}
```

---

## Remote State

### S3 + DynamoDB (AWS)

```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/orders-service/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                      # AES-256 server-side encryption
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/..."  # CMK
    dynamodb_table = "terraform-state-locks"   # prevents concurrent apply
  }
}
```

```hcl
# One-time setup: create the S3 bucket + DynamoDB table
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.state.arn
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### State Isolation Rules

```
RULE: One state file per environment per service, never shared state.

environments/dev/main.tf    → key: "dev/orders-service/terraform.tfstate"
environments/staging/main.tf → key: "staging/orders-service/terraform.tfstate"
environments/prod/main.tf   → key: "prod/orders-service/terraform.tfstate"

Why isolation matters:
  - A botched plan in dev cannot corrupt prod state
  - Blast radius is bounded to one environment
  - State files can have different IAM permissions (prod = restricted)
  - Terraform refresh in dev doesn't slow down prod operations

NEVER: key: "all/terraform.tfstate"  # one giant state = shared fate
```

### Cross-State References

```hcl
# Reference outputs from another state file
# Use sparingly — creates coupling between state files

data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "${var.environment}/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Then use:
subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids

# PREFERRED alternative: pass values via variables instead
# Decoupled: networking team publishes VPC ID via SSM Parameter Store
data "aws_ssm_parameter" "vpc_id" {
  name = "/${var.environment}/networking/vpc-id"
}
```

---

## Variable Management

### .tfvars Per Environment

```hcl
# environments/dev/terraform.tfvars
environment    = "dev"
aws_region     = "us-east-1"
instance_class = "db.t4g.medium"    # smaller in dev
min_capacity   = 1
max_capacity   = 2
team           = "platform"
cost_center    = "engineering"

# environments/prod/terraform.tfvars
environment    = "prod"
aws_region     = "us-east-1"
instance_class = "db.r6g.xlarge"   # larger in prod
min_capacity   = 3
max_capacity   = 10
team           = "platform"
cost_center    = "engineering"
```

```bash
# Apply with explicit tfvars
terraform -chdir=environments/prod apply -var-file=terraform.tfvars

# NEVER commit these files:
# .gitignore
*.tfstate
*.tfstate.backup
.terraform/
*.tfvars           # may contain non-sensitive env values; acceptable to commit
*.tfvars.json      # ditto
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

### Sensitive Variables — Never in tfvars

```bash
# Inject secrets via environment variables (CI/CD)
export TF_VAR_master_password="$(aws secretsmanager get-secret-value \
  --secret-id prod/orders-db/password \
  --query SecretString --output text)"

terraform apply -var-file=terraform.tfvars
# master_password comes from env var, not tfvars file

# GitHub Actions example
- name: Apply Terraform
  env:
    TF_VAR_master_password: ${{ secrets.DB_MASTER_PASSWORD }}
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  run: terraform -chdir=environments/prod apply -auto-approve -var-file=terraform.tfvars
```

---

## Workspace vs Directory Structure

```
Workspaces:
  terraform workspace new dev
  terraform workspace new prod
  
  Single directory, multiple state files (one per workspace)
  State key: "terraform.tfstate.d/prod/terraform.tfstate"
  
  PROS: simpler for ephemeral environments (PR previews)
  CONS: same .tf files for all envs; can't have per-env module versions;
        state isolation weaker; easy to apply to wrong workspace

Directory per environment (PREFERRED for production):
  environments/dev/   → separate .tf files, separate backend config
  environments/prod/  → separate .tf files, separate backend config
  
  PROS: true isolation; per-env variable files; can use different module versions;
        IAM policies per-env; impossible to accidentally apply dev plan to prod
  CONS: more files; some duplication in main.tf files

Rule: use workspaces for ephemeral per-PR environments;
      use directories for long-lived dev / staging / prod.
```

---

## Drift Detection

```bash
# Detect if real infra diverged from Terraform config
# Exit codes: 0 = no changes, 1 = error, 2 = changes detected (drift)
terraform plan -detailed-exitcode -var-file=terraform.tfvars
echo $?   # 2 if drifted

# CI job: scheduled drift detection
#!/bin/bash
set -e
cd environments/prod
terraform init -backend-config=backend.tfvars
terraform plan -detailed-exitcode -var-file=terraform.tfvars -out=drift.tfplan
EXIT_CODE=$?
if [ $EXIT_CODE -eq 2 ]; then
  echo "DRIFT DETECTED in prod"
  # Post to Slack / PagerDuty / create Jira ticket
  curl -X POST $SLACK_WEBHOOK_URL \
    -d '{"text":"⚠️ Terraform drift detected in prod. Review: '"$PLAN_URL"'"}'
fi
```

```yaml
# GitHub Actions scheduled drift check
name: Drift Detection
on:
  schedule:
    - cron: '0 8 * * *'   # daily at 8am
  workflow_dispatch:

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - name: Check drift
        run: |
          cd environments/prod
          terraform init
          terraform plan -detailed-exitcode || {
            if [ $? -eq 2 ]; then
              echo "Drift detected"
              exit 2
            fi
          }
```

---

## Import — Adopt Existing Resources

```bash
# Workflow: bring manually created resources under Terraform management
# Step 1: Write the Terraform resource config (must match real resource)
# Step 2: Import the real resource into state
# Step 3: terraform plan shows no diff → done

# Import existing RDS instance
terraform import \
  -var-file=terraform.tfvars \
  module.orders_db.aws_db_instance.this \
  orders-prod-db-identifier

# Verify: plan should show no changes
terraform plan -var-file=terraform.tfvars
# Expected: "No changes. Your infrastructure matches the configuration."

# Terraform 1.5+: import block (declarative, code-reviewable)
import {
  to = module.orders_db.aws_db_instance.this
  id = "orders-prod-db-identifier"
}
```

---

## Atlantis — GitOps for Terraform

```yaml
# atlantis.yaml — project configuration
version: 3
projects:
  - name: orders-service-dev
    dir: environments/dev
    workspace: default
    terraform_version: v1.7.5
    autoplan:
      when_modified: ["**/*.tf", "../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - mergeable    # PR must be mergeable
      - approved     # at least one approval

  - name: orders-service-prod
    dir: environments/prod
    workspace: default
    terraform_version: v1.7.5
    autoplan:
      when_modified: ["**/*.tf"]
      enabled: true
    apply_requirements:
      - mergeable
      - approved     # require 2 approvals for prod
```

```
Atlantis Workflow:
  1. Engineer opens PR with .tf changes
  2. Atlantis auto-runs: terraform plan
  3. Plan output posted as PR comment (reviewable by peers)
  4. After code review + approval: commenter posts "atlantis apply"
  5. Atlantis runs apply; result posted as PR comment
  6. PR merged (apply already done; merge = audit record)

Benefits:
  - Plan visible in PR before merge (team reviews infra changes)
  - Full audit trail: every apply linked to a PR
  - No shared developer AWS credentials (Atlantis has them)
  - apply_requirements enforce approval policy per environment
```

---

## Tagging Strategy

```hcl
# locals.tf — define tags once, apply everywhere
locals {
  common_tags = {
    Name               = "${var.project}-${var.environment}"
    Environment        = var.environment           # dev / staging / prod
    Project            = var.project               # orders-service
    Team               = var.team                  # platform-engineering
    CostCenter         = var.cost_center           # CC-1234
    ManagedBy          = "terraform"               # NOT manually managed
    TerraformWorkspace = terraform.workspace
    Repository         = "github.com/myorg/infra"  # find the code
  }
}

# Apply to every resource via tags argument
resource "aws_db_instance" "this" {
  # ... config ...
  tags = merge(local.common_tags, {
    Name = "${local.common_tags["Name"]}-rds"
  })
}

resource "aws_eks_cluster" "this" {
  # ...
  tags = local.common_tags
}
```

```
Required tags (enforce via AWS Config Rule or OPA policy):
  Name        — human-readable resource name
  Environment — enables per-env cost filtering
  Team        — who to contact for this resource
  CostCenter  — cost allocation for billing
  ManagedBy   — "terraform" (proves it's not a rogue manual resource)

Why ManagedBy=terraform matters:
  AWS Cost Explorer: filter by tag → see cost of managed vs. unmanaged infra
  Drift detection: untagged resource = likely manually created outside Terraform
  Security audit: "show me all resources NOT tagged ManagedBy=terraform"
```

---

## Production Checklist

```
Module design:
  [ ] Single responsibility — module does one thing
  [ ] All variables typed + validated + described
  [ ] No secrets in outputs (sensitive=true on input, not re-exposed)
  [ ] required_providers pinned in versions.tf per module
  [ ] Module tested in isolation before production use

State management:
  [ ] Remote state backend configured (S3 + DynamoDB, or Terraform Cloud)
  [ ] State encryption enabled (S3 SSE with CMK)
  [ ] State locking enabled (DynamoDB table)
  [ ] Separate state per environment (key includes env name)
  [ ] .tfstate never committed to git (.gitignore includes *.tfstate)
  [ ] State backed up / versioned (S3 versioning enabled)

Variables:
  [ ] Environment-specific values in .tfvars, not hardcoded in .tf
  [ ] Secrets injected via TF_VAR_ env vars or Vault, not .tfvars
  [ ] sensitive=true on all password/key variables
  [ ] Validation blocks on critical variables (env name, CIDR blocks)

Operations:
  [ ] terraform plan reviewed before every apply (never apply blindly)
  [ ] Drift detection scheduled (daily CI job)
  [ ] Atlantis (or equivalent) enforces plan-on-PR, apply-on-merge
  [ ] All resources tagged with Name, Environment, Team, CostCenter, ManagedBy=terraform
  [ ] Import used for existing resources (none created manually then ignored)
```

---

## Anti-Patterns

```
WRONG: count for environment branching
  resource "aws_instance" "web" {
    count         = var.environment == "prod" ? 2 : 1
    instance_type = var.environment == "prod" ? "m5.xlarge" : "t3.micro"
  }
  // Plan output confusing; removing an instance renumbers the others
RIGHT: separate modules with separate tfvars per environment.

WRONG: Hardcoded region and account ID
  provider "aws" { region = "us-east-1" }
  arn = "arn:aws:iam::123456789:role/my-role"
  // Cannot deploy to another region or account without search-replace
RIGHT: var.aws_region, data.aws_caller_identity.current.account_id

WRONG: No pin on provider version
  required_providers {
    aws = { source = "hashicorp/aws" }  # no version = takes latest
  }
  // AWS provider 5.0 broke 3.x configs; CI was green, then suddenly not
RIGHT: version = "~> 5.40"  — always pessimistic constraint.

WRONG: Storing secrets in state
  resource "aws_db_instance" "db" {
    password = "super-secret-password"  # stored in tfstate in plaintext
  }
RIGHT: generate password via random_password resource + store in Secrets Manager;
       reference SecretString at runtime via data source, not as Terraform attribute.

WRONG: Applying without reviewing plan
  terraform apply -auto-approve  # in production
  // "I thought it would only change the tag; it replaced the DB"
RIGHT: terraform plan -out=tfplan, then human reviews, then terraform apply tfplan.

WRONG: Giant single state file
  environments/prod/main.tf manages VPC + EKS + 20 services + databases + CDN
  // State file is 50MB; plan takes 10 minutes; every change risks everything
RIGHT: split by service/component; separate state per bounded context.

WRONG: Using workspace for prod/staging isolation
  terraform workspace select prod && terraform apply
  // "Did you select prod or staging?" — easy to mistake; both share same .tf files
RIGHT: separate directory per long-lived environment.

WRONG: Terraform modules that call other modules 3 levels deep
  root → platform-module → networking-module → subnet-module → route-table-module
  // Impossible to understand, debug, or test in isolation
RIGHT: keep module depth to 2 levels maximum. Flat is better than nested.
```
