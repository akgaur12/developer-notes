# Chapter 13 — Advanced HCL: Loops, Conditionals & Dynamic Blocks

## Learning Objectives

By the end of this chapter you will be able to:

- Use `count` and `for_each` to create multiple resources without repetition
- Write `for` expressions to transform lists and maps
- Generate dynamic nested blocks from variables using `dynamic`
- Apply `flatten` and `setproduct` to work with complex collection shapes
- Validate infrastructure with `precondition` and `postcondition` lifecycle blocks
- Render templates with `templatefile` and heredoc strings
- Refactor resources safely using the `moved` block

---

## 13.1 count — Create Multiple Similar Resources

```hcl
# Create 3 EC2 instances
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.al2.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"   # web-0, web-1, web-2
  }
}

# Reference by index
output "web_ips" {
  value = aws_instance.web[*].public_ip   # splat expression
}

# Conditional creation: create only in prod
resource "aws_db_instance" "read_replica" {
  count = var.environment == "prod" ? 1 : 0
  # ...
}

# Access: aws_db_instance.read_replica[0].endpoint
# Or check: length(aws_db_instance.read_replica) > 0
```

**count limitation:** resources are indexed by position. If you remove the middle item, Terraform destroys and recreates all subsequent items. Use `for_each` instead when the resource has a meaningful identifier.

---

## 13.2 for_each — Create Resources from a Map or Set

```hcl
# From a map — each.key and each.value are available
variable "subnets" {
  type = map(object({
    cidr_block = string
    public     = bool
  }))
  default = {
    "public-1a"  = { cidr_block = "10.0.1.0/24", public = true }
    "public-1b"  = { cidr_block = "10.0.3.0/24", public = true }
    "private-1a" = { cidr_block = "10.0.2.0/24", public = false }
    "private-1b" = { cidr_block = "10.0.4.0/24", public = false }
  }
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  map_public_ip_on_launch = each.value.public

  tags = {
    Name   = each.key
    Public = each.value.public
  }
}

# Reference: aws_subnet.this["public-1a"].id
# All IDs: [for k, v in aws_subnet.this : v.id]
# Public IDs only: [for k, v in aws_subnet.this : v.id if v.public]

# From a set of strings
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.value
}
```

---

## 13.3 for Expressions — Transform Collections

```hcl
# List comprehension
locals {
  instance_ids = [for instance in aws_instance.web : instance.id]
  upper_names  = [for name in var.names : upper(name)]

  # With condition
  running_ids = [for i in aws_instance.web : i.id if i.instance_state == "running"]
}

# Map comprehension
locals {
  # Transform a list to a map
  instance_map = {for i, instance in aws_instance.web : "web-${i}" => instance.id}

  # Invert a map
  inverted = {for k, v in var.tags : v => k}

  # Build a map from resource for_each
  subnet_ids = {for k, v in aws_subnet.this : k => v.id}
  # Result: { "public-1a" = "subnet-xxx", "private-1a" = "subnet-yyy", ... }
}
```

---

## 13.4 dynamic Blocks — Conditional/Multiple Nested Blocks

Without `dynamic`, making nested blocks configurable means duplicating block definitions or hardcoding values. The `dynamic` meta-argument solves this:

```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  # Generate ingress blocks from a variable
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
}

# Usage:
# ingress_rules = [
#   { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"], description = "HTTPS" },
#   { port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"], description = "HTTP" }
# ]
```

More `dynamic` block examples:

```hcl
# Conditional nested block — include only when variable is set
resource "aws_cloudwatch_log_group" "app" {
  name = "/app/logs"

  dynamic "retention_policy" {
    for_each = var.log_retention_days != null ? [1] : []
    content {
      retention_in_days = var.log_retention_days
    }
  }
}

# RDS parameter group with dynamic parameters
resource "aws_db_parameter_group" "postgres" {
  family = "postgres15"
  name   = "custom-pg15"

  dynamic "parameter" {
    for_each = var.db_parameters
    content {
      name  = parameter.key
      value = parameter.value
    }
  }
}

# var.db_parameters = { "log_connections" = "1", "max_connections" = "200" }
```

---

## 13.5 flatten and setproduct for Complex Collections

```hcl
# Create a resource for every combination of two lists
locals {
  environments = ["dev", "staging", "prod"]
  regions      = ["us-east-1", "eu-west-1"]

  # All combinations
  env_region_pairs = setproduct(local.environments, local.regions)
  # [[dev, us-east-1], [dev, eu-west-1], [staging, us-east-1], ...]
}

# Flatten nested structures
variable "users_by_team" {
  type = map(list(string))
  default = {
    platform = ["alice", "bob"]
    backend  = ["carol", "dave"]
  }
}

locals {
  all_users = flatten([
    for team, users in var.users_by_team : [
      for user in users : {
        user = user
        team = team
      }
    ]
  ])
}

resource "aws_iam_user" "all" {
  for_each = {for u in local.all_users : u.user => u}
  name     = each.value.user
  tags     = { Team = each.value.team }
}
```

`flatten` collapses a list-of-lists into a single list. Combined with `for_each`, it lets you drive resource creation from nested data structures without pre-processing outside of Terraform.

---

## 13.6 Preconditions and Postconditions

Introduced in Terraform 1.2, lifecycle `precondition` and `postcondition` blocks let you assert requirements about your infrastructure at plan and apply time — catching misconfigurations before they cause outages.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "The selected AMI must be x86_64 architecture."
    }

    postcondition {
      condition     = self.instance_state == "running"
      error_message = "Instance did not reach running state."
    }
  }
}

# Output with precondition
output "api_endpoint" {
  value = "https://${aws_lb.main.dns_name}/api"

  precondition {
    condition     = aws_lb.main.state == "active"
    error_message = "ALB must be in active state to output the endpoint."
  }
}
```

| Block | When it runs | What `self` refers to |
|---|---|---|
| `precondition` | Before create/update | The planned new values |
| `postcondition` | After create/update | The actual applied values |

---

## 13.7 templatefile and heredoc

Use `templatefile` when your user data, config files, or scripts are complex enough to deserve their own file:

```hcl
locals {
  user_data = templatefile("${path.module}/templates/user_data.tpl", {
    environment   = var.environment
    app_version   = var.app_version
    db_secret_arn = aws_secretsmanager_secret.db.arn
    region        = data.aws_region.current.name
  })
}
```

`templates/user_data.tpl`:

```
#!/bin/bash
set -e
export ENVIRONMENT="${environment}"
export APP_VERSION="${app_version}"
export DB_SECRET="${db_secret_arn}"
export AWS_REGION="${region}"

%{ for cmd in startup_commands ~}
${cmd}
%{ endfor ~}
```

Template directives use `%{ }` syntax and support `if`, `else`, `endif`, `for`, and `endfor`. The `~` strip marker trims surrounding whitespace to keep rendered output clean.

For short inline strings, use a heredoc in HCL directly:

```hcl
locals {
  policy_json = <<-EOT
    {
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Action": "s3:GetObject",
        "Resource": "${aws_s3_bucket.assets.arn}/*"
      }]
    }
  EOT
}
```

---

## 13.8 moved Block — Refactor Without Destroy

Renaming a resource in your config without a `moved` block causes Terraform to destroy the old resource and create a new one — potential downtime for stateful resources.

```hcl
# Renamed a resource in config (aws_instance.app → aws_instance.web_server)
moved {
  from = aws_instance.app
  to   = aws_instance.web_server
}

# Moving a resource into a module
moved {
  from = aws_instance.web
  to   = module.app.aws_instance.web
}

# Changing from count to for_each
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["web-server-1"]
}
```

The `moved` block instructs Terraform to update the state entry without touching the real resource. After running `terraform apply` successfully, delete the `moved` block — it is a one-time migration aid, not a permanent declaration.

---

## Summary

| Feature | Purpose | Key Syntax |
|---|---|---|
| `count` | N identical resources | `count = 3`, `count.index`, `resource[*]` |
| `for_each` | Resources keyed by identifier | `for_each = map/set`, `each.key`, `each.value` |
| `for` expression | Transform collections | `[for x in list : x.id]`, `{for k, v in map : k => v}` |
| `dynamic` | Configurable nested blocks | `dynamic "block" { for_each = ... content { } }` |
| `flatten` / `setproduct` | Complex collection shapes | Flatten nested lists, cartesian product |
| `precondition` / `postcondition` | Inline assertions | `lifecycle { precondition { condition = ... } }` |
| `templatefile` | External template rendering | `templatefile("path", { var = val })` |
| `moved` | Safe refactoring | `moved { from = old_addr  to = new_addr }` |

---

## Knowledge Check

1. You have `count = 3` creating three EC2 instances. You remove the instance at index 1. What does Terraform do, and why is `for_each` a better choice here?
2. Write a `for` expression that produces a map of `{ name => id }` from a list of objects where each object has `name` and `id` attributes.
3. What is the difference between a `dynamic` block and a regular repeated nested block?
4. When would you use `postcondition` rather than `precondition`?
5. You rename `aws_s3_bucket.data` to `aws_s3_bucket.raw_data` in your config. Without a `moved` block, what happens? How do you prevent the bucket from being recreated?

---

## Hands-on Exercise

**Goal:** Refactor a static Terraform configuration to use advanced HCL features.

**Starting point** — you have a security group with three hardcoded `ingress` blocks and four `aws_subnet` resources declared individually.

**Tasks:**

1. **Dynamic ingress rules** — replace the hardcoded `ingress` blocks with a `dynamic "ingress"` block driven by a `var.ingress_rules` list of objects. Test that `terraform plan` shows no changes when you supply the same values as the hardcoded rules.

2. **Subnets via for_each** — replace the four individual `aws_subnet` resources with a single `aws_subnet.this` resource using `for_each` over a map variable. Confirm all four subnets still appear in the plan.

3. **Safe rename** — rename `aws_subnet.this` to `aws_subnet.vpc_subnet` in your config. Add a `moved` block, run `terraform plan`, verify Terraform shows zero destroys, then remove the `moved` block and apply.

4. **Bonus** — add a `precondition` to one of your subnet resources that asserts the CIDR block falls within `10.0.0.0/8`.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./12-aws-databases.md">← Previous: Building AWS Databases with Terraform</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./14-cicd-integration.md">Next: CI/CD Integration & Atlantis →</a>
</div>
