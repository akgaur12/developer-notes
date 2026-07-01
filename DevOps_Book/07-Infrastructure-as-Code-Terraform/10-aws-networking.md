# Chapter 10 — Building AWS Networking with Terraform

## Learning Objectives

By the end of this chapter you will be able to:

- Translate manually-built AWS networking infrastructure into Terraform HCL
- Write a complete, production-grade VPC configuration including subnets, route tables, IGW, and NAT Gateway
- Define layered security groups using security group ID references (not CIDRs)
- Provision ACM certificates with Route53 DNS validation using `for_each`
- Expose all networking resources as module outputs for consumption by compute and database modules

---

## 10.1 Converting Manual AWS to Terraform

In Topic 6 you built AWS infrastructure by clicking through the Console. Now you'll declare the exact same infrastructure in Terraform. The mental shift is straightforward: stop thinking "what do I click?" and start thinking "what do I declare?"

Every resource you created manually maps 1:1 to a Terraform resource block. The difference is that Terraform remembers what it created, can recreate it identically in another region, and can tear it all down with a single `terraform destroy`.

**The workflow:**

```
Console (click) → HCL (declare) → terraform apply (create)
```

Key insight: Terraform resolves dependencies automatically. You don't need to worry about "create the VPC first, then the subnets." You just declare what you want, reference resources by their Terraform addresses, and Terraform figures out the order.

---

## 10.2 Complete VPC Configuration

The following is a full production VPC with public and private subnets across two availability zones.

### variables.tf

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
}

variable "project" {
  description = "Project name used in resource naming and tags"
  type        = string
}

variable "azs" {
  description = "List of availability zones to deploy into"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}
```

### locals.tf

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### main.tf — VPC and Subnets

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# Public subnets — one per AZ
resource "aws_subnet" "public" {
  count = length(var.azs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${var.azs[count.index]}"
    Tier = "public"
    # Required for EKS ALB Ingress Controller (Topic 8)
    "kubernetes.io/role/elb" = "1"
  })
}

# Private subnets — one per AZ
resource "aws_subnet" "private" {
  count = length(var.azs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${var.azs[count.index]}"
    Tier = "private"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

# Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-eip"
  })

  depends_on = [aws_internet_gateway.main]
}

# NAT Gateway — placed in the first public subnet
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat"
  })

  depends_on = [aws_internet_gateway.main]
}

# Public route table — routes 0.0.0.0/0 to IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route table — routes 0.0.0.0/0 to NAT Gateway
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-rt"
  })
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

### Bonus: VPC Endpoint for S3

```hcl
# Gateway endpoint — free, keeps S3 traffic off the internet/NAT
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${data.aws_region.current.name}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-s3-endpoint"
  })
}

data "aws_region" "current" {}
```

---

## 10.3 Security Groups

Security groups in AWS act as stateful firewalls. The key design principle: reference security group IDs in rules instead of CIDR blocks wherever possible. When the upstream SG changes, the downstream rule automatically tracks it.

```hcl
# ALB security group — accepts HTTP and HTTPS from anywhere
resource "aws_security_group" "alb" {
  name        = "${local.name_prefix}-alb"
  description = "ALB inbound HTTPS"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-alb-sg" })
}

# App security group — only accepts traffic from ALB SG
resource "aws_security_group" "app" {
  name        = "${local.name_prefix}-app"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # reference by SG ID, not CIDR
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-app-sg" })
}

# DB security group — only accepts from app SG
resource "aws_security_group" "db" {
  name   = "${local.name_prefix}-db"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-db-sg" })
}
```

The security group chain `ALB → App → DB` means:

- The internet can only reach the ALB on ports 80/443
- The ALB can only reach app instances on port 8080
- App instances can only reach the database on port 5432
- Nothing can reach the database directly from outside

---

## 10.4 Route53 and ACM

### Hosted Zone (data source for an existing zone)

```hcl
# Hosted zone (existing zone — use data source)
data "aws_route53_zone" "main" {
  name         = var.domain_name
  private_zone = false
}

# ACM certificate with DNS validation
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = ["*.${var.domain_name}"]
  validation_method         = "DNS"
  lifecycle {
    create_before_destroy = true
  }
}

# DNS validation records
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# ALB DNS alias record
resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"
  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}
```

The `for_each` on `domain_validation_options` is a standard pattern. ACM returns one validation record per domain in the SAN list; the `for_each` creates a Route53 record for each one. `aws_acm_certificate_validation` then waits (up to 45 minutes) for ACM to confirm the DNS records exist.

---

## 10.5 VPC Outputs

All outputs that downstream modules (compute, databases) will consume:

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

output "alb_security_group_id" {
  description = "ID of the ALB security group"
  value       = aws_security_group.alb.id
}

output "app_security_group_id" {
  description = "ID of the application security group"
  value       = aws_security_group.app.id
}

output "db_security_group_id" {
  description = "ID of the database security group"
  value       = aws_security_group.db.id
}

output "nat_gateway_id" {
  description = "ID of the NAT Gateway"
  value       = aws_nat_gateway.main.id
}

output "certificate_arn" {
  description = "ARN of the validated ACM certificate"
  value       = aws_acm_certificate_validation.main.certificate_arn
}
```

---

## 10.6 Common Networking Patterns

### Using for_each for Subnets (More Stable Than count)

The `count`-based approach used above is simple, but `for_each` is more robust when you need to add or remove specific subnets without disrupting others:

```hcl
variable "public_subnets" {
  description = "Map of AZ to CIDR block for public subnets"
  type        = map(string)
  default = {
    "us-east-1a" = "10.0.0.0/24"
    "us-east-1b" = "10.0.1.0/24"
  }
}

resource "aws_subnet" "public" {
  for_each = var.public_subnets

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value
  availability_zone       = each.key
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${each.key}"
  })
}
```

With `count`, removing the first AZ shifts all indices and triggers a replacement of every subnet. With `for_each`, each subnet is addressed by its AZ key — removing one AZ only destroys that subnet.

### VPC Peering (Dev to Prod)

```hcl
# Requester — from dev VPC
resource "aws_vpc_peering_connection" "dev_to_prod" {
  vpc_id        = aws_vpc.dev.id
  peer_vpc_id   = aws_vpc.prod.id
  peer_region   = "us-east-1"
  auto_accept   = false

  tags = { Name = "dev-to-prod" }
}

# Accepter — must be explicitly accepted (usually in a separate AWS account/provider)
resource "aws_vpc_peering_connection_accepter" "dev_to_prod" {
  provider                  = aws.prod
  vpc_peering_connection_id = aws_vpc_peering_connection.dev_to_prod.id
  auto_accept               = true
}
```

### S3 VPC Endpoint

Already shown in section 10.2 — this gateway endpoint is free and routes all S3 traffic over the AWS backbone, saving NAT Gateway data processing fees ($0.045/GB).

---

## 10.7 Networking Best Practices

```
CIDR Planning
  Use /16 for VPC CIDRs — 65,536 IPs, plenty of room to grow
  Use /24 for subnets — 256 IPs each, easy to reason about
  Leave CIDR space for future subnets (e.g., 10.0.20.x for a new tier)

High Availability
  One NAT Gateway per AZ for HA ($0.045/hr × AZ count — ~$32/month each)
  In dev: one NAT Gateway is fine to save cost

Security Groups
  Reference SG IDs not CIDRs in SG rules — auto-updates when SG changes
  Keep egress open (0.0.0.0/0) unless you have strict compliance requirements
  Enable VPC Flow Logs for security auditing and incident response

Performance
  Use VPC endpoints for S3, DynamoDB, ECR — keep traffic off internet
  Use interface endpoints for Secrets Manager, SSM, CloudWatch in private subnets

EKS Compatibility
  Tag public subnets: kubernetes.io/role/elb = 1
  Tag private subnets: kubernetes.io/role/internal-elb = 1
  Required for EKS to auto-discover subnets for load balancers (Topic 8)
```

---

## Summary

- You declare the same AWS networking components in HCL that you clicked through in the Console in Topic 6
- A production VPC needs: VPC, IGW, public subnets, private subnets, EIP, NAT Gateway, and two route tables
- Security groups chain using SG ID references: ALB → App → DB
- ACM certificate DNS validation uses `for_each` over `domain_validation_options`
- Module outputs expose all IDs/ARNs that downstream modules need — they should never hard-code resource IDs

---

## Knowledge Check

1. What is the difference between `count` and `for_each` for subnet creation, and when would you prefer `for_each`?
2. Why do you reference `aws_security_group.alb.id` in the app security group's ingress rule instead of a CIDR block?
3. What does `aws_acm_certificate_validation` actually do — is it creating a resource in AWS?
4. Why does the NAT Gateway have `depends_on = [aws_internet_gateway.main]`?
5. What is a VPC Gateway Endpoint for S3, and why does it save money?

---

## Hands-on Exercise

Write a complete Terraform VPC configuration with the following:

- A VPC with CIDR `10.0.0.0/16`, DNS support and DNS hostnames enabled
- 2 public subnets across 2 AZs (`10.0.0.0/24`, `10.0.1.0/24`) with `map_public_ip_on_launch = true`
- 2 private subnets across the same 2 AZs (`10.0.10.0/24`, `10.0.11.0/24`)
- An Internet Gateway attached to the VPC
- An Elastic IP and NAT Gateway in the first public subnet
- A public route table routing `0.0.0.0/0` to the IGW, associated with both public subnets
- A private route table routing `0.0.0.0/0` to the NAT Gateway, associated with both private subnets
- Three security groups: ALB (HTTP/HTTPS from internet), App (port 8080 from ALB SG only), DB (port 5432 from App SG only)
- An ACM certificate for a domain of your choice with DNS validation records in Route53
- Outputs for all IDs and ARNs that a compute module would need

Use `locals` for `name_prefix` and `common_tags`. Use variables for `vpc_cidr`, `environment`, `project`, `azs`, and `domain_name`.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="09-workspaces-and-environments.md">← Previous: Workspaces & Environments</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="11-aws-compute.md">Next: Building AWS Compute with Terraform →</a>
</div>
