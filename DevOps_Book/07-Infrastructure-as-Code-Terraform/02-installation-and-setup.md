# Chapter 2 — Installation, Setup & First Resource

---

## Learning Objectives

By the end of this chapter you will be able to:

- Install Terraform on Linux and macOS and manage versions with tfenv
- Understand the standard Terraform project file structure
- Write and understand a provider configuration block
- Create your first Terraform resource (an S3 bucket with versioning)
- Read and interpret `terraform plan` output symbols
- Inspect and understand the state file
- Make changes to existing resources and apply them
- Destroy resources safely
- Import existing AWS resources into Terraform management
- Set up a correct `.gitignore` for a Terraform project

---

## 2.1 Installing Terraform

**Ubuntu / Debian:**

```bash
# Add HashiCorp's GPG key and apt repository
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform

# Verify
terraform version
# Terraform v1.9.x
# on linux_amd64
```

**macOS:**

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

terraform version
```

**Windows (Chocolatey):**

```powershell
choco install terraform
terraform version
```

**tfenv — manage multiple Terraform versions (recommended for teams):**

When working across multiple projects that pin different Terraform versions, `tfenv` lets you switch between them easily.

```bash
# Install tfenv
brew install tfenv          # macOS
# or: git clone https://github.com/tfutils/tfenv.git ~/.tfenv

# Install a specific version
tfenv install 1.9.0
tfenv install 1.8.5

# Set the version to use
tfenv use 1.9.0

# Pin a version per-project (create .terraform-version in project root)
echo "1.9.0" > .terraform-version
# tfenv automatically uses this version when you cd into the directory

terraform version
# Terraform v1.9.0
```

---

## 2.2 AWS Authentication

Terraform's AWS provider uses the same credential chain as the AWS CLI. Set up credentials before running any Terraform commands.

```bash
# Option 1: AWS CLI profile (recommended for local development)
aws configure
# AWS Access Key ID: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name: us-east-1
# Default output format: json

# Option 2: Environment variables (good for CI/CD)
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG..."
export AWS_DEFAULT_REGION="us-east-1"

# Option 3: IAM Role (best for EC2/Lambda/CI runners — no static credentials)
# Terraform auto-detects the instance role — nothing to configure

# Verify your credentials work
aws sts get-caller-identity
# {
#   "UserId": "AIDIOSFODNN7EXAMPLE",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/your-name"
# }
```

---

## 2.3 Project Structure

**Simple project (single environment):**

```
my-infrastructure/
├── main.tf           # Main resource definitions
├── variables.tf      # Input variable declarations
├── outputs.tf        # Output value declarations
├── providers.tf      # Provider configuration
├── versions.tf       # Required version constraints
├── terraform.tfvars  # Variable values (do NOT commit if it contains secrets)
└── .terraform.lock.hcl  # Provider version lock file (DO commit this)
```

**Larger project (multiple environments with modules):**

```
infra/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── ec2/
│   └── rds/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
└── .terraform.lock.hcl
```

Convention: split your configuration across multiple files by concern. Terraform merges all `.tf` files in a directory — the split is for readability, not functional necessity.

---

## 2.4 Provider Configuration

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # any 5.x version (5.0, 5.1, 5.31 — never 6.0)
    }
  }
}

provider "aws" {
  region = "us-east-1"

  # Default tags applied to every resource that supports tags
  # This ensures every resource is tagged ManagedBy and Environment
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = "development"
    }
  }
}
```

**Version constraint operators:**

| Operator | Meaning | Example |
|----------|---------|---------|
| `= 5.31.0` | Exactly this version | Pinned — brittle for providers |
| `>= 5.0` | This version or higher | Too broad — allows breaking changes |
| `~> 5.0` | Any 5.x (>= 5.0, < 6.0) | Recommended for major versions |
| `~> 5.31` | Any 5.31.x (>= 5.31, < 5.32) | Recommended for tighter control |
| `>= 5.0, < 6.0` | Explicit range | Equivalent to `~> 5.0` |

**Initialise the project:**

```bash
terraform init

# Output:
# Initializing the backend...
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Installing hashicorp/aws v5.31.0...
# - Installed hashicorp/aws v5.31.0 (signed by HashiCorp)
#
# Terraform has been successfully initialized!
#
# Terraform has created a lock file .terraform.lock.hcl to record the
# provider selections it made above. Include this file in your version
# control repository so that Terraform can guarantee to make the same
# selections by default when you run "terraform init" in the future.
```

---

## 2.5 Your First Resource: an S3 Bucket

We'll create an S3 bucket with versioning enabled and public access blocked. This is the minimum secure S3 configuration.

```hcl
# main.tf
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-2024"   # Must be globally unique across all AWS accounts

  tags = {
    Name    = "my-bucket"
    Project = "terraform-tutorial"
  }
}

resource "aws_s3_bucket_versioning" "my_bucket" {
  bucket = aws_s3_bucket.my_bucket.id   # Reference the bucket's ID attribute

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "my_bucket" {
  bucket = aws_s3_bucket.my_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Run the plan:**

```bash
terraform plan

# Output:
# Terraform will perform the following actions:
#
#   # aws_s3_bucket.my_bucket will be created
#   + resource "aws_s3_bucket" "my_bucket" {
#       + arn                         = (known after apply)
#       + bucket                      = "my-unique-bucket-name-2024"
#       + id                          = (known after apply)
#       + region                      = (known after apply)
#       + tags                        = {
#           + "Name"    = "my-bucket"
#           + "Project" = "terraform-tutorial"
#         }
#     }
#
#   # aws_s3_bucket_public_access_block.my_bucket will be created
#   + resource "aws_s3_bucket_public_access_block" "my_bucket" { ... }
#
#   # aws_s3_bucket_versioning.my_bucket will be created
#   + resource "aws_s3_bucket_versioning" "my_bucket" { ... }
#
# Plan: 3 to add, 0 to change, 0 to destroy.
```

**Apply the changes:**

```bash
terraform apply

# Terraform shows the plan again and prompts:
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
# Enter a value: yes

# aws_s3_bucket.my_bucket: Creating...
# aws_s3_bucket.my_bucket: Creation complete after 1s [id=my-unique-bucket-name-2024]
# aws_s3_bucket_versioning.my_bucket: Creating...
# ...
# Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

# For automation (CI/CD), skip the confirmation prompt:
terraform apply -auto-approve
```

---

## 2.6 Understanding Plan Output Symbols

```
Symbol  Colour   Meaning
──────  ──────   ───────
+       green    Resource will be CREATED
-       red      Resource will be DESTROYED
~       yellow   Resource will be UPDATED IN PLACE (no interruption)
-/+     red      Resource will be DESTROYED then RECREATED
                 ⚠ Causes downtime! A -/+ on a database = data loss risk!
<=      blue     Data source will be READ
```

**Always read the plan before applying.** The most dangerous symbols are `-` and `-/+`.

A `-/+ aws_db_instance.main` means your database will be destroyed and a new empty one created. A `-/+ aws_instance.web` means your EC2 instance will be terminated and replaced. For stateless resources like security groups or IAM policies, `-/+` is fine. For stateful resources (databases, EBS volumes), it is a serious event that requires explicit sign-off.

---

## 2.7 Inspecting State

```bash
# List all resources Terraform is managing
terraform state list
# aws_s3_bucket.my_bucket
# aws_s3_bucket_versioning.my_bucket
# aws_s3_bucket_public_access_block.my_bucket

# Show details of a specific resource (pulled from state)
terraform state show aws_s3_bucket.my_bucket
# # aws_s3_bucket.my_bucket:
# resource "aws_s3_bucket" "my_bucket" {
#     arn                         = "arn:aws:s3:::my-unique-bucket-name-2024"
#     bucket                      = "my-unique-bucket-name-2024"
#     id                          = "my-unique-bucket-name-2024"
#     region                      = "us-east-1"
#     ...
# }

# Show all defined outputs
terraform output

# Inspect the raw state file (JSON — informational only, do NOT edit)
cat terraform.tfstate
```

---

## 2.8 Making Changes

Edit `main.tf` to add a tag:

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-2024"

  tags = {
    Name        = "my-bucket"
    Project     = "terraform-tutorial"
    CostCenter  = "CC-1234"   # Added
  }
}
```

```bash
terraform plan
# ~ aws_s3_bucket.my_bucket will be updated in-place
#   ~ tags = {
#       + "CostCenter" = "CC-1234"
#         # (2 unchanged elements hidden)
#     }
#
# Plan: 0 to add, 1 to change, 0 to destroy.

terraform apply
# Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

This is an in-place update (`~`). Adding a tag does not require recreating the bucket — AWS supports tag updates without interruption.

---

## 2.9 Destroying Resources

```bash
# Destroy a specific resource only (useful during development)
terraform destroy -target=aws_s3_bucket_versioning.my_bucket

# Destroy everything in the configuration
terraform destroy

# Output:
# Plan: 0 to add, 0 to change, 3 to destroy.
# Do you really want to destroy all resources?
# Enter a value: yes
#
# Destroy complete! Resources: 3 destroyed.
```

`terraform destroy` in production should NEVER be run without:
1. Thoroughly reading the destroy plan
2. Ensuring all stateful data is backed up
3. Explicit approval from team leadership

In many organisations, `terraform destroy` on a production workspace is blocked by CI policy.

---

## 2.10 Importing Existing Resources

If you have resources already created in AWS (by clicking in the console, or by another tool), you can bring them under Terraform management.

```bash
# Import an existing S3 bucket into Terraform state
# Syntax: terraform import <resource_type>.<name> <aws_resource_id>
terraform import aws_s3_bucket.existing_bucket my-existing-bucket-name

# Terraform will now track this bucket in state.
# BUT: you still need to write the matching .tf config block.

# Use terraform show to see the current state as HCL config
terraform show

# Then run plan — it should show 0 changes if your .tf matches reality
terraform plan
# No changes. Your infrastructure matches the configuration.
```

**Terraform 1.5+ import blocks (declarative import):**

```hcl
# main.tf — define the import inline
import {
  to = aws_s3_bucket.existing_bucket
  id = "my-existing-bucket-name"
}

resource "aws_s3_bucket" "existing_bucket" {
  bucket = "my-existing-bucket-name"
}
```

```bash
# Terraform 1.5+: generate config automatically from an existing resource
terraform plan -generate-config-out=generated.tf
# Creates generated.tf with the HCL for the imported resource
```

---

## 2.11 The .terraform Directory and Lock File

```
.terraform/                    # Downloaded provider plugins
                               # → DO NOT commit (add to .gitignore)
                               # → Regenerated by terraform init

.terraform.lock.hcl            # Records exact provider versions selected
                               # → DO commit — ensures all team members and CI
                               #   use the same provider versions

terraform.tfstate              # Local state file
                               # → DO NOT commit local state
                               # → Use remote backend in production (Chapter 7)

terraform.tfstate.backup       # Previous state (kept automatically)
                               # → DO NOT commit
```

**`.gitignore` for Terraform projects:**

```gitignore
# Terraform providers (large binary files, regenerated by init)
.terraform/

# Local state files (use remote backend instead)
terraform.tfstate
terraform.tfstate.backup

# Variable files that may contain secrets
*.tfvars
*.tfvars.json

# Keep an example tfvars file (no real values) to document required variables
!example.tfvars
!example.tfvars.json

# Override files (local developer overrides, not for sharing)
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Crash log from Terraform
crash.log
crash.*.log
```

The `.terraform.lock.hcl` file is intentionally NOT in `.gitignore`. Commit it. It ensures reproducibility across machines and CI runners.

---

## Summary

You have installed Terraform, configured the AWS provider, created an S3 bucket with versioning and public access block, read plan output, inspected state, made an in-place update, and destroyed resources. These are the fundamental operations everything else builds on.

The key mental model: Terraform reads your `.tf` files (desired state), reads the state file (known current state), asks AWS what actually exists (actual state), and calculates the minimum changes to get from actual to desired.

---

## Knowledge Check

1. What does `terraform init` do? When should you run it?
2. What is the difference between `terraform plan` and `terraform apply`?
3. What does the `-/+` symbol in a plan output mean? Why is it dangerous for databases?
4. What is the `.terraform.lock.hcl` file and why should you commit it to Git?
5. You have an EC2 instance created manually in the AWS console. How do you bring it under Terraform management?

---

## Hands-On Exercise

Complete the following sequence in your own AWS account (use the free tier):

1. Install Terraform and configure the AWS provider pointing to `us-east-1`
2. Create `providers.tf`, `main.tf`, `variables.tf`, and `outputs.tf`
3. Write a configuration that creates:
   - An S3 bucket with a globally unique name
   - Versioning enabled on the bucket
   - Public access block on the bucket
   - An output that prints the bucket ARN
4. Run `terraform init`, `terraform plan`, `terraform apply`
5. Inspect the state: `terraform state list`, `terraform state show`
6. Add a tag to the bucket and apply again — observe the `~` update
7. Run `terraform destroy` to clean up

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./01-introduction.md">← Previous: Introduction to IaC & Terraform</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./03-hcl-fundamentals.md">Next: HCL Fundamentals →</a>
</div>
