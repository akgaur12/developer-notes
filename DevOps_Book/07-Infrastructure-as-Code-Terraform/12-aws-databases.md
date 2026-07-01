# Chapter 12 — Building AWS Databases with Terraform

## Learning Objectives

By the end of this chapter you will be able to:

- Provision an RDS PostgreSQL instance with parameter groups, subnet groups, Multi-AZ, and encryption
- Store and rotate database credentials securely using Secrets Manager and `random_password`
- Create ElastiCache Redis clusters with encryption in transit and at rest
- Define DynamoDB tables with TTL, point-in-time recovery, and on-demand billing
- Manage KMS keys for encrypting all database-tier resources
- Expose database connection details as sensitive outputs for use by the compute tier

---

## 12.1 RDS PostgreSQL

RDS is the most configuration-dense AWS resource in Terraform. The resource has dozens of arguments — the ones below represent a production baseline.

```hcl
resource "aws_db_subnet_group" "main" {
  name        = "${local.name_prefix}-db-subnet-group"
  subnet_ids  = var.private_subnet_ids
  description = "RDS subnet group for ${local.name_prefix}"
  tags        = local.common_tags
}

resource "aws_db_parameter_group" "postgres" {
  family = "postgres15"
  name   = "${local.name_prefix}-pg15"

  parameter {
    name  = "log_connections"
    value = "1"
  }
  parameter {
    name  = "log_min_duration_statement"
    value = "1000"   # log queries taking longer than 1 second
  }
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }

  tags = local.common_tags
}

resource "aws_db_instance" "main" {
  identifier        = "${local.name_prefix}-postgres"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = var.db_instance_class
  allocated_storage = var.db_storage_gb
  storage_type      = "gp3"
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  db_name  = var.db_name
  username = var.db_username
  password = random_password.db.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [var.db_security_group_id]
  parameter_group_name   = aws_db_parameter_group.postgres.name

  multi_az               = var.environment == "prod"
  publicly_accessible    = false
  deletion_protection    = var.environment == "prod"
  skip_final_snapshot    = var.environment != "prod"
  final_snapshot_identifier = var.environment == "prod" ? "${local.name_prefix}-final-snapshot" : null

  backup_retention_period = var.environment == "prod" ? 7 : 1
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  performance_insights_enabled          = var.environment == "prod"
  performance_insights_retention_period = var.environment == "prod" ? 7 : null

  enabled_cloudwatch_logs_exports = ["postgresql"]

  tags = local.common_tags

  lifecycle {
    prevent_destroy = tobool(var.environment == "prod")
    ignore_changes  = [password]   # managed externally after first create
  }
}
```

**Key decisions explained:**

| Argument | Dev | Prod | Why |
|---|---|---|---|
| `multi_az` | false | true | Standby in second AZ, automatic failover in ~60s |
| `skip_final_snapshot` | true | false | Prod keeps a final snapshot before deletion |
| `deletion_protection` | false | true | Prevents `terraform destroy` from deleting prod DB |
| `backup_retention_period` | 1 day | 7 days | Longer window to recover from data corruption |
| `performance_insights_enabled` | false | true | Query-level performance analysis |

### Generating and Storing the Password

```hcl
resource "random_password" "db" {
  length           = 24
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

# Store password in Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "${local.name_prefix}/db/password"
  description             = "RDS master password for ${local.name_prefix}"
  recovery_window_in_days = var.environment == "prod" ? 30 : 0
  kms_key_id              = aws_kms_key.secrets.arn
  tags                    = local.common_tags
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = var.db_username
    password = random_password.db.result
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = var.db_name
  })
}
```

The secret is stored as a JSON object so the application can parse `host`, `port`, `username`, `password`, and `dbname` from a single `GetSecretValue` API call using a standard connection string builder.

**Why `ignore_changes = [password]`?**

On first apply, Terraform sets the RDS master password to `random_password.db.result`. After that, you might rotate the password outside Terraform (via Secrets Manager rotation). `ignore_changes` tells Terraform to stop tracking the `password` field so subsequent applies do not reset it back to the original value.

---

## 12.2 RDS Read Replica

```hcl
resource "aws_db_instance" "read_replica" {
  count = var.environment == "prod" ? 1 : 0

  identifier             = "${local.name_prefix}-postgres-replica"
  replicate_source_db    = aws_db_instance.main.identifier
  instance_class         = var.db_replica_instance_class
  publicly_accessible    = false
  storage_encrypted      = true
  skip_final_snapshot    = true
  deletion_protection    = false
  vpc_security_group_ids = [var.db_security_group_id]

  tags = merge(local.common_tags, { Role = "read-replica" })
}
```

Read replicas use `count = var.environment == "prod" ? 1 : 0` — the conditional pattern for optional resources. In dev there is no replica; in prod there is one. `replicate_source_db` points to the primary instance identifier.

---

## 12.3 ElastiCache Redis

```hcl
resource "aws_elasticache_subnet_group" "main" {
  name       = "${local.name_prefix}-redis-subnet-group"
  subnet_ids = var.private_subnet_ids
  tags       = local.common_tags
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id       = "${local.name_prefix}-redis"
  description                = "Redis cluster for ${local.name_prefix}"
  node_type                  = var.redis_node_type
  port                       = 6379
  num_cache_clusters         = var.environment == "prod" ? 2 : 1
  automatic_failover_enabled = var.environment == "prod"
  multi_az_enabled           = var.environment == "prod"
  subnet_group_name          = aws_elasticache_subnet_group.main.name
  security_group_ids         = [aws_security_group.redis.id]
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  engine_version             = "7.0"

  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis.name
    destination_type = "cloudwatch-logs"
    log_format       = "text"
    log_type         = "slow-log"
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_log_group" "redis" {
  name              = "/elasticache/${local.name_prefix}/redis"
  retention_in_days = 7
  tags              = local.common_tags
}
```

**Important:** `automatic_failover_enabled` requires `num_cache_clusters >= 2`. These must change together — if you set `automatic_failover_enabled = true` with `num_cache_clusters = 1`, the apply will fail.

`transit_encryption_enabled` means connections to Redis must use TLS. Update your application's Redis client to use `rediss://` (with double `s`) as the URL scheme.

---

## 12.4 DynamoDB

DynamoDB is simpler to provision than RDS because there is no concept of a server — you just define the table schema and billing mode.

```hcl
resource "aws_dynamodb_table" "sessions" {
  name         = "${local.name_prefix}-sessions"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "session_id"

  attribute {
    name = "session_id"
    type = "S"
  }

  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  point_in_time_recovery {
    enabled = var.environment == "prod"
  }

  server_side_encryption {
    enabled = true
  }

  tags = local.common_tags
}
```

**DynamoDB billing modes:**

- `PAY_PER_REQUEST` — on-demand. No capacity planning. Pays per read/write unit. Good for unpredictable or low traffic
- `PROVISIONED` — you set read/write capacity units. Cheaper at high predictable scale. Supports auto-scaling

### DynamoDB Auto-scaling (for PROVISIONED mode)

```hcl
resource "aws_appautoscaling_target" "dynamodb_read" {
  count              = var.dynamodb_billing_mode == "PROVISIONED" ? 1 : 0
  max_capacity       = 100
  min_capacity       = 5
  resource_id        = "table/${aws_dynamodb_table.sessions.name}"
  scalable_dimension = "dynamodb:table:ReadCapacityUnits"
  service_namespace  = "dynamodb"
}

resource "aws_appautoscaling_policy" "dynamodb_read" {
  count              = var.dynamodb_billing_mode == "PROVISIONED" ? 1 : 0
  name               = "${local.name_prefix}-dynamodb-read-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.dynamodb_read[0].resource_id
  scalable_dimension = aws_appautoscaling_target.dynamodb_read[0].scalable_dimension
  service_namespace  = aws_appautoscaling_target.dynamodb_read[0].service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBReadCapacityUtilization"
    }
    target_value = 70.0
  }
}
```

---

## 12.5 KMS Keys for Encryption

Every database-tier resource (RDS, ElastiCache, Secrets Manager) should be encrypted with a customer-managed KMS key rather than the AWS default key. This gives you independent audit trails and the ability to revoke access by disabling the key.

```hcl
resource "aws_kms_key" "rds" {
  description             = "${local.name_prefix} RDS encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  tags                    = local.common_tags
}

resource "aws_kms_alias" "rds" {
  name          = "alias/${local.name_prefix}-rds"
  target_key_id = aws_kms_key.rds.key_id
}

resource "aws_kms_key" "secrets" {
  description             = "${local.name_prefix} Secrets Manager encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  tags                    = local.common_tags
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/${local.name_prefix}-secrets"
  target_key_id = aws_kms_key.secrets.key_id
}
```

**Why `enable_key_rotation = true`?** AWS rotates the underlying key material once per year automatically. The key ID and ARN stay the same — no application changes required. This satisfies most compliance requirements (SOC 2, PCI DSS, HIPAA) without any operational burden.

**Why `deletion_window_in_days = 30`?** KMS keys cannot be deleted immediately. The 7-30 day window gives you time to cancel an accidental deletion. Everything encrypted with the key becomes permanently unreadable if the key is deleted.

---

## 12.6 Database Outputs

```hcl
output "rds_endpoint" {
  description = "RDS instance endpoint (host:port)"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

output "rds_address" {
  description = "RDS instance hostname"
  value       = aws_db_instance.main.address
  sensitive   = true
}

output "db_secret_arn" {
  description = "ARN of the Secrets Manager secret containing DB credentials"
  value       = aws_secretsmanager_secret.db_password.arn
}

output "redis_endpoint" {
  description = "Redis primary endpoint address"
  value       = aws_elasticache_replication_group.main.primary_endpoint_address
  sensitive   = true
}

output "redis_port" {
  description = "Redis port"
  value       = aws_elasticache_replication_group.main.port
}

output "dynamodb_table_name" {
  description = "DynamoDB sessions table name"
  value       = aws_dynamodb_table.sessions.name
}

output "rds_kms_key_arn" {
  description = "ARN of the KMS key used to encrypt RDS"
  value       = aws_kms_key.rds.arn
}
```

Mark endpoints `sensitive = true` so they are redacted from `terraform plan` output and cannot be accidentally logged in CI pipelines. They are still accessible via `terraform output -json` when needed.

---

## 12.7 Handling Database Migrations

A common question: where do schema migrations run? You have two options.

### Option 1: Terraform Provisioner (not recommended for production)

```hcl
# Run migrations after RDS is ready
resource "terraform_data" "db_migrations" {
  triggers_replace = [aws_db_instance.main.id]

  provisioner "local-exec" {
    command = "flyway -url=jdbc:postgresql://${aws_db_instance.main.address}/${var.db_name} migrate"
    environment = {
      FLYWAY_PASSWORD = random_password.db.result
      FLYWAY_USER     = var.db_username
    }
  }

  depends_on = [aws_db_instance.main]
}
```

This works, but ties migrations to Terraform runs. If a migration fails, the `terraform apply` fails and Terraform marks the resource as tainted.

### Option 2: CI/CD Pipeline (recommended)

Run migrations as a separate step in your deployment pipeline after `terraform apply` completes:

```yaml
# GitHub Actions example
- name: Apply Terraform
  run: terraform apply -auto-approve

- name: Run DB Migrations
  run: |
    DB_SECRET=$(terraform output -raw db_secret_arn)
    CREDS=$(aws secretsmanager get-secret-value --secret-id $DB_SECRET --query SecretString --output text)
    DB_URL="postgresql://$(echo $CREDS | jq -r .username):$(echo $CREDS | jq -r .password)@$(echo $CREDS | jq -r .host)/$(echo $CREDS | jq -r .dbname)"
    flyway -url=jdbc:$DB_URL migrate
```

This separates concerns: Terraform manages infrastructure, your CI/CD pipeline manages application deployment including migrations.

---

## Summary

- RDS requires four supporting resources: subnet group, parameter group, security group, and KMS key — before the DB instance itself
- Use `random_password` + Secrets Manager to generate and store the initial password securely; use `ignore_changes = [password]` to allow external rotation
- Environment-conditional expressions (`var.environment == "prod" ? x : y`) let one configuration serve both dev and prod with different durability settings
- ElastiCache requires `num_cache_clusters >= 2` when `automatic_failover_enabled = true`
- DynamoDB `PAY_PER_REQUEST` is the safe default; switch to `PROVISIONED` with auto-scaling at high sustained throughput
- KMS customer-managed keys with `enable_key_rotation = true` satisfy most compliance requirements automatically
- Mark all endpoint outputs `sensitive = true` to prevent accidental credential leakage in logs

---

## Knowledge Check

1. What is the purpose of `ignore_changes = [password]` in the `aws_db_instance` resource?
2. Why must `automatic_failover_enabled` and `num_cache_clusters >= 2` be set together in `aws_elasticache_replication_group`?
3. What happens to data encrypted with a KMS key if the key is deleted?
4. What is the difference between `skip_final_snapshot = true` and `deletion_protection = true`?
5. Why is running database migrations in a CI/CD pipeline generally preferable to Terraform provisioners?

---

## Hands-on Exercise

Create a complete database stack in Terraform with the following:

- An RDS PostgreSQL 15 instance with:
  - A custom parameter group enabling `log_connections` and `log_min_duration_statement = 1000`
  - A subnet group using private subnets from the networking module
  - `multi_az` conditionally enabled in prod
  - `deletion_protection` and `skip_final_snapshot` set appropriately per environment
  - A `random_password` with 24 characters stored in Secrets Manager as a JSON object with `username`, `password`, `host`, `port`, `dbname`
- A KMS key for RDS encryption and a separate KMS key for Secrets Manager, both with key rotation enabled
- An ElastiCache Redis 7.0 replication group with:
  - Encryption at rest and in transit
  - 2 nodes in prod, 1 in dev
  - Automatic failover enabled in prod
  - Slow log delivery to CloudWatch Logs
- A DynamoDB table for session storage with:
  - `PAY_PER_REQUEST` billing
  - TTL on `expires_at`
  - Point-in-time recovery enabled in prod
- Outputs for `rds_endpoint`, `db_secret_arn`, `redis_endpoint`, and `dynamodb_table_name` — mark connection endpoints as `sensitive = true`

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="11-aws-compute.md">← Previous: Building AWS Compute with Terraform</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="13-advanced-hcl.md">Next: Advanced HCL: Loops, Conditionals & Dynamic Blocks →</a>
</div>
