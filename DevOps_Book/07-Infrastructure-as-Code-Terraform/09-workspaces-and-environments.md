# Chapter 9 — Workspaces & Environments

---

## Learning Objectives

By the end of this chapter you will be able to:

- Articulate the problem with copy-pasting Terraform directories for each environment
- Create, switch, and delete Terraform workspaces from the CLI
- Understand how workspaces map to separate state files in S3
- Use `terraform.workspace` in configuration logic
- Explain why separate environment directories with shared modules is the recommended approach
- Structure an `environments/` directory tree for dev, staging, and prod
- Configure multi-account AWS deployments with per-environment provider assume_role
- Describe the environment promotion pattern for safely rolling out changes
- Know when Terragrunt adds value and what it solves

---

## 9.1 The Multi-Environment Problem

Every production system needs at least two environments: somewhere to test changes (dev/staging) and production. The naive Terraform approach is to copy the entire project directory for each environment:

```
infra-dev/
infra-staging/
infra-prod/
```

**Why this fails:**

- **90% duplicate code:** a bug fix in the VPC logic must be applied in three places
- **Configurations drift:** over time developers make "temporary" changes to dev that never make it to prod
- **Inconsistency:** dev and prod gradually diverge until you can't trust that testing in dev means anything
- **Toil:** every change is a three-PR process

There are two common solutions — Terraform Workspaces and separate environment directories — and one thin-wrapper tool, Terragrunt, that reduces boilerplate further.

---

## 9.2 Terraform Workspaces

A workspace is a named slot for a separate state file. The same Terraform code in the same directory can manage multiple independent sets of infrastructure by switching between workspaces.

**Workspace CLI commands:**

```bash
terraform workspace list        # list all workspaces (* marks current)
# * default

terraform workspace new dev
# Created and switched to workspace "dev"!

terraform workspace new staging
terraform workspace new prod

terraform workspace select prod
# Switched to workspace "prod"!

terraform workspace show
# prod

terraform workspace delete dev  # workspace must have no state (empty) first
```

**How workspaces map to S3 state files:**

```
s3://my-state-bucket/
├── env:/
│   ├── dev/
│   │   └── terraform.tfstate
│   ├── staging/
│   │   └── terraform.tfstate
│   └── prod/
│       └── terraform.tfstate
└── terraform.tfstate   # default workspace
```

Each workspace gets its own state file, so `terraform apply` in the `prod` workspace only touches prod resources.

**Using the workspace name in configuration:**

```hcl
locals {
  is_production = terraform.workspace == "prod"

  instance_type = local.is_production ? "t3.large"  : "t3.micro"
  min_capacity  = local.is_production ? 3            : 1
  multi_az      = local.is_production
  db_class      = local.is_production ? "db.t3.large" : "db.t3.micro"
}

resource "aws_instance" "app" {
  instance_type = local.instance_type
  # ...
}

resource "aws_db_instance" "main" {
  instance_class = local.db_class
  multi_az       = local.multi_az
  # ...
}
```

**When to use workspaces:**

Workspaces work well for **lightweight environment differences** — same account, same region, slightly different sizes. They become awkward when environments need meaningfully different configurations (different accounts, different VPC layouts, different provider settings), because all that logic has to live as `terraform.workspace == "prod" ? ... : ...` conditionals scattered throughout the code.

---

## 9.3 Separate Environment Directories (Recommended)

For most real-world setups, the better pattern is separate directories per environment that all **call the same shared modules**. The modules are the single source of truth; the environment directories only supply variable values.

**Directory layout:**

```
infra/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── app/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── providers.tf
    │   └── terraform.tfvars
    ├── staging/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── providers.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── variables.tf
        ├── providers.tf
        └── terraform.tfvars
```

**environments/prod/main.tf:**

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}

module "vpc" {
  source = "../../modules/vpc"
  version = "1.3.0"   # or use relative path for local modules

  name                 = "prod-vpc"
  vpc_cidr             = var.vpc_cidr
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  azs                  = var.azs
  enable_nat_gateway   = true
  tags                 = local.common_tags
}

module "app" {
  source = "../../modules/app"

  name       = "prod-app"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  min_size   = 3
  max_size   = 10
  tags       = local.common_tags
}
```

**environments/prod/terraform.tfvars:**

```hcl
vpc_cidr             = "10.1.0.0/16"
public_subnet_cidrs  = ["10.1.1.0/24", "10.1.3.0/24", "10.1.5.0/24"]
private_subnet_cidrs = ["10.1.2.0/24", "10.1.4.0/24", "10.1.6.0/24"]
azs                  = ["us-east-1a", "us-east-1b", "us-east-1c"]
```

**environments/dev/main.tf:**

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "dev/terraform.tfstate"    # different key from prod
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}

module "vpc" {
  source = "../../modules/vpc"

  name                 = "dev-vpc"
  vpc_cidr             = var.vpc_cidr
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  azs                  = var.azs
  enable_nat_gateway   = false   # save cost in dev
  tags                 = local.common_tags
}

module "app" {
  source = "../../modules/app"

  name       = "dev-app"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  min_size   = 1
  max_size   = 2
  tags       = local.common_tags
}
```

**environments/dev/terraform.tfvars:**

```hcl
vpc_cidr             = "10.0.0.0/16"
public_subnet_cidrs  = ["10.0.1.0/24"]
private_subnet_cidrs = ["10.0.2.0/24"]
azs                  = ["us-east-1a"]
```

---

## 9.4 Multi-Account Environments

The gold standard for environment isolation is **separate AWS accounts** — one for dev, one for staging, one for prod. This is the approach recommended by AWS in the Well-Architected Framework and used by most mature organisations.

**Benefits:**

- A runaway `terraform destroy` in dev cannot touch prod resources
- IAM policies, service quotas, and billing are completely separate
- Security tooling (GuardDuty, Security Hub) has a smaller blast radius per account
- Cost allocation is clean: prod account = prod costs

**Provider configuration with account-specific assume_role:**

```hcl
# environments/prod/providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::111122223333:role/TerraformDeployRole"
    session_name = "terraform-prod"
  }

  default_tags {
    tags = {
      Environment = "prod"
      ManagedBy   = "terraform"
    }
  }
}
```

```hcl
# environments/dev/providers.tf
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::444455556666:role/TerraformDeployRole"
    session_name = "terraform-dev"
  }

  default_tags {
    tags = {
      Environment = "dev"
      ManagedBy   = "terraform"
    }
  }
}
```

The CI/CD pipeline assumes different IAM roles per environment. The same Terraform module code runs in both — only the provider (and therefore the AWS account) differs.

---

## 9.5 Environment Promotion Pattern

A promotion pipeline ensures the same module version that was tested in dev and staging reaches prod — not a slightly different version written under time pressure.

```
dev  →  staging  →  prod

Step 1: Apply module vpc v1.5.0 to dev environment → run smoke tests
Step 2: Apply module vpc v1.5.0 to staging → run integration tests
Step 3: PR approved by platform team → apply to prod (requires manual approval gate)

Key principle: same module version, same variable structure, only values differ.
```

**In practice — pin module versions per environment and promote the pin:**

```hcl
# environments/dev/main.tf
module "vpc" {
  source  = "git::https://github.com/myorg/terraform-module-vpc.git?ref=v1.5.0"
  # ...
}

# environments/staging/main.tf  (update after dev tests pass)
module "vpc" {
  source  = "git::https://github.com/myorg/terraform-module-vpc.git?ref=v1.5.0"
  # ...
}

# environments/prod/main.tf  (update after staging tests pass)
module "vpc" {
  source  = "git::https://github.com/myorg/terraform-module-vpc.git?ref=v1.5.0"
  # ...
}
```

Promotion is a pull request that bumps the `ref=` tag in the next environment's `main.tf`. Code review on that PR is the approval gate.

---

## 9.6 Terragrunt Overview

[Terragrunt](https://terragrunt.gruntwork.io/) is a thin wrapper around Terraform that reduces the environment boilerplate described above. Instead of duplicating `backend` configuration and provider settings across every environment directory, you define them once in a root `terragrunt.hcl` and reference them from each leaf.

**Example root terragrunt.hcl:**

```hcl
# infra/terragrunt.hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "mycompany-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

**Example leaf environments/prod/vpc/terragrunt.hcl:**

```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "git::https://github.com/myorg/terraform-module-vpc.git?ref=v1.5.0"
}

inputs = {
  name     = "prod-vpc"
  vpc_cidr = "10.1.0.0/16"
  # ...
}
```

**Apply all environments at once:**

```bash
terragrunt run-all apply   # applies all leaf modules in dependency order
terragrunt run-all plan    # plan all
```

Terragrunt is widely used in large organisations with many environments and modules. This course focuses on native Terraform, but if you encounter a codebase with `.hcl` files in every directory, that is Terragrunt.

---

## 9.7 Environment Isolation Checklist

Use this checklist when setting up a new environment:

```
Infrastructure isolation
□ Dev and prod are in separate AWS accounts (minimum: separate VPCs with no peering)
□ Each environment has its own state file under a unique S3 key
□ Environments cannot communicate (no VPC peering between dev and prod)

Deployment safety
□ CI/CD applies dev automatically on merge to main
□ CI/CD requires a manual approval gate before applying to prod
□ Prod has prevent_destroy = true on stateful resources (databases, S3 buckets)
□ Prod plan is always reviewed before apply (plan output posted as PR comment)

Credentials & secrets
□ Per-environment IAM roles with least-privilege policies
□ Per-environment secrets (dev DB password is different from prod DB password)
□ Secrets never stored in .tfvars files committed to git — use AWS Secrets Manager

Tagging & cost
□ All resources tagged with Environment for cost allocation in AWS Cost Explorer
□ Tagged with ManagedBy = terraform to distinguish from manually created resources

Cost optimisation
□ Dev environments auto-shutdown or scale to zero on nights and weekends
   (Scheduled Lambda or AWS Instance Scheduler — saves ~70% of dev compute cost)
□ Spot instances or t3 burstable types used in dev/staging where appropriate
```

---

## Summary

| Approach | Pros | Cons |
|---|---|---|
| Copy-paste directories | Simple to understand | Code duplication, configuration drift |
| Workspaces | Single codebase, easy to switch | Conditionals everywhere, same account/region |
| Separate env directories | Clean isolation, per-env config, multi-account | Some repetition of backend/provider blocks |
| Terragrunt | Eliminates backend/provider boilerplate | Another tool to learn and maintain |

**Recommendation for most teams:** separate environment directories sharing common modules, with each environment in its own AWS account. Add Terragrunt if the number of environments and modules grows to the point where the boilerplate becomes painful.

---

## Knowledge Check

1. What problem do Terraform workspaces solve, and what is their main limitation compared to separate environment directories?
2. When you switch to a new workspace and run `terraform apply`, where does Terraform store the new state file (assuming an S3 backend)?
3. Why is it safer to deploy each environment into a separate AWS account rather than separate VPCs in the same account?
4. Describe the environment promotion pattern. What changes between dev, staging, and prod — and what stays the same?
5. You have 20 environment directories all with identical `backend "s3" {}` blocks. Which tool could reduce this repetition, and what is the key mechanism it uses?

---

## Hands-on Exercise

**Goal:** refactor a single Terraform configuration into separate dev and prod environments sharing common modules.

1. Take the VPC module you built in Chapter 8 (`modules/vpc/`) and place it under `infra/modules/vpc/`.

2. Create `infra/environments/dev/main.tf` and `infra/environments/prod/main.tf`. Both should call `../../modules/vpc` but supply different CIDR blocks and different state backend keys (e.g. `dev/vpc/terraform.tfstate` vs `prod/vpc/terraform.tfstate`).

3. Create a `terraform.tfvars` file in each environment directory with the environment-specific variable values.

4. Run `terraform init && terraform apply` in the dev environment. Note the S3 key where state is stored.

5. Run `terraform init && terraform apply` in the prod environment. Confirm a separate state file is created at the prod key in S3.

6. Run `terraform state list` in both directories and verify that each environment manages its own independent set of resources with no overlap.

7. **Bonus:** add the `prevent_destroy = true` lifecycle rule to the VPC resource inside the module. Attempt to destroy the prod environment. Observe the error Terraform produces.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-modules.md">← Previous: Modules</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-aws-networking.md">Next: Building AWS Networking with Terraform →</a>
</div>
