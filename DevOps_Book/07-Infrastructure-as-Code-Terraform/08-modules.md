# Chapter 8 — Modules

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what a Terraform module is and why modules matter
- Build a well-structured, reusable module with proper variables and outputs
- Call a local module from a root configuration
- Use public modules from the Terraform Registry
- Pin module versions to prevent unexpected breakage
- Compose multiple modules together to build complex infrastructure
- Recognise and avoid common module anti-patterns
- Describe options for sharing modules across teams

---

## 8.1 What Are Modules?

A **module** is a reusable, self-contained package of Terraform configurations that manages a related set of resources. Think of it like a function in a programming language: it takes inputs (variables), performs a specific task (creates resources), and exposes outputs that callers can consume.

**Why use modules?**

- **DRY (Don't Repeat Yourself):** write your VPC logic once, use it in every environment
- **Encapsulation:** callers don't need to understand every resource inside — they just provide inputs
- **Consistency:** every team that uses the `vpc` module gets the same security, tagging, and naming conventions
- **Collaboration:** platform teams publish modules; application teams consume them without needing deep Terraform expertise

**Every Terraform directory is a module.** The directory you run `terraform apply` in is the **root module**. Any other directory you reference with `module {}` blocks is a **child module**.

---

## 8.2 Module Structure

A well-structured module follows a consistent file layout:

```
modules/
└── vpc/
    ├── main.tf          # all resource definitions
    ├── variables.tf     # module inputs
    ├── outputs.tf       # module outputs
    ├── versions.tf      # required providers and Terraform version
    └── README.md        # usage documentation with example
```

**variables.tf — module inputs:**

```hcl
variable "name" {
  description = "Name prefix applied to all resources"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC (e.g. 10.0.0.0/16)"
  type        = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr must be a valid CIDR block."
  }
}

variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for public subnets, one per AZ"
  type        = list(string)
  default     = []
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets, one per AZ"
  type        = list(string)
  default     = []
}

variable "azs" {
  description = "List of Availability Zones to deploy subnets into"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Create a NAT gateway so private subnets can reach the internet"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Map of tags applied to all resources"
  type        = map(string)
  default     = {}
}
```

**outputs.tf — module outputs:**

```hcl
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_id" {
  description = "ID of the NAT gateway (empty string if not created)"
  value       = var.enable_nat_gateway ? aws_nat_gateway.this[0].id : ""
}
```

---

## 8.3 VPC Module Example

**main.tf — complete VPC module:**

```hcl
# VPC
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, { Name = var.name })
}

# Internet Gateway
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(var.tags, { Name = "${var.name}-igw" })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${var.azs[count.index]}"
    Tier = "public"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]

  tags = merge(var.tags, {
    Name = "${var.name}-private-${var.azs[count.index]}"
    Tier = "private"
  })
}

# Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"

  tags = merge(var.tags, { Name = "${var.name}-nat-eip" })
}

# NAT Gateway (placed in the first public subnet)
resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = merge(var.tags, { Name = "${var.name}-nat" })

  depends_on = [aws_internet_gateway.this]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(var.tags, { Name = "${var.name}-public-rt" })
}

# Private Route Table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.this[0].id
    }
  }

  tags = merge(var.tags, { Name = "${var.name}-private-rt" })
}

# Route Table Associations — Public
resource "aws_route_table_association" "public" {
  count = length(var.public_subnet_cidrs)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Route Table Associations — Private
resource "aws_route_table_association" "private" {
  count = length(var.private_subnet_cidrs)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

---

## 8.4 Using a Local Module

From a root module (e.g. `environments/prod/main.tf`), call the local module with a relative path:

```hcl
module "vpc" {
  source = "../../modules/vpc"

  name                 = "prod-vpc"
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.3.0/24"]
  private_subnet_cidrs = ["10.0.2.0/24", "10.0.4.0/24"]
  azs                  = ["us-east-1a", "us-east-1b"]
  enable_nat_gateway   = true

  tags = local.common_tags
}

# Consume module outputs in other resources
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = module.vpc.private_subnet_ids[0]
}
```

After adding a `module` block, always run `terraform init` before `terraform plan` — Terraform needs to download or resolve the module source.

---

## 8.5 Public Registry Modules

[registry.terraform.io](https://registry.terraform.io) hosts thousands of community and official modules. The `terraform-aws-modules` GitHub organisation maintains the most widely-used AWS modules (VPC, EKS, RDS, etc.) and carries **official** status from HashiCorp.

```hcl
# Official VPC module from the registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = false   # one NAT per AZ for HA
}

# Official EKS module consuming VPC outputs
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "prod-eks"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
}
```

**Module tiers on the registry:**

| Tier | Badge | Meaning |
|---|---|---|
| Official | Blue "Official" | Maintained by HashiCorp |
| Partner | Purple "Partner" | Maintained by a verified technology partner |
| Community | None | Maintained by the community — review carefully |

Always pin a `version` constraint when using registry modules; otherwise `terraform init -upgrade` can pull a breaking major version.

---

## 8.6 Module Versioning

```hcl
# Pin to an exact version — most predictable for production
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}

# Allow patch updates only (safe in most cases)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"
}

# Git-based module pinned to a tag — good for internal modules
module "internal_vpc" {
  source = "git::https://github.com/myorg/terraform-modules.git//vpc?ref=v1.2.0"
}

# Git pinned to a specific commit hash — highest stability guarantee
module "internal_vpc" {
  source = "git::https://github.com/myorg/terraform-modules.git//vpc?ref=abc1234"
}
```

**Rule:** always pin module versions. An unpinned module will silently pull the latest version on the next `terraform init -upgrade`, potentially introducing breaking changes or unintended resource replacements.

---

## 8.7 Module Composition

Real infrastructure is built by composing modules together. Outputs from one module become inputs to the next:

```hcl
# 1. Networking layer
module "vpc" {
  source = "../../modules/vpc"
  name   = "prod"
  # ...
}

# 2. Database layer — needs VPC outputs
module "rds" {
  source = "../../modules/rds"

  name               = "prod-db"
  vpc_id             = module.vpc.vpc_id          # output from vpc module
  subnet_ids         = module.vpc.private_subnet_ids
  # ...
}

# 3. Application layer — needs both
module "app" {
  source = "../../modules/app"

  name             = "prod-app"
  vpc_id           = module.vpc.vpc_id
  subnet_ids       = module.vpc.private_subnet_ids
  db_endpoint      = module.rds.endpoint          # output from rds module
  db_secret_arn    = module.rds.secret_arn
  # ...
}
```

Terraform automatically resolves the dependency graph: `vpc` is created first, then `rds` and `app` (in parallel if they don't depend on each other).

---

## 8.8 Module Anti-Patterns

```
AVOID

✗ One giant module
  Everything in a single module directory.
  Hard to reuse parts independently; a change to one resource plans everything.

✗ Overly generic module
  50 variables to cover every possible use case.
  Callers can't understand what to pass; just use resources directly.

✗ Leaking internals
  Outputting every resource attribute from a module.
  Callers become dependent on internal implementation details.

✗ No versioning
  Module source without a version constraint.
  Breaks silently when the module author publishes a new version.

✗ Monolith state
  All modules called from one root with a single state file.
  Every plan touches all resources; a bad apply can destroy everything.


RECOMMENDED

✓ Small, focused modules — one module = one infrastructure concern (vpc, rds, app)
✓ Opinionated defaults — sensible defaults for 80% of cases, callers override what they need
✓ Clean interface — minimise required variables; most should have defaults
✓ Stable outputs — only expose what callers genuinely need
✓ Well-documented README with a copy-paste usage example
✓ Automated tests with Terratest or similar before publishing a new version
```

---

## 8.9 Creating an Internal Module Registry

For sharing modules within an organisation you have several options:

**Option 1 — Git repositories (simplest)**

One repository per module, tagged with semantic versions:

```
github.com/myorg/terraform-module-vpc    (tags: v1.0.0, v1.1.0, v2.0.0)
github.com/myorg/terraform-module-rds
github.com/myorg/terraform-module-eks
```

Callers reference modules by tag:

```hcl
module "vpc" {
  source = "git::https://github.com/myorg/terraform-module-vpc.git?ref=v1.1.0"
}
```

**Option 2 — Terraform Cloud private registry**

Upload module packages to the HCP Terraform (formerly Terraform Cloud) private registry. Supports semantic versioning, usage tracking, and access control. Callers reference modules the same way as the public registry but using a private hostname.

**Option 3 — Artifactory or Nexus**

Generic binary repository managers support Terraform modules as versioned artifacts.

**Best practice for internal modules:**

- One Git repository per module (`terraform-module-<name>`)
- Tagged releases following semantic versioning (`v<MAJOR>.<MINOR>.<PATCH>`)
- Automated tests with [Terratest](https://github.com/gruntwork-io/terratest) run on every PR
- CHANGELOG.md documenting breaking changes between major versions
- A team or individual designated as the module owner for maintenance and reviews

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Module | Reusable configuration package with inputs, outputs, and resources |
| Module structure | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md` |
| Local module | Source is a relative filesystem path; run `terraform init` after adding |
| Registry module | Source is `<namespace>/<module>/<provider>`; always pin `version` |
| Git module | Source is a git URL with `?ref=<tag>` for version pinning |
| Module outputs | Used as inputs to other modules — explicit dependency chain |
| Composition | Build complex infra by wiring module outputs into module inputs |
| Anti-patterns | Giant modules, no versioning, leaking internals, monolith state |

---

## Knowledge Check

1. What is the difference between a root module and a child module in Terraform?
2. You add a `module` block to your configuration. What command must you run before `terraform plan` will succeed?
3. Why should you always specify a `version` constraint when using a module from the Terraform Registry?
4. A colleague suggests putting all your VPC, RDS, EKS, and application resources into one large module so everything deploys together. What problems does this create?
5. How do you pass the output of a `vpc` module as an input to an `rds` module in the same root configuration?

---

## Hands-on Exercise

**Goal:** build and consume a reusable VPC module.

1. Create `modules/vpc/` with `main.tf`, `variables.tf`, `outputs.tf`, and `versions.tf` using the code from sections 8.2 and 8.3. Include variables for `name`, `vpc_cidr`, `public_subnet_cidrs`, `private_subnet_cidrs`, `azs`, `enable_nat_gateway`, and `tags`.

2. Create `environments/dev/main.tf` that calls the module with a `10.0.0.0/16` CIDR and two AZs. Set `enable_nat_gateway = false` to save cost in dev.

3. Create `environments/prod/main.tf` that calls the same module with a `10.1.0.0/16` CIDR and `enable_nat_gateway = true`.

4. Run `terraform init && terraform plan` in both environment directories and confirm each produces its own independent plan.

5. Create a second module `modules/security-groups/` that accepts `vpc_id` as a required input variable and creates an application security group. Call this module from both environment directories, passing `module.vpc.vpc_id` as the input.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-remote-backends.md">← Previous: Remote Backends & State Locking</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-workspaces-and-environments.md">Next: Workspaces & Environments →</a>
</div>
