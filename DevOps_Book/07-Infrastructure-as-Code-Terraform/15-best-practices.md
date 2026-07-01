# Chapter 15 — Best Practices & Patterns

## Learning Objectives

By the end of this chapter you will be able to:

- Structure a Terraform project for production with clear directory conventions
- Apply consistent naming and tagging strategies across all resources
- Handle secrets safely — generating, storing, and referencing them without exposure
- Design a state-splitting strategy that prevents blast-radius incidents
- Protect critical resources from accidental destruction
- Refactor existing Terraform safely using `moved` blocks and `state mv`
- Conduct a thorough code review on a Terraform pull request

---

## 15.1 Code Organisation

```
Recommended project structure for production:
infra/
├── bootstrap/          # S3 backend + DynamoDB lock (run once, local state)
│   └── main.tf
├── modules/            # reusable internal modules
│   ├── vpc/
│   ├── app/
│   ├── rds/
│   └── eks/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── providers.tf
│   │   ├── versions.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
└── .github/
    └── workflows/
        └── terraform.yml
```

**Rules:**

- One root module per environment — separate state, separate apply
- Shared logic lives in `modules/` — never copy HCL between environments
- `bootstrap/` is the only config with local state — run once, commit the small state file
- `versions.tf` pins Terraform and provider versions — prevents unexpected upgrades

---

## 15.2 Naming Conventions

```hcl
# Resource naming: {project}-{environment}-{component}-{suffix}
locals {
  name_prefix = "${var.project}-${var.environment}"
}

resource "aws_vpc" "main" {
  tags = { Name = "${local.name_prefix}-vpc" }
}

resource "aws_subnet" "private" {
  count = length(var.azs)
  tags  = { Name = "${local.name_prefix}-private-${var.azs[count.index]}" }
}

# Terraform resource naming: snake_case, descriptive
# Good:  aws_instance.web_server
# Bad:   aws_instance.ws1
# Bad:   aws_instance.webServer

# File naming: match resource type where possible
# main.tf             — primary resources
# variables.tf        — variable declarations
# outputs.tf          — output declarations
# data.tf             — data sources
# iam.tf              — all IAM resources
# security_groups.tf  — all security groups
```

---

## 15.3 Tagging Strategy

```hcl
# locals.tf — define tags once, merge everywhere
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.team_name
    CostCenter  = var.cost_center
    Repo        = "https://github.com/myorg/infra"
  }
}

# Merge with resource-specific tags
resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
    Role = "webserver"
  })
}

# AWS provider default_tags: apply to ALL resources automatically
provider "aws" {
  default_tags {
    tags = local.common_tags
  }
}

# With default_tags: only add resource-specific tags in each resource block
resource "aws_instance" "web" {
  tags = { Name = "${local.name_prefix}-web", Role = "webserver" }
  # common_tags applied automatically by provider
}
```

---

## 15.4 Secret Management

```
NEVER in Terraform:
  Hardcoded passwords in .tf files
  Passwords in terraform.tfvars committed to git
  Passwords in environment variables printed in CI logs

DO:
  Generate passwords with random_password — never know the value yourself
  Store in Secrets Manager — app reads at runtime
  Mark variables sensitive = true — suppressed in plan/apply output
  Use environment variables for CI credentials: TF_VAR_db_password
  Use OIDC for AWS auth in CI — no static credentials at all
  Encrypt state at rest (S3 SSE-KMS) — state contains all secret values
  Restrict state bucket access — only CI/CD and senior engineers
```

---

## 15.5 State File Strategy

```
Monolith state (avoid):
  One terraform.tfstate for everything
  → terraform plan takes forever (queries all 200 resources)
  → One mistake destroys everything
  → VPC change blocks app deployment

Split state (recommended):
  network/terraform.tfstate       (VPC, subnets, peering)
  platform/terraform.tfstate      (EKS, ECR, shared tools)
  prod/app/terraform.tfstate      (app resources)
  prod/database/terraform.tfstate (RDS, ElastiCache)

Splitting rule: split along dependency lines
  - Network changes rarely → separate state
  - Database is risky → separate state from app
  - App deploys frequently → its own fast state
```

---

## 15.6 Preventing Accidental Destruction

```hcl
# Critical resources: add prevention
resource "aws_db_instance" "main" {
  deletion_protection = true   # RDS won't delete if this is set

  lifecycle {
    prevent_destroy = true   # Terraform errors if you try to destroy
  }
}

resource "aws_s3_bucket" "state" {
  lifecycle {
    prevent_destroy = true
  }
}

# For the entire workspace: set TF_CLI_ARGS_destroy=-target=non_existent
# Effectively disables destroy unless you override
```

---

## 15.7 Code Review Checklist

```
Before merging a Terraform PR, check:

Resources:
  Does the plan match what was intended? (no surprise destroys)
  Are there any -/+ (replace) operations on critical resources? (DB, VPC)
  Are all resources tagged with required tags?
  Are security groups minimally permissive (not 0.0.0.0/0 except on ALB)?
  Is data encrypted at rest (EBS, RDS, S3)?
  Are passwords generated, not hardcoded?

Code:
  Are variable types and validation rules correct?
  Are outputs marked sensitive where appropriate?
  Are module versions pinned?
  Does the code pass terraform validate and terraform fmt -check?
  Are resources using lifecycle prevent_destroy where appropriate?

State:
  Is the backend configuration correct for this environment?
  Will the state key conflict with another configuration?
```

---

## 15.8 Refactoring Terraform Safely

```bash
# Rule: never delete a resource from config without thinking about
# what Terraform will do

# Scenario 1: rename a resource
# Wrong: delete aws_instance.app, add aws_instance.web_server → destroy + create
# Right: add moved block first, then rename

# Scenario 2: move resource into a module
# Wrong: cut/paste into module → destroy + create
# Right: use moved block
moved {
  from = aws_instance.app
  to   = module.app.aws_instance.main
}

# Scenario 3: convert count to for_each
# Wrong: change to for_each → destroys all count instances, recreates as for_each
# Right: moved blocks for each instance
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["server-1"]
}

# Scenario 4: extract a child module to separate state
# Use terraform state mv to move resources to new state file
# Can't do this with a moved block — must use state commands
terraform state pull > /tmp/old-state.json
# manually edit and push — last resort; use Terraform Cloud or state import
```

---

## 15.9 Performance: Speeding Up Terraform

```bash
# Partial plan — only plan changed resources
terraform plan -target=module.app
terraform apply -target=aws_autoscaling_group.app

# Parallelism (default is 10 concurrent operations)
terraform apply -parallelism=20   # be careful — some APIs rate-limit

# Skip refresh for known-good state
terraform apply -refresh=false   # risky if state is stale

# Most impactful: split state correctly
# 500 resources in one state = slow plan; split into 100 each = 5x faster
```

---

## 15.10 Terraform Style Guide Summary

```
Files:
  Use separate files: main.tf, variables.tf, outputs.tf, data.tf, locals.tf
  Group related resources in named files: iam.tf, security_groups.tf

HCL:
  terraform fmt: run before every commit
  2-space indent (terraform fmt enforces this)
  Blank line between resource blocks
  Argument alignment within blocks (terraform fmt does this)
  Use locals for repeated expressions, not inline everywhere

Modules:
  README.md with usage example, inputs table, outputs table
  Minimal required variables, good defaults
  Version all module sources

Operations:
  Always plan before apply
  Never force-unlock without verifying nobody else is running
  Never edit state files manually
  Review plan output for -/+ (replace) operations carefully
```

---

## Summary

This chapter covered the patterns that separate production Terraform from hobby projects. The most impactful practices are: splitting state along dependency lines, using `default_tags` on the AWS provider, generating passwords with `random_password`, and adding `prevent_destroy` on databases and state buckets. Refactoring safely with `moved` blocks is the single biggest thing that prevents accidental resource replacement.

---

## Knowledge Check

1. Why should `bootstrap/` use local state rather than the remote backend you're creating?
2. You need to rename `aws_instance.app` to `aws_instance.web_server`. What happens if you just rename it in the config without a `moved` block?
3. What is the difference between `deletion_protection = true` (an AWS API flag) and `lifecycle { prevent_destroy = true }` (a Terraform meta-argument)? When does each one help?
4. Your `terraform plan` is taking 4 minutes. The config has 400 resources across VPC, database, and app layers, all in one root module. What is the most impactful fix?
5. A colleague's PR adds a new `aws_db_instance` but stores the password as `password = var.db_password` with the variable sourced from a `terraform.tfvars` file that is committed to git. List all the things wrong with this approach and the correct alternative.

---

## Hands-On Exercise

Audit an existing Terraform codebase against this chapter's checklist. Fix at least three issues:

1. Add `default_tags` to the AWS provider block (removing duplicated tag blocks from individual resources).
2. Add `prevent_destroy = true` to any database or stateful resource that lacks it.
3. If all resources are in a single root module, use `terraform state mv` to split the networking resources (VPC, subnets, security groups) into a separate root module with its own state file.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="14-cicd-integration.md">← Previous: CI/CD Integration & Atlantis</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="16-projects.md">Next: Hands-On Projects →</a>
</div>
