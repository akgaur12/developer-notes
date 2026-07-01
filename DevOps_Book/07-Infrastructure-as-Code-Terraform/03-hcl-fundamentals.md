# Chapter 3 — HCL Fundamentals

---

## Learning Objectives

By the end of this chapter you will be able to:

- Read and write HCL syntax confidently
- Identify and use all major block types (resource, data, variable, output, locals, module)
- Reference attributes from resources, data sources, variables, locals, and modules
- Use the correct type constraint for any variable
- Use common built-in functions (string, collection, numeric, encoding)
- Write conditional expressions to vary configuration by environment
- Use `terraform fmt` and `terraform validate` as development hygiene
- Use `terraform console` to test expressions interactively

---

## 3.1 HCL Overview

HCL (HashiCorp Configuration Language) is the language you write Terraform configurations in.

Key properties:
- Human-readable: designed to be read and written by people, not just machines
- JSON-compatible: every valid HCL file has an equivalent JSON representation (`.tf.json`)
- Declarative: you describe *what* you want, not *how* to do it
- Merged: all `.tf` files in a directory are read together as a single configuration — order does not matter

```
my-infra/
├── main.tf        ┐
├── variables.tf   ├── Terraform merges all of these into one configuration
├── outputs.tf     │
└── providers.tf   ┘
```

Use `.tf` (HCL) for all hand-written configuration. Use `.tf.json` only for machine-generated configs (e.g., from a code generator).

---

## 3.2 Basic Syntax

```hcl
# Single-line comment

/*
  Multi-line comment
*/

# ── Blocks ────────────────────────────────────────────────────────────────
# Syntax: block_type "label1" "label2" { ... }
# Not all block types take labels — see 3.3 for which do

resource "aws_s3_bucket" "my_bucket" {    # block_type "type" "name"
  bucket        = "my-bucket"             # string argument
  force_destroy = true                    # bool argument
}

# ── Primitive value types ──────────────────────────────────────────────────
string_attr    = "hello"
number_attr    = 42
float_attr     = 3.14
bool_true      = true
bool_false     = false
null_attr      = null

# ── Multi-line string (heredoc) ────────────────────────────────────────────
# <<-EOT strips leading whitespace so you can indent it nicely
description = <<-EOT
  This is a multi-line
  string value.
  Indentation is stripped.
EOT

# ── List (sequence of values, same type) ──────────────────────────────────
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
ingress_ports      = [22, 80, 443]

# ── Map (key-value pairs, string keys, same-type values) ──────────────────
tags = {
  Environment = "prod"
  Owner       = "team-platform"
  CostCenter  = "CC-9999"
}
```

---

## 3.3 Block Types

Terraform has a fixed set of block types. Each serves a specific purpose.

**`terraform` block — Terraform settings:**

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Backend — where to store state (covered in Chapter 7)
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

**`provider` block — configure a provider:**

```hcl
provider "aws" {
  region = "us-east-1"
}

# Multiple providers of the same type (e.g., two AWS regions)
provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}
```

**`resource` block — declare an infrastructure object:**

```hcl
resource "aws_instance" "web" {         # resource "TYPE" "NAME"
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
# Referenced elsewhere as: aws_instance.web.id
```

**`data` block — read existing or external data (read-only):**

```hcl
data "aws_ami" "amazon_linux" {         # data "TYPE" "NAME"
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
# Referenced elsewhere as: data.aws_ami.amazon_linux.id
```

**`variable` block — declare an input:**

```hcl
variable "instance_type" {              # variable "NAME"
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type to use for web servers"
}
# Referenced elsewhere as: var.instance_type
```

**`output` block — declare an output value:**

```hcl
output "bucket_arn" {                   # output "NAME"
  value       = aws_s3_bucket.my_bucket.arn
  description = "The ARN of the S3 bucket"
}
```

**`locals` block — computed local values:**

```hcl
locals {
  environment = "production"
  name_prefix = "myapp-${local.environment}"
  common_tags = {
    Environment = local.environment
    ManagedBy   = "terraform"
  }
}
# Referenced elsewhere as: local.name_prefix, local.common_tags
```

**`module` block — use a reusable module:**

```hcl
module "vpc" {                          # module "NAME"
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"
}
# Referenced elsewhere as: module.vpc.vpc_id, module.vpc.private_subnets
```

---

## 3.4 References and Expressions

Terraform uses a consistent reference syntax. Once you know it, reading any Terraform code becomes straightforward.

```hcl
# ── Resource attribute ──────────────────────────────────────────────────────
# Syntax: resource_type.resource_name.attribute
resource "aws_s3_bucket_versioning" "example" {
  bucket = aws_s3_bucket.my_bucket.id
}

# ── Variable ────────────────────────────────────────────────────────────────
# Syntax: var.variable_name
resource "aws_instance" "web" {
  instance_type = var.instance_type
}

# ── Data source ─────────────────────────────────────────────────────────────
# Syntax: data.data_type.data_name.attribute
resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux.id
}

# ── Module output ───────────────────────────────────────────────────────────
# Syntax: module.module_name.output_name
resource "aws_instance" "web" {
  subnet_id = module.vpc.private_subnets[0]
}

# ── Local value ─────────────────────────────────────────────────────────────
# Syntax: local.local_name
resource "aws_s3_bucket" "example" {
  bucket = "${local.name_prefix}-data"
}

# ── String interpolation ────────────────────────────────────────────────────
# Use ${ } inside double-quoted strings to embed expressions
resource "aws_s3_bucket" "example" {
  bucket = "myapp-${var.environment}-${var.region}-data"
}

# ── List index ──────────────────────────────────────────────────────────────
variable "subnets" {
  type    = list(string)
  default = ["subnet-aaa", "subnet-bbb"]
}
resource "aws_instance" "web" {
  subnet_id = var.subnets[0]    # "subnet-aaa"
}

# ── Map key ─────────────────────────────────────────────────────────────────
variable "ami_map" {
  type = map(string)
  default = {
    "us-east-1" = "ami-0c55b159cbfafe1f0"
    "us-west-2" = "ami-0892d3c7ee96c0bf7"
  }
}
resource "aws_instance" "web" {
  ami = var.ami_map["us-east-1"]    # or: var.ami_map[var.region]
}
```

---

## 3.5 Type Constraints

Variables use type constraints to validate inputs and provide clear interfaces.

```hcl
# ── Primitive types ──────────────────────────────────────────────────────────
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

# ── Collection types ─────────────────────────────────────────────────────────
variable "allowed_cidr_blocks" {
  type    = list(string)
  default = ["10.0.0.0/16", "192.168.0.0/24"]
}

variable "instance_counts_by_az" {
  type    = map(number)
  default = {
    "us-east-1a" = 2
    "us-east-1b" = 2
    "us-east-1c" = 1
  }
}

variable "allowed_ports" {
  type    = set(number)       # like list but unordered, no duplicates
  default = [22, 80, 443]
}

# ── Structural types ──────────────────────────────────────────────────────────
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = optional(string, "")   # optional with default (Terraform 1.3+)
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP from internet"
    }
  ]
}

variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    multi_az       = bool
  })
}

# ── any — disables type checking (use sparingly) ──────────────────────────────
variable "extra_tags" {
  type    = any
  default = {}
}
```

**Variable validation (Terraform 0.13+):**

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}
```

---

## 3.6 Built-in Functions

Terraform has a rich library of built-in functions. You cannot define your own functions in HCL (use modules for reusable logic).

**String functions:**

```hcl
upper("hello")                  # "HELLO"
lower("HELLO")                  # "hello"
title("hello world")            # "Hello World"
trimspace("  hello  ")          # "hello"
trim("__hello__", "_")          # "hello"
trimprefix("hello-world", "hello-")  # "world"
trimsuffix("hello.txt", ".txt")      # "hello"
substr("hello world", 0, 5)     # "hello"
length("hello")                 # 5
replace("foo bar baz", " ", "-")     # "foo-bar-baz"
split(",", "a,b,c")             # ["a", "b", "c"]
join(", ", ["a", "b", "c"])     # "a, b, c"
format("Hello, %s! You are %d.", "Alice", 30)  # "Hello, Alice! You are 30."
startswith("hello", "hel")      # true
endswith("hello", "llo")        # true
```

**Collection functions:**

```hcl
length(["a", "b", "c"])         # 3
length({a=1, b=2})              # 2
element(["a", "b", "c"], 1)     # "b"   (wraps around: element(list, 3) = "a")
slice(["a","b","c","d"], 1, 3)  # ["b", "c"]
contains(["a", "b"], "a")       # true
index(["a", "b", "c"], "b")     # 1
flatten([[1,2],[3,[4,5]]])       # [1,2,3,4,5]
distinct(["a","b","a","c"])     # ["a","b","c"]
compact(["a", "", "b", null])   # ["a","b"]
concat(["a","b"], ["c","d"])    # ["a","b","c","d"]
reverse(["a","b","c"])          # ["c","b","a"]
sort(["c","a","b"])             # ["a","b","c"]
keys({a=1, b=2, c=3})          # ["a","b","c"]
values({a=1, b=2, c=3})        # [1,2,3]
merge({a=1}, {b=2}, {a=3})     # {a=3, b=2}  (later maps override earlier)
lookup({env="prod"}, "env", "dev")  # "prod" (third arg = default if key missing)
toset(["a","b","a"])            # {"a","b"}
tolist(toset(["b","a"]))        # ["a","b"]
tomap({a="1", b="2"})          # {a="1", b="2"}
```

**Numeric functions:**

```hcl
max(1, 5, 3)       # 5
min(1, 5, 3)       # 1
abs(-5)            # 5
ceil(1.2)          # 2
floor(1.8)         # 1
parseint("FF", 16) # 255
log(8, 2)          # 3.0
pow(2, 8)          # 256.0
```

**Encoding functions:**

```hcl
base64encode("hello")          # "aGVsbG8="
base64decode("aGVsbG8=")       # "hello"
jsonencode({key = "value"})    # "{\"key\":\"value\"}"
jsondecode("{\"key\":\"value\"}")  # {key = "value"}
yamlencode({key = "value"})    # "key: value\n"
urlencode("hello world")       # "hello+world"
```

**Filesystem functions (evaluated at plan time, not at resource creation time):**

```hcl
# Read a file's content as a string
user_data = file("./scripts/init.sh")

# Read and render a template file
user_data = templatefile("./templates/user_data.tpl", {
  hostname = var.hostname
  region   = var.aws_region
})

# Get the directory of the current module
path.module    # directory of the current .tf file
path.root      # directory of the root module
path.cwd       # current working directory

# Examples
policy = file("${path.module}/policies/s3_read.json")
```

---

## 3.7 Conditional Expressions

The conditional expression follows the same ternary pattern used in most programming languages.

```hcl
# Syntax: condition ? value_if_true : value_if_false

variable "environment" {
  type = string
}

locals {
  # Choose instance type based on environment
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

  # Enable multi-AZ only in prod
  multi_az = var.environment == "prod" ? true : false

  # Shorter form for bool conditionals
  multi_az_short = var.environment == "prod"

  # Nested conditionals (readability suffers — use locals to break them up)
  db_instance_class = (
    var.environment == "prod"    ? "db.r6g.xlarge" :
    var.environment == "staging" ? "db.t3.medium"  :
    "db.t3.micro"
  )
}

resource "aws_db_instance" "main" {
  instance_class          = local.instance_type
  multi_az                = local.multi_az
  backup_retention_period = var.environment == "prod" ? 7 : 1
  deletion_protection     = var.environment == "prod" ? true : false
}

# Conditional resource creation (count = 0 means "don't create")
variable "create_bastion" {
  type    = bool
  default = false
}

resource "aws_instance" "bastion" {
  count         = var.create_bastion ? 1 : 0
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

---

## 3.8 Terraform Console — Interactive Playground

`terraform console` starts a REPL where you can evaluate HCL expressions interactively. Use it to test functions and expressions before embedding them in configurations. It has access to your current state, so you can query real resource attributes.

```bash
terraform console

> 2 + 2
4

> "hello ${var.environment}"
"hello production"

> length(["a", "b", "c"])
3

> join(",", ["us-east-1a", "us-east-1b"])
"us-east-1a,us-east-1b"

> upper("terraform")
"TERRAFORM"

> merge({a=1, b=2}, {b=3, c=4})
{
  "a" = 1
  "b" = 3
  "c" = 4
}

> [for s in ["hello", "world"] : upper(s)]
[
  "HELLO",
  "WORLD",
]

> var.environment == "prod" ? "db.r6g.xlarge" : "db.t3.micro"
"db.t3.micro"

> Ctrl+C (or Ctrl+D) to exit
```

Whenever you're unsure what a function returns or how an expression evaluates, test it in `terraform console` before using it in your configuration.

---

## 3.9 Formatting and Validation

Always run `fmt` and `validate` before committing or running a plan.

```bash
# terraform fmt — auto-format all .tf files in the current directory
# Equivalent to gofmt for Go code — eliminates formatting debates
terraform fmt

# Format recursively (all subdirectories)
terraform fmt -recursive

# Check-only mode (exit 1 if any file needs formatting — use in CI)
terraform fmt -check
terraform fmt -check -recursive

# Show which files would be changed
terraform fmt -list=true -write=false

# terraform validate — check syntax and internal consistency
# Does NOT contact the cloud API — purely static analysis
terraform validate

# Success output:
# Success! The configuration is valid.

# Error example:
# Error: Reference to undeclared resource
#   on main.tf line 14, in resource "aws_instance" "web":
#   14:   subnet_id = aws_subnet.nonexistent.id
```

**CI pipeline snippet:**

```yaml
# .github/workflows/terraform.yml
- name: Terraform Format Check
  run: terraform fmt -check -recursive

- name: Terraform Validate
  run: terraform validate

- name: Terraform Plan
  run: terraform plan -out=tfplan
```

Both `fmt -check` and `validate` should run in CI before `plan`. They catch mistakes early and enforce consistent style across the team.

---

## Summary

HCL is a small language with a predictable structure. Every configuration is composed of blocks, each of which has a type, optional labels, and arguments. References always follow a `type.name.attribute` pattern. Type constraints make interfaces explicit and catch mistakes early. Built-in functions handle string manipulation, collection transformation, and encoding without needing external tools. Conditional expressions keep configurations DRY across environments.

The two habits to build now: run `terraform fmt` before every commit, and use `terraform console` whenever you're unsure how an expression evaluates.

---

## Knowledge Check

1. What is the difference between a `resource` block and a `data` block?
2. Write a reference to the `id` attribute of a resource `aws_vpc` named `main`.
3. What type constraint would you use for a variable that accepts a list of port numbers?
4. What does `merge({a=1, b=2}, {b=5, c=3})` return?
5. You have a variable `environment` that can be `"dev"`, `"staging"`, or `"prod"`. Write a local value that sets `backup_days` to `30` for prod, `7` for staging, and `1` for dev.

---

## Hands-On Exercise

Write a Terraform configuration that demonstrates at least 5 different HCL features:

1. A `locals` block that computes a `name_prefix` using string interpolation
2. A conditional expression that chooses `instance_type` based on an `environment` variable
3. Two different variable types: one `string` with a validation, one `list(string)`
4. A `data` source (e.g., `aws_availability_zones` to look up available AZs)
5. An output that uses a built-in function (e.g., `upper()` or `join()`)

After writing the configuration, run `terraform console` to test each expression interactively, and run `terraform fmt` and `terraform validate` to check your work.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./02-installation-and-setup.md">← Previous: Installation, Setup & First Resource</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./04-providers-and-resources.md">Next: Providers & Resources →</a>
</div>
