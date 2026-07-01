# Chapter 4 — Providers & Resources

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what Terraform providers are and how they work
- Configure providers, including multiple aliases for multi-region and multi-account setups
- Understand the anatomy of a resource block and all its components
- Distinguish between implicit and explicit resource dependencies
- Apply lifecycle rules to control how Terraform creates, updates, and destroys resources
- Use the `random` provider for unique naming and secret generation
- Reference resource attributes to wire resources together

---

## 4.1 What Are Providers?

Providers are plugins that allow Terraform to interact with external APIs. Each provider knows how to create, read, update, and delete specific resource types by translating your HCL configuration into API calls.

**Provider categories:**

| Category | Maintained by | Examples |
|---|---|---|
| Official | HashiCorp | `aws`, `google`, `azurerm`, `kubernetes`, `helm` |
| Partner | The vendor | `datadog`, `pagerduty`, `github`, `cloudflare` |
| Community | Individuals/orgs | Various open-source contributors |

All providers are published at [registry.terraform.io](https://registry.terraform.io).

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # ~> = compatible version constraint (>= 5.0, < 6.0)
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }
}
```

**Version constraints quick reference:**

| Constraint | Meaning |
|---|---|
| `= 5.1.0` | Exactly 5.1.0 |
| `!= 5.1.0` | Anything except 5.1.0 |
| `>= 5.0` | 5.0 or newer |
| `~> 5.0` | >= 5.0, < 6.0 (patch and minor upgrades) |
| `~> 5.1` | >= 5.1, < 5.2 (patch upgrades only) |

Always pin providers in production. Unpinned providers can silently introduce breaking changes.

---

## 4.2 Provider Configuration

```hcl
# Multiple provider configurations (aliases)
provider "aws" {
  region = "us-east-1"
  alias  = "us_east"
}

provider "aws" {
  region = "eu-west-1"
  alias  = "eu_west"
}

# Use specific provider alias on a resource
resource "aws_s3_bucket" "eu_bucket" {
  provider = aws.eu_west
  bucket   = "my-eu-bucket"
}

# Cross-account provider via IAM role assumption
provider "aws" {
  alias = "prod_account"
  assume_role {
    role_arn = "arn:aws:iam::999888777:role/terraform-admin"
  }
}
```

Provider credentials should come from environment variables or IAM instance profiles — never hardcode them in `.tf` files. The AWS provider reads `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_REGION` automatically.

---

## 4.3 Resource Block Deep Dive

```hcl
# Full resource block anatomy
resource "aws_instance" "web" {    # type = "aws_instance", name = "web"
  # Required arguments
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Optional arguments
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  iam_instance_profile        = aws_iam_instance_profile.web.name
  associate_public_ip_address = true
  key_name                    = var.key_pair_name

  # Nested block (not an argument, a block within a block)
  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true
  }

  user_data = templatefile("${path.module}/user_data.tpl", {
    hostname    = "web-server"
    environment = var.environment
  })

  tags = merge(local.common_tags, {
    Name = "web-server"
    Role = "web"
  })

  # Lifecycle rules (covered in 4.5)
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [user_data, ami]
  }
}
```

**Resource address format:** `resource_type.resource_name`
For the example above: `aws_instance.web`

---

## 4.4 Resource Dependencies

Terraform builds a dependency graph to determine the correct order of operations and to parallelise where possible.

```hcl
# Implicit dependency: Terraform auto-detects by reference
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id    # implicit dependency on aws_vpc.main
  cidr_block = "10.0.1.0/24"
}

# Explicit dependency: use depends_on when there is no attribute reference
resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.lambda.name
  policy_arn = aws_iam_policy.lambda_policy.arn
}

resource "aws_lambda_function" "processor" {
  function_name = "processor"
  role          = aws_iam_role.lambda.arn

  # Lambda needs the policy attached before it can run,
  # but there is no attribute reference to create an implicit dep.
  depends_on = [aws_iam_role_policy_attachment.attach]
}
```

Terraform builds a dependency graph and applies resources in parallel where possible. `depends_on` forces sequential execution. Over-using `depends_on` reduces parallelism — only use it when you genuinely need it.

---

## 4.5 Lifecycle Rules

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.medium"

  lifecycle {
    # Create the new instance before destroying the old one (zero-downtime replacement)
    create_before_destroy = true

    # Never destroy this resource — terraform destroy or resource removal will error
    prevent_destroy = true

    # Don't update these attributes even if they change in config or remote state.
    # Useful for: AMI (avoids recreating instance every time the AMI is updated),
    # user_data (avoid re-running init scripts after first boot).
    ignore_changes = [ami, user_data, tags["LastDeployedAt"]]

    # Postcondition: run a validation check after resource is created/updated (TF 1.4+)
    # postcondition {
    #   condition     = self.public_ip != ""
    #   error_message = "Instance must have a public IP"
    # }
  }
}
```

**Lifecycle rule summary:**

| Rule | Effect |
|---|---|
| `create_before_destroy = true` | New resource is provisioned before old one is removed |
| `prevent_destroy = true` | `terraform destroy` or config removal errors instead of deleting |
| `ignore_changes = [attr]` | Drift in listed attributes is ignored and not corrected |

---

## 4.6 The random Provider

```hcl
# Generate unique suffixes to avoid name collisions across accounts/regions
resource "random_id" "suffix" {
  byte_length = 4
}

resource "random_password" "db_password" {
  length           = 20
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_s3_bucket" "data" {
  bucket = "myapp-data-${random_id.suffix.hex}"  # e.g., "myapp-data-a1b2c3d4"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "dbadmin"
    password = random_password.db_password.result
  })
}
```

`random_id` and `random_password` are generated once and stored in state. They will not change on subsequent `terraform apply` runs unless the resource is replaced.

---

## 4.7 The null_resource and terraform_data

```hcl
# null_resource: run provisioners without a real cloud resource
# terraform_data is the modern replacement in TF 1.4+
resource "terraform_data" "run_migrations" {
  triggers_replace = [aws_db_instance.main.id]   # re-run if DB is replaced

  provisioner "local-exec" {
    command = "python manage.py migrate --settings=prod"
    environment = {
      DATABASE_URL = "postgresql://${var.db_user}:${var.db_pass}@${aws_db_instance.main.address}/mydb"
    }
  }
}
```

Avoid provisioners when possible — they are procedural and can fail in non-idempotent ways. Prefer `user_data` for EC2 bootstrapping, or bake AMIs with Packer.

---

## 4.8 Referencing Resource Attributes

```hcl
# Every resource exposes attributes after it is created.
# Format: resource_type.resource_name.attribute_name

aws_vpc.main.id                     # the VPC ID
aws_instance.web.public_ip          # public IP (known after apply)
aws_s3_bucket.data.arn              # S3 bucket ARN
aws_iam_role.lambda.name            # IAM role name
aws_db_instance.main.endpoint       # RDS endpoint (host:port)
aws_db_instance.main.address        # RDS hostname only
```

Some attributes are only known after apply — Terraform shows `(known after apply)` in the plan. You can still reference them; Terraform resolves them during apply.

---

## 4.9 Common AWS Provider Resources Reference

```hcl
# Networking
aws_vpc, aws_subnet, aws_internet_gateway, aws_nat_gateway
aws_route_table, aws_route_table_association, aws_route
aws_security_group, aws_security_group_rule
aws_vpc_endpoint

# Compute
aws_instance, aws_launch_template, aws_autoscaling_group
aws_ami, aws_key_pair, aws_placement_group

# Load Balancing
aws_lb, aws_lb_listener, aws_lb_target_group, aws_lb_listener_rule

# Storage
aws_s3_bucket, aws_s3_bucket_versioning, aws_s3_bucket_policy
aws_ebs_volume, aws_volume_attachment

# Database
aws_db_instance, aws_db_subnet_group, aws_db_parameter_group
aws_elasticache_cluster, aws_elasticache_subnet_group

# IAM
aws_iam_role, aws_iam_policy, aws_iam_role_policy_attachment
aws_iam_user, aws_iam_group, aws_iam_instance_profile

# DNS & CDN
aws_route53_zone, aws_route53_record
aws_cloudfront_distribution, aws_acm_certificate

# Serverless
aws_lambda_function, aws_lambda_event_source_mapping
aws_api_gatewayv2_api, aws_api_gatewayv2_route
```

---

## Summary

- Providers are plugins that translate HCL into API calls. They must be declared in `required_providers` with a pinned version.
- Multiple provider aliases allow multi-region and multi-account deployments from a single configuration.
- Resource blocks have required arguments, optional arguments, nested blocks, and a `lifecycle` meta-argument.
- Terraform automatically detects implicit dependencies via attribute references and builds an optimised parallel execution graph.
- Use `depends_on` only when there is no attribute reference to express the dependency.
- `create_before_destroy`, `prevent_destroy`, and `ignore_changes` give you fine-grained control over resource lifecycle events.
- The `random` provider generates stable, unique values stored in state.
- Provisioners (`terraform_data`, `local-exec`) are a last resort — prefer declarative approaches.

---

## Knowledge Check

1. What is the difference between an official provider and a partner provider?
2. Why should you always pin provider versions in production?
3. What is the difference between an implicit and explicit dependency? Give an example of when `depends_on` is necessary.
4. What does `create_before_destroy = true` do, and when would you use it?
5. A colleague hardcoded an AMI ID in a resource block. How would you use `ignore_changes` to prevent Terraform from replacing the instance every time a new AMI is released?

---

## Hands-on Exercise

Create a Terraform configuration that provisions at least five AWS resources demonstrating:

- Both implicit and explicit dependencies (use `depends_on` at least once)
- A `lifecycle` block with `create_before_destroy = true` on the most critical resource
- The `random` provider to ensure globally unique bucket and resource names
- At least one nested block (e.g., `root_block_device`, `ingress`)
- Tags applied using `merge()` with a shared locals map

Run `terraform plan` and verify the dependency graph ordering matches your expectations.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="03-hcl-fundamentals.md">← Previous: HCL Fundamentals</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="05-variables-outputs-data.md">Next: Variables, Outputs & Data Sources →</a>
</div>
