# Chapter 7 — Remote Backends & State Locking

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why local state is insufficient for team workflows
- Configure an S3 + DynamoDB remote backend from scratch
- Migrate an existing local state to a remote backend
- Read outputs from another Terraform config via `terraform_remote_state`
- Implement partial backend configuration for multi-environment deployments
- Recover previous state versions from S3 versioning
- Understand when to use Terraform Cloud as an alternative

---

## 7.1 Why Remote Backends?

Problems with local state:

- **No team sharing:** each engineer has their own state file
- **No locking:** two applies simultaneously = corrupted state
- **No backup:** laptop lost = state lost = Terraform doesn't know what it created
- **No audit:** who changed what when?
- **Security risk:** state in git (with secrets) = breach

Remote backend: store state in a shared, versioned, encrypted location with locking.

---

## 7.2 S3 + DynamoDB Backend (Standard AWS Setup)

```hcl
# Create the backend infrastructure FIRST (manually or with a separate "bootstrap" config)
# backend.tf in your main project:

terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/myapp/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "alias/terraform-state"

    # DynamoDB table for locking
    dynamodb_table = "terraform-state-locks"
  }
}
```

Bootstrap resources to create the state bucket (run once, commit state locally):

```hcl
# bootstrap/main.tf — run this once with local state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true   # NEVER accidentally delete the state bucket
  }
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
  bucket = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

resource "aws_kms_key" "terraform_state" {
  description             = "Terraform state encryption key"
  deletion_window_in_days = 30
}
```

---

## 7.3 Migrating from Local to Remote Backend

```bash
# 1. Add backend configuration to providers.tf
# 2. Run terraform init — Terraform detects the change

terraform init
# Initializing the backend...
# Do you want to copy existing state to the new backend?
# Enter a value: yes
#
# Successfully configured the backend "s3"!
# Terraform will automatically use this backend unless the backend configuration changes.

# 3. Delete local state files (they're now in S3)
rm terraform.tfstate terraform.tfstate.backup
```

---

## 7.4 Backend State Keys / Path Convention

```
S3 bucket: mycompany-terraform-state
├── network/
│   └── terraform.tfstate          (VPC, subnets, peering)
├── platform/
│   └── terraform.tfstate          (EKS, ECR, shared tooling)
├── prod/
│   ├── myapp/terraform.tfstate    (app resources)
│   └── database/terraform.tfstate (RDS)
└── dev/
    ├── myapp/terraform.tfstate
    └── database/terraform.tfstate
```

Each Terraform "root module" (directory) has its own state file. Shared resources like VPCs have their own state, referenced by other configs via the `terraform_remote_state` data source.

---

## 7.5 Reading Another Config's State

```hcl
# In the app config, read VPC outputs from the network config
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id              = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [data.terraform_remote_state.network.outputs.app_security_group_id]
}
```

> **Note:** this creates tight coupling between configs. Alternative: use AWS data sources (`data "aws_vpc"` with tag filters) — more loosely coupled and does not require the caller to have read access to the network state file.

---

## 7.6 IAM Policy for Terraform Backend

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::mycompany-terraform-state/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::mycompany-terraform-state"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/terraform-state-locks"
    },
    {
      "Effect": "Allow",
      "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
      "Resource": "arn:aws:kms:us-east-1:*:key/*"
    }
  ]
}
```

Attach this policy to the IAM role your CI/CD pipeline assumes. Developers can have a read-only variant (`s3:GetObject`, `dynamodb:GetItem`) if they only need to run `terraform plan`.

---

## 7.7 Terraform Cloud Backend

```hcl
# Use Terraform Cloud for free remote state + team features
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-myapp"
    }
  }
}
```

```bash
terraform login   # opens browser for authentication
terraform init    # configures the Cloud backend
```

Benefits:

- Free for small teams (up to 5 users)
- State stored and versioned automatically
- Browser UI to view state, plans, runs
- Team collaboration features
- Policy as Code (Sentinel — Enterprise)

---

## 7.8 Partial Backend Configuration

```hcl
# Don't hardcode backend config — allows different backends per environment
terraform {
  backend "s3" {
    # Only specify what's constant
    region = "us-east-1"
    # bucket and key come from -backend-config flags
  }
}
```

```bash
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="key=prod/myapp/terraform.tfstate" \
  -backend-config="dynamodb_table=terraform-state-locks"
```

This allows the same code to target different buckets and keys in different environments without modifying the source files.

---

## 7.9 State Versioning and Recovery

```bash
# S3 versioning is enabled on the state bucket
# List all versions of the state file
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix prod/myapp/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified,IsLatest]' \
  --output table

# Download a specific historical version
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key prod/myapp/terraform.tfstate \
  --version-id <VersionId> \
  terraform.tfstate.backup

# Restore old state (if apply corrupted state)
aws s3 cp terraform.tfstate.backup \
  s3://mycompany-terraform-state/prod/myapp/terraform.tfstate
```

> **Warning:** restoring old state tells Terraform it manages a different set of resources than actually exist. Always review `terraform plan` immediately after a restore before running `apply`.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Remote backend | Shared, encrypted, versioned state with locking |
| S3 + DynamoDB | Standard AWS pattern; S3 stores state, DynamoDB holds locks |
| Bootstrap | Create backend infra first with local state, then migrate |
| State keys | One state file per root module; use a consistent path hierarchy |
| `terraform_remote_state` | Read outputs from another config; use sparingly to avoid tight coupling |
| Partial config | Pass bucket/key at init time for environment flexibility |
| Terraform Cloud | Managed alternative; good for small teams |

---

## Knowledge Check

1. Why is storing Terraform state in a Git repository considered a security risk?
2. What happens if two engineers run `terraform apply` simultaneously without state locking?
3. What AWS resources are required to implement the standard S3 remote backend with locking?
4. When would you use `terraform_remote_state` vs an AWS data source like `data "aws_vpc"`?
5. What is partial backend configuration and why is it useful in a multi-environment setup?

---

## Hands-on Exercise

**Goal:** implement the S3 + DynamoDB backend from scratch.

1. Create the bootstrap Terraform config (`bootstrap/main.tf`) with the S3 bucket, versioning, encryption, public access block, DynamoDB table, and KMS key. Apply it with local state.
2. In your main project, add the `backend "s3"` block pointing to the resources you just created.
3. Run `terraform init` and confirm Terraform offers to copy existing state. Accept.
4. Delete the local `.tfstate` files and verify they are no longer present.
5. Verify the state file appears in S3 under the correct key path.
6. Simulate a lock: open two terminals, start a `terraform apply` in the first, and immediately run `terraform apply` in the second. Observe the lock error in the second terminal.
7. List the S3 object versions for your state file using the AWS CLI.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-state-management.md">← Previous: State Management</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-modules.md">Next: Modules →</a>
</div>
