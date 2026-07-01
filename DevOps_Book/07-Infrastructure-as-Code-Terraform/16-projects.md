# Chapter 16 — Hands-On Projects

## Learning Objectives

By the end of this chapter you will have:

- Bootstrapped a secure remote state backend from scratch
- Built a production-grade, reusable VPC module
- Terraformed a complete 3-tier application stack (ALB, ASG, RDS, Redis)
- Created a GitOps CI/CD pipeline for Terraform using GitHub Actions
- Assembled a full production platform as a portfolio-quality capstone

---

## Project 1 — Terraform Bootstrap (Beginner)

**Goal:** Create the remote state infrastructure that all other Terraform projects depend on.

**What you'll build:**

- S3 bucket for Terraform state (versioned, encrypted with KMS, private)
- DynamoDB table for state locking
- KMS key for state encryption
- IAM policy granting access to the state backend

```hcl
# bootstrap/main.tf — run with local state, commit the tfstate

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  # No backend block — intentional: this is the bootstrap
}

provider "aws" {
  region = var.region
}

variable "region"     { type = string; default = "us-east-1" }
variable "project"    { type = string }
variable "account_id" { type = string }

resource "aws_kms_key" "terraform_state" {
  description             = "Terraform state encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  tags                    = { Name = "${var.project}-terraform-state-key" }
}

resource "aws_kms_alias" "terraform_state" {
  name          = "alias/${var.project}-terraform-state"
  target_key_id = aws_kms_key.terraform_state.key_id
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "${var.project}-terraform-state-${var.account_id}"
  lifecycle { prevent_destroy = true }
  tags   = { Name = "Terraform State", ManagedBy = "terraform" }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "${var.project}-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
  tags = { Name = "Terraform State Lock" }
}

output "state_bucket_name" { value = aws_s3_bucket.terraform_state.bucket }
output "lock_table_name"   { value = aws_dynamodb_table.terraform_locks.name }
output "kms_key_arn"       { value = aws_kms_key.terraform_state.arn }
```

**Success criteria:** S3 bucket exists in AWS with versioning, encryption, and public access blocked. DynamoDB table exists. You can migrate another Terraform project's local state to this backend.

---

## Project 2 — VPC Module (Intermediate)

**Goal:** Build a production-grade, reusable VPC module and publish it for your org.

**Module interface:**

```hcl
# Usage
module "vpc" {
  source = "../../modules/vpc"

  name    = "prod"
  project = "myapp"
  cidr    = "10.0.0.0/16"
  azs     = ["us-east-1a", "us-east-1b"]

  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.3.0/24"]
  private_subnet_cidrs = ["10.0.2.0/24", "10.0.4.0/24"]
  data_subnet_cidrs    = ["10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway       = true
  single_nat_gateway       = false   # one per AZ for HA
  enable_s3_endpoint       = true
  enable_dynamodb_endpoint = true

  tags = { Environment = "prod", ManagedBy = "terraform" }
}
```

**Module outputs needed:** `vpc_id`, `public_subnet_ids`, `private_subnet_ids`, `data_subnet_ids`, `nat_gateway_ids`, `vpc_cidr_block`.

**Implementation requirements:**

1. Three tiers: public (ALB), private (app), data (RDS/Redis — no internet route at all)
2. NAT Gateway: conditional; `single_nat_gateway=false` creates one per AZ
3. VPC endpoints: S3 and DynamoDB gateway endpoints (free; keeps traffic off internet)
4. Enable VPC flow logs to CloudWatch Logs
5. Tags: all subnets tagged with `Tier` (public/private/data) and the AZ

**Test it:** use the module from both dev (`single_nat_gateway=true`, single AZ) and prod (`single_nat_gateway=false`, two AZs). Verify costs differ. Verify flow logs capture traffic.

---

## Project 3 — Complete Application Stack (Intermediate)

**Goal:** Terraform the full 3-tier application — VPC, ALB, ASG, RDS, Redis.

**Architecture in Terraform:**

```
modules/
├── vpc/          (Project 2)
├── compute/      (ALB + ASG + Launch Template)
└── database/     (RDS + ElastiCache + Secrets Manager)

environments/
└── prod/
    └── main.tf   (composes the 3 modules)
```

**What each module must create:**

`modules/compute/`:
- `aws_launch_template` (IMDSv2, gp3 encrypted volume, SSM managed, latest AL2 AMI via data source)
- `aws_autoscaling_group` (multi-AZ, ELB health check, instance_refresh rolling)
- `aws_autoscaling_policy` (target tracking, CPU 60%)
- `aws_lb` (application, access logs to S3)
- `aws_lb_target_group` (HTTP, /health path, deregistration_delay=30)
- `aws_lb_listener` (HTTPS 443, TLS 1.3, forward to TG)
- `aws_lb_listener` (HTTP 80, redirect to HTTPS)
- `aws_iam_role` + profile for EC2 (SSM, Secrets Manager, S3)
- `aws_cloudwatch_metric_alarm` for high CPU and 5xx errors

`modules/database/`:
- `aws_db_instance` (PostgreSQL 15, parameter group, encrypted, multi_az conditional)
- `aws_db_subnet_group`
- `random_password` + `aws_secretsmanager_secret_version`
- `aws_elasticache_replication_group` (Redis 7, encrypted in transit + at rest)
- `aws_elasticache_subnet_group`
- `aws_kms_key` for each (RDS, secrets)

Root `environments/prod/main.tf`: wires modules together, passing VPC outputs to compute and database.

---

## Project 4 — CI/CD Pipeline for Terraform (Advanced)

**Goal:** Full GitOps Terraform pipeline — plan on PR comment, apply on merge, drift detection on schedule.

**GitHub Actions workflows to create:**

`terraform-plan.yml` (triggers on PR):
- Triggered on `pull_request` to `main`, path filter: `infra/**`
- OIDC auth to AWS
- `terraform init`, `fmt -check`, `validate`
- `terraform plan -out=tfplan`
- Upload `tfplan` as artifact
- Post formatted plan output as PR comment (show `+` / `-` / `~` counts and full diff)
- Fail the check if plan fails

`terraform-apply.yml` (triggers on push to main):
- Triggered on `push` to `main`, path filter: `infra/**`
- Downloads `tfplan` artifact from the matching PR's plan run
- OIDC auth
- `terraform init`
- `terraform apply` the saved plan (not re-plan — use the reviewed plan)

`terraform-drift.yml` (scheduled):
- Runs Mon–Fri at 8am UTC
- `terraform plan -refresh-only -detailed-exitcode`
- Exit code 2 = drift → post to Slack via webhook
- Report: which resources drifted and how

**IAM role (create with Terraform in `bootstrap/`):**
- OIDC trust for `repo:myorg/infra:*`
- Permissions scoped to the resources this pipeline manages
- Read-only for plan runs, full write for apply runs (separate roles)

---

## Project 5 — Capstone: Production Platform (Advanced)

**Goal:** Terraform the entire production platform. Everything declared in Terraform — no manual AWS console steps.

1. **Bootstrap**: remote state (Project 1)
2. **Network**: VPC module (Project 2) with 3 AZs, flow logs
3. **Security**: GuardDuty, AWS Config, CloudTrail, KMS keys
4. **Compute**: Application stack (Project 3) with CI/CD pipeline (Project 4)
5. **CDN**: CloudFront distribution in front of ALB, Route53 records, ACM cert
6. **DNS**: Route53 A alias records for all public endpoints
7. **Monitoring**: CloudWatch dashboard (`aws_cloudwatch_dashboard`), all alarms, SNS topics

```hcl
# environments/prod/main.tf — the capstone root module

module "network" {
  source = "../../modules/vpc"
  # ... VPC variables
}

module "security" {
  source = "../../modules/security"
  # ... GuardDuty, Config, CloudTrail
}

module "compute" {
  source         = "../../modules/compute"
  vpc_id         = module.network.vpc_id
  subnet_ids     = module.network.private_subnet_ids
  alb_subnet_ids = module.network.public_subnet_ids
  # ...
}

module "database" {
  source     = "../../modules/database"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.data_subnet_ids
  # ...
}

module "cdn" {
  source          = "../../modules/cdn"
  alb_dns_name    = module.compute.alb_dns_name
  certificate_arn = module.compute.certificate_arn
  # ...
}

module "monitoring" {
  source         = "../../modules/monitoring"
  asg_name       = module.compute.asg_name
  alb_arn_suffix = module.compute.alb_arn_suffix
  rds_identifier = module.database.rds_identifier
  # ...
}
```

This is portfolio-quality Terraform. Put it in a public GitHub repo. It demonstrates every skill from this course and is directly applicable to real production environments.

---

## Summary

These 5 projects build on each other in a deliberate progression. Start with Project 1 (takes about an hour), and each subsequent project reuses what you built before. By the end you have a complete, production-grade, GitOps-managed infrastructure platform — the kind that appears in job descriptions for senior DevOps and platform engineering roles.

| Project | Level | Approximate Time | Key Skills |
|---------|-------|-----------------|------------|
| 1 — Bootstrap | Beginner | 1 hour | S3, DynamoDB, KMS, remote state |
| 2 — VPC Module | Intermediate | 3 hours | Modules, for_each, conditional resources |
| 3 — App Stack | Intermediate | 5 hours | Multi-module composition, IAM, secrets |
| 4 — CI/CD Pipeline | Advanced | 4 hours | GitHub Actions, OIDC, drift detection |
| 5 — Capstone | Advanced | 8 hours | Everything — full production platform |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-best-practices.md">← Previous: Best Practices & Patterns</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="17-course-summary.md">Next: Course Summary →</a>
</div>
