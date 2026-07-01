# Chapter 5 — Variables, Outputs & Data Sources

---

## Learning Objectives

By the end of this chapter you will be able to:

- Declare input variables with types, defaults, descriptions, and validation rules
- Supply variable values via `.tfvars` files, environment variables, and CLI flags
- Use `locals` to compute derived values and avoid repetition
- Expose resource attributes as outputs and consume them in CI/CD pipelines
- Use data sources to read existing infrastructure without managing it
- Fetch secrets and configuration from AWS SSM Parameter Store and Secrets Manager
- Pass outputs between independent Terraform state files via `terraform_remote_state`

---

## 5.1 Input Variables

```hcl
# variables.tf

# Simple variable with type and default
variable "region" {
  type        = string
  description = "AWS region to deploy into"
  default     = "us-east-1"
}

# Required variable (no default — must be supplied by the caller)
variable "environment" {
  type        = string
  description = "Deployment environment: dev, staging, or prod"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}

# Sensitive variable (value is redacted from plan/apply output and logs)
variable "db_password" {
  type      = string
  sensitive = true
}

# Complex type: list of objects
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    description = string
  }))
  default = [
    { port = 80,  protocol = "tcp", description = "HTTP" },
    { port = 443, protocol = "tcp", description = "HTTPS" }
  ]
}
```

**Supported types:**

| Type | Example |
|---|---|
| `string` | `"us-east-1"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b", "c"]` |
| `map(string)` | `{ key = "value" }` |
| `set(string)` | Unordered unique strings |
| `object({...})` | Structured object with named attributes |
| `tuple([...])` | Fixed-length sequence of typed values |
| `any` | No type constraint (avoid unless necessary) |

---

## 5.2 Setting Variable Values

Variable values are resolved in order of precedence (highest wins):

```bash
# 1. Command-line flag (highest precedence)
terraform apply -var="environment=prod"
terraform apply -var="environment=prod" -var="instance_type=t3.large"

# 2. .tfvars file explicitly specified
terraform apply -var-file="prod.tfvars"

# 3. terraform.tfvars (auto-loaded if present in the working directory)
#    also: terraform.tfvars.json

# 4. *.auto.tfvars files (auto-loaded alphabetically)
#    e.g., prod.auto.tfvars, common.auto.tfvars

# 5. Environment variables: TF_VAR_<variable_name>
export TF_VAR_environment=prod
export TF_VAR_db_password=mysecretpassword
terraform apply

# 6. Default value in the variable declaration (lowest precedence)
```

```hcl
# prod.tfvars — values for production
environment    = "prod"
instance_type  = "t3.large"
min_capacity   = 3
max_capacity   = 10
multi_az       = true
```

Keep one `.tfvars` file per environment and commit them to source control. Never commit sensitive values — use `TF_VAR_*` environment variables or fetch from a secrets store (see 5.6).

---

## 5.3 Locals

Locals are computed values that are derived from variables, resource attributes, or other locals. They reduce repetition and make complex expressions readable.

```hcl
# locals.tf

locals {
  # Consistent name prefix for all resources
  name_prefix = "${var.project}-${var.environment}"

  # Common tags applied to every resource
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.owner
  }

  # Conditional logic
  is_production  = var.environment == "prod"
  instance_type  = local.is_production ? "t3.large" : "t3.micro"
  retention_days = local.is_production ? 30 : 7
}

# Use locals anywhere in the configuration
resource "aws_s3_bucket" "data" {
  bucket = "${local.name_prefix}-data"
  tags   = local.common_tags
}
```

Locals are not exposed outside the module — they are for internal use only. To expose values to callers, use outputs (5.4).

---

## 5.4 Output Values

Outputs expose resource attributes after `terraform apply`. They serve three purposes:

1. Display useful information to the operator after an apply
2. Feed values into CI/CD pipelines via `terraform output`
3. Expose values to other Terraform configurations via `terraform_remote_state`

```hcl
# outputs.tf

output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "rds_endpoint" {
  description = "RDS instance connection endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true   # redacted in CLI output; still stored in state
}

# Output a structured map of all useful endpoints
output "service_endpoints" {
  description = "All service connection endpoints"
  value = {
    alb   = aws_lb.main.dns_name
    rds   = aws_db_instance.main.address
    redis = aws_elasticache_cluster.main.cache_nodes[0].address
  }
}
```

```bash
# Retrieve outputs after apply
terraform output
terraform output alb_dns_name
terraform output -raw alb_dns_name    # no quotes, suitable for shell variable assignment
terraform output -json                # all outputs as JSON (for scripting)

# In CI/CD pipelines:
ALB_DNS=$(terraform output -raw alb_dns_name)
curl "https://$ALB_DNS/health"
```

---

## 5.5 Data Sources

Data sources let you read existing resources or external information without creating or managing anything. They are read-only.

```hcl
# Fetch the latest Amazon Linux 2 AMI dynamically
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id   # always uses the latest AL2 AMI
  instance_type = var.instance_type
}
```

```hcl
# Read your current AWS account ID and region
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

resource "aws_s3_bucket" "audit_logs" {
  bucket = "audit-logs-${data.aws_caller_identity.current.account_id}-${data.aws_region.current.name}"
}

# Read an existing VPC that is not managed by this Terraform configuration
data "aws_vpc" "shared" {
  filter {
    name   = "tag:Name"
    values = ["shared-vpc"]
  }
}

resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.shared.id
  cidr_block = "10.0.10.0/24"
}
```

Data sources are evaluated during `terraform plan`, so their values are available in the execution plan output.

---

## 5.6 Fetching SSM Parameters and Secrets

```hcl
# Read a value from AWS SSM Parameter Store
data "aws_ssm_parameter" "db_host" {
  name = "/myapp/prod/db-host"
}

# Read a secret from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "prod/database/credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

resource "aws_lambda_function" "api" {
  # ...
  environment {
    variables = {
      DB_HOST     = data.aws_ssm_parameter.db_host.value
      DB_PASSWORD = local.db_creds.password   # marked sensitive automatically
    }
  }
}
```

This pattern avoids storing secrets in `.tfvars` files or source control. The secrets live in AWS and are fetched at plan/apply time by whoever is running Terraform.

---

## 5.7 Variable Validation

```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr must be a valid IPv4 CIDR block."
  }

  validation {
    condition     = split("/", var.vpc_cidr)[1] == "16"
    error_message = "vpc_cidr must be a /16 block for this module."
  }
}

variable "tags" {
  type = map(string)

  validation {
    condition     = contains(keys(var.tags), "Environment")
    error_message = "tags must include an 'Environment' key."
  }
}
```

Validations run before any API calls are made. They catch configuration mistakes at plan time, giving fast feedback without requiring apply to fail mid-run.

---

## 5.8 Passing Outputs Between Configurations

Large infrastructures are split across multiple Terraform state files (e.g., network, compute, databases in separate root modules). Use `terraform_remote_state` to read another configuration's outputs.

```hcl
# Read outputs from the network layer's state file
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
}
```

The calling configuration only has read access to outputs — it cannot access resources or locals from the remote state. This is intentional: it enforces clean API boundaries between infrastructure layers.

---

## Summary

- Input variables parameterise configurations. Always add `description`, `type`, and `validation` blocks.
- Variable values are resolved in a clear precedence order: CLI flag > `-var-file` > `terraform.tfvars` > `*.auto.tfvars` > `TF_VAR_*` > default.
- `sensitive = true` on a variable or output redacts its value in CLI output but does not encrypt it in state.
- Locals compute derived values and apply DRY principles — use them for name prefixes, tag maps, and conditional expressions.
- Outputs expose values to operators and downstream configurations. Mark secrets as `sensitive`.
- Data sources are read-only: they fetch existing resources without creating anything.
- Fetch secrets at apply time from SSM or Secrets Manager rather than storing them in `.tfvars` files.
- `terraform_remote_state` is the standard way to share outputs between independent state files.

---

## Knowledge Check

1. What is the precedence order when the same variable is set in both `terraform.tfvars` and a `TF_VAR_*` environment variable?
2. What is the difference between a `variable` and a `local`?
3. You have a resource attribute that you want to expose to a CI/CD pipeline. What is the correct mechanism?
4. Why should you use a data source to fetch the latest AMI rather than hardcoding an AMI ID?
5. A sensitive variable is marked `sensitive = true`. Is its value encrypted in the state file?

---

## Hands-on Exercise

Refactor a monolithic `main.tf` into properly separated files:

1. Move all `variable` blocks to `variables.tf`. Add type constraints, descriptions, and validation rules to at least two variables.
2. Extract computed values (name prefix, tag map, conditional instance type) into `locals.tf`.
3. Move all `output` blocks to `outputs.tf`. Mark at least one output `sensitive = true`.
4. Create `prod.tfvars` and `dev.tfvars` with meaningfully different values. Apply each and observe the differences in the plan.
5. Replace any hardcoded AMI ID with a `data "aws_ami"` block that always fetches the latest Amazon Linux 2 AMI.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="04-providers-and-resources.md">← Previous: Providers & Resources</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="06-state-management.md">Next: State Management →</a>
</div>
