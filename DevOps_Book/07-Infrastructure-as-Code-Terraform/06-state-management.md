# Chapter 6 — State Management

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what Terraform state is and why it exists
- Use `terraform state` subcommands to inspect, move, and remove resources
- Understand state locking and why it is critical in team environments
- Handle sensitive values in state safely
- Detect and reconcile infrastructure drift
- Import existing resources into Terraform management
- Force resource replacement with `-replace`

---

## 6.1 What Is Terraform State?

Terraform state is a JSON file (`terraform.tfstate`) that maps your Terraform resource definitions to real-world infrastructure objects. It is Terraform's source of truth about what it manages.

**Why state is necessary:**

- Without state, Terraform cannot know which real resources already exist — it would attempt to create duplicates on every apply.
- State stores the resource ID that ties a Terraform block to a specific AWS/GCP/Azure object.
- State caches resource attribute values to calculate accurate diffs without querying every API on every plan.
- State tracks the dependency graph for correct destroy ordering.

```json
// terraform.tfstate (simplified)
{
  "version": 4,
  "terraform_version": "1.9.0",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_bucket",
      "instances": [{
        "attributes": {
          "id": "my-unique-bucket-name-2024",
          "arn": "arn:aws:s3:::my-unique-bucket-name-2024",
          "bucket": "my-unique-bucket-name-2024",
          "region": "us-east-1"
        }
      }]
    }
  ]
}
```

The state file is generated and maintained automatically by Terraform. You should almost never edit it by hand.

---

## 6.2 State Operations

```bash
# List all resources currently tracked in state
terraform state list

# Show all attributes of a specific resource
terraform state show aws_instance.web

# Move/rename a resource in state without destroying and recreating it
# Use this when you rename a resource block in your .tf files
terraform state mv aws_instance.web aws_instance.web_server

# Move a resource into a module
terraform state mv aws_instance.web module.app.aws_instance.web

# Remove a resource from state (stop tracking it) without destroying the real resource
# Useful when you want Terraform to "forget" about a resource you will manage elsewhere
terraform state rm aws_s3_bucket.logs
# The bucket still exists in AWS, but Terraform no longer tracks it.

# Pull the current remote state to a local file (for inspection or backup)
terraform state pull > current_state.json

# Push a modified state back to the backend (DANGEROUS — last resort only)
terraform state push modified_state.json
```

**When to use `state mv`:**
- You renamed a resource block in your `.tf` file
- You are moving a resource into or out of a module
- You are reorganising resources between configurations

Without `state mv`, Terraform sees a rename as "delete old + create new", which means downtime or data loss for critical resources.

---

## 6.3 State Locking

**The problem:** Two engineers run `terraform apply` simultaneously. Both read the same state, both compute a plan, and both try to write an updated state. The second write corrupts the state because it was based on stale data.

**The solution:** State locking. When any operation that modifies state begins, Terraform acquires a lock. Other operations wait or fail until the lock is released.

- S3 backend uses DynamoDB for locking (covered in Chapter 7)
- Local state has no locking — never use local state in team environments

```bash
# If a lock is stuck (someone's apply crashed mid-run)
terraform force-unlock <LOCK_ID>

# The LOCK_ID is printed in the error message you see when trying to run Terraform:
#   Error: Error locking state: Error acquiring the state lock:
#   Lock Info:
#     ID:        <LOCK_ID>
#     Path:      s3://my-state-bucket/prod/terraform.tfstate
#     Operation: OperationTypeApply
#     ...

# Only force-unlock if you are certain no other Terraform process is running.
```

---

## 6.4 Sensitive Values in State

WARNING: Terraform state stores all resource attributes in plaintext by default. This includes:

- Database passwords
- IAM secret access keys
- Private certificate private keys
- Any value passed to a `sensitive` variable

```hcl
# Marking an output sensitive hides it in CLI output...
output "db_password" {
  value     = random_password.db.result
  sensitive = true
}
# ...but the value is still stored in plaintext in terraform.tfstate.
```

**Mitigation checklist:**

| Control | How |
|---|---|
| Encrypt state at rest | S3 SSE-KMS (specify a KMS key, not the default SSE-S3) |
| Encrypt state in transit | Always use HTTPS (S3 enforces this) |
| Restrict state access | IAM policies on the S3 bucket and DynamoDB table |
| Rotate compromised secrets | Use Secrets Manager/Vault for secrets; store only ARNs in Terraform |
| Never commit state to git | Add `terraform.tfstate` and `terraform.tfstate.backup` to `.gitignore` |

---

## 6.5 Handling State Drift

Drift occurs when the real infrastructure no longer matches the state file. Common causes:

- Manual changes made in the AWS console or CLI
- Another tool or person modifying the same resource
- AWS making automatic changes (e.g., auto-scaling, certificate renewal)

```bash
# Refresh state: query the real current state of all managed resources
# and update the state file to match — without changing infrastructure.
terraform apply -refresh-only

# Review the proposed state updates, then confirm.
# This is how you reconcile drift without forcing infrastructure changes.

# terraform refresh is deprecated in TF 1.5+; use -refresh-only instead.

# Regular terraform plan already shows drift automatically:
# ~ aws_security_group.web will be updated in-place
#   ~ ingress {
#       - cidr_blocks = ["10.0.0.0/8"]   (manual addition detected)
#       + cidr_blocks = []
#     }
```

If you want to keep the manual change, update your Terraform config to match. If you want to revert it, run `terraform apply`.

---

## 6.6 State File Best Practices

```
Never commit terraform.tfstate to git — add it to .gitignore immediately.
Always use a remote backend for team usage (S3 + DynamoDB, Terraform Cloud, etc.).
Never manually edit the state file JSON — use terraform state subcommands.
Use terraform state mv for refactoring resource names and module moves.
Enable S3 bucket versioning on your state bucket — enables rollback on corruption.
Restrict state bucket access with IAM: only CI/CD roles and senior engineers.
Encrypt state at rest with SSE-KMS and enforce HTTPS-only bucket policies.
Keep one state file per environment (dev, staging, prod) — never share state.
```

---

## 6.7 Workspaces (Local State)

```bash
# Workspaces allow one configuration to maintain multiple independent state files.
# The default workspace is named "default".

terraform workspace list
# * default

terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select prod

# In configuration: use terraform.workspace to branch on environment
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  count         = terraform.workspace == "prod" ? 3 : 1
}
```

Workspaces are convenient for small teams but have limitations: all workspaces share the same backend configuration and the same set of provider credentials. For strict environment isolation, separate root modules with separate state files are usually preferred. Workspaces are covered in depth in Chapter 9.

---

## 6.8 Importing Existing Resources

If you have infrastructure that was created outside Terraform and you want to bring it under Terraform management, use `terraform import`.

```bash
# Import an EC2 instance into state
terraform import aws_instance.web i-1234567890abcdef0

# Import an S3 bucket
terraform import aws_s3_bucket.logs my-existing-log-bucket

# Import an RDS instance
terraform import aws_db_instance.main myapp-prod-db

# After import:
# 1. Write the matching resource block in your .tf files.
# 2. Run terraform plan.
# 3. Iterate on the config until plan shows: No changes. Your infrastructure matches the configuration.
```

**Declarative import blocks (Terraform 1.5+):**

```hcl
# Define the import in config instead of running a CLI command
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket-name"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket-name"
  # add all required and relevant attributes
}
```

Terraform 1.5+ can also generate the resource configuration automatically with `terraform plan -generate-config-out=generated.tf` — a useful starting point for large imports.

---

## 6.9 Taint and Replace

Sometimes a resource is in a bad state (e.g., an EC2 instance with a corrupt filesystem, an RDS instance that failed a snapshot). You need Terraform to destroy and recreate it on the next apply.

```bash
# Modern approach (Terraform 1.2+): targeted replace
terraform apply -replace=aws_instance.web
# Plan shows: -/+ aws_instance.web (tainted, will be destroyed and then created)

# You can specify multiple resources
terraform apply -replace=aws_instance.web -replace=aws_instance.worker

# Legacy approach (deprecated in TF 0.15.2, removed in TF 1.0+)
terraform taint aws_instance.web
terraform plan
terraform apply
```

`-replace` is surgical: only the specified resource is replaced. Everything else in the plan is unchanged.

---

## 6.10 Debugging State Issues

```bash
# Enable verbose logging to diagnose unexpected plan/apply behaviour
export TF_LOG=DEBUG
export TF_LOG_PATH=./terraform.log
terraform plan

# Log levels (from most to least verbose): TRACE, DEBUG, INFO, WARN, ERROR

# Validate that the state file is valid JSON
terraform state pull | python3 -m json.tool > /dev/null && echo "State is valid JSON"

# Find which Terraform resource manages a specific AWS resource ID
terraform state list | xargs -I{} terraform state show {} | grep "i-1234567890abcdef0"

# Count resources in state
terraform state list | wc -l
```

---

## Summary

- State is a JSON file mapping Terraform resource blocks to real-world infrastructure. It is essential for plan accuracy and is maintained automatically.
- Use `terraform state list`, `state show`, `state mv`, and `state rm` for safe state manipulation — never edit the JSON directly.
- State locking prevents concurrent applies from corrupting state. Use DynamoDB with the S3 backend.
- All resource attributes are stored in plaintext in state. Encrypt the state bucket with KMS and restrict access.
- Drift is detected automatically by `terraform plan`. Use `terraform apply -refresh-only` to reconcile state without changing infrastructure.
- `terraform import` and declarative `import` blocks bring existing resources under Terraform management.
- `terraform apply -replace` forces recreation of a specific resource without affecting others.

---

## Knowledge Check

1. Why does Terraform need a state file? What would happen on `terraform apply` without one?
2. You rename `aws_instance.web` to `aws_instance.web_server` in your `.tf` file and run `terraform plan`. What does Terraform propose, and how do you avoid downtime?
3. A teammate's `terraform apply` crashed mid-run and the state is now locked. What command do you run, and what precaution must you take first?
4. You find that a colleague manually added an inbound security group rule in the AWS console. How does Terraform surface this, and what are your two options for resolving it?
5. A sensitive output is marked `sensitive = true`. Is it safe to store the state file in an unencrypted S3 bucket?

---

## Hands-on Exercise

Practice the full state operations workflow:

1. Run `terraform state list` to see all managed resources.
2. Pick one resource and run `terraform state show` to inspect its attributes.
3. Rename a resource block in your `.tf` file, then use `terraform state mv` to update state without recreating the resource.
4. Remove a low-risk resource from state with `terraform state rm`, confirm it still exists in AWS, then re-import it with `terraform import`.
5. Make a manual change to a resource in the AWS console (e.g., add a tag or security group rule), then run `terraform plan` and observe how Terraform surfaces the drift.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="05-variables-outputs-data.md">← Previous: Variables, Outputs & Data Sources</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="07-remote-backends.md">Next: Remote Backends & State Locking →</a>
</div>
