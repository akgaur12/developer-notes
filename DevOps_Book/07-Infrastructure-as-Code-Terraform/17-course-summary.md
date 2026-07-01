# Chapter 17 — Course Summary & Next Steps

## 17.1 What You've Learned

Congratulations on completing **Infrastructure as Code (Terraform)** — Topic 7 of the DevOps Learning Path. You can now turn manual AWS infrastructure into reproducible, reviewable, version-controlled code.

| Chapter | What You Can Do Now |
|---------|---------------------|
| 01 Intro to IaC | Explain why IaC beats manual infra; describe Terraform's architecture |
| 02 Setup | Install Terraform, configure providers, write and apply first resources |
| 03 HCL | Write any HCL construct: resources, variables, locals, data sources, outputs |
| 04 Providers | Configure multi-region/multi-account providers; use lifecycle rules |
| 05 Variables & Outputs | Build clean module interfaces; use data sources; validate inputs |
| 06 State | Understand state, manipulate it safely, handle drift |
| 07 Remote Backends | Set up S3+DynamoDB backend; migrate local state; recover from corruption |
| 08 Modules | Build and publish reusable modules; use public Registry modules |
| 09 Environments | Manage dev/staging/prod without code duplication |
| 10 AWS Networking | Terraform VPCs, security groups, ACM, Route53 |
| 11 AWS Compute | Terraform EC2, ALB, ASG, Lambda, CloudWatch |
| 12 AWS Databases | Terraform RDS, ElastiCache, DynamoDB with encryption |
| 13 Advanced HCL | count, for_each, dynamic, moved, preconditions |
| 14 CI/CD | Plan on PR, apply on merge, Atlantis, drift detection |
| 15 Best Practices | Code organisation, naming, secrets, state strategy |
| 16 Projects | Complete production-grade infrastructure deployed end-to-end |

---

## 17.2 Completion Checklist

```
Core Terraform Skills:
  Can write and apply a Terraform config from scratch
  Can refactor a monolith into modules without destroying resources
  Can set up S3 + DynamoDB remote backend
  Can manage multiple environments with separate state
  Can use count, for_each, and dynamic blocks fluently
  Can import existing AWS resources into Terraform state
  Can debug state issues and recover from corruption

AWS with Terraform:
  Provisioned a complete VPC (subnets, IGW, NAT, routes) in Terraform
  Provisioned ALB + ASG with rolling deployment in Terraform
  Provisioned RDS with encryption and Secrets Manager integration
  Created IAM roles using aws_iam_policy_document data source
  Set up ACM certificate with Route53 DNS validation

CI/CD:
  GitHub Actions pipeline running plan on PR and apply on merge
  OIDC auth (no static credentials in CI)
  Drift detection running on schedule
  Pre-commit hooks enforcing fmt and validate

Projects:
  Completed Project 1: Bootstrap state infrastructure
  Completed Project 2: Reusable VPC module
  Completed Project 3: Complete 3-tier application stack
  Completed Project 4: GitOps CI/CD pipeline
  Completed Project 5: Full production platform capstone
```

---

## 17.3 Terraform Quick Reference

```bash
# Lifecycle commands
terraform init                       # download providers, configure backend
terraform fmt                        # format all .tf files
terraform validate                   # check syntax
terraform plan                       # show what will change
terraform plan -out=tfplan           # save plan
terraform apply                      # apply (prompts for confirmation)
terraform apply -auto-approve        # apply without prompt (CI)
terraform apply tfplan               # apply saved plan
terraform destroy                    # destroy everything
terraform destroy -target=resource   # destroy specific resource

# State commands
terraform state list                 # list all managed resources
terraform state show ADDR            # show resource attributes
terraform state mv FROM TO           # rename without destroying
terraform state rm ADDR              # remove from state (keep real resource)
terraform state pull                 # download state to stdout
terraform state push FILE            # upload state file (danger!)
terraform force-unlock LOCK_ID       # release stuck lock

# Import commands
terraform import ADDR ID             # import existing resource into state
# TF 1.5+: use import {} block in config

# Workspace commands
terraform workspace list
terraform workspace new NAME
terraform workspace select NAME
terraform workspace delete NAME

# Inspection commands
terraform output                     # show all outputs
terraform output -json               # JSON format (for scripting)
terraform show                       # show all state in readable format
terraform graph | dot -Tsvg > graph.svg  # visualise dependency graph
terraform providers                  # show required providers

# Debugging
export TF_LOG=DEBUG                  # enable debug logging
export TF_LOG_PATH=./debug.log      # log to file
terraform console                   # interactive expression evaluation
```

---

## 17.4 HCL Quick Reference

```hcl
# Resource
resource "TYPE" "NAME" { ... }

# Data source
data "TYPE" "NAME" { ... }

# Variable
variable "NAME" { type = string; default = "value" }

# Local
locals { name = "value" }

# Output
output "NAME" { value = resource.name.attr }

# Module
module "NAME" { source = "./path"; version = "~> 1.0" }

# References
var.name                   # variable
local.name                 # local value
resource.type.name.attr    # resource attribute
data.type.name.attr        # data source attribute
module.name.output         # module output
path.module                # current module directory
terraform.workspace        # current workspace name

# Expressions
condition ? true_val : false_val      # conditional
[for item in list : item.attr]        # list comprehension
{for k, v in map : k => upper(v)}    # map comprehension
list[*].attr                          # splat
merge(map1, map2)                     # merge maps
toset(["a", "b"])                     # list to set (for for_each)
```

---

## 17.5 What's Next: Topic 8 — Kubernetes Basics

You now manage AWS infrastructure as code. The natural next step is running containerised applications at scale — and that means Kubernetes.

AWS provides EKS (Elastic Kubernetes Service). In Topic 8 you'll provision an EKS cluster using Terraform (you'll write the Terraform yourself using what you learned here), then learn to operate it: deploying applications as Pods, exposing them as Services, routing traffic with Ingress, managing config with ConfigMaps and Secrets, and scaling with HPA.

Everything from this course applies directly:

| What you learned | How it maps to Kubernetes |
|---|---|
| VPC, subnets, security groups | EKS node networking |
| IAM roles | IRSA (IAM Roles for Service Accounts) |
| ECR | Kubernetes image pulls |
| ALB | AWS Load Balancer Controller for Kubernetes Ingress |
| CloudWatch | Container Insights for Kubernetes monitoring |

---

## 17.6 The Full Picture

```
Topics 1–4:   Core foundations
  Linux, Networking, Git, Docker

Topics 5–7:   Cloud and automation  (YOU ARE HERE — finishing 7)
  CI/CD Pipelines, AWS, Terraform

Topics 8–9:   Container orchestration
  Kubernetes Basics, Advanced Kubernetes

Topics 10–11: Production operations
  Monitoring & Logging, Security (DevSecOps)
```

You are now 7 of 11 courses complete — well past the halfway point. The skills from here forward build directly on everything you've learned. The Kubernetes course will feel natural because you already understand networking (Topic 2), containers (Topic 4), AWS (Topic 6), and IaC (Topic 7).

---

## 17.7 Recommended Resources

- **Terraform: Up & Running** by Yevgeniy Brikman — the definitive Terraform book; covers modules, state, and testing in depth
- **The Terraform Book** by James Turnbull — practical day-to-day reference
- **HashiCorp Learn** (developer.hashicorp.com/terraform) — official tutorials and free hands-on labs
- **Terraform Registry** (registry.terraform.io) — community modules and provider documentation
- **Gruntwork Blog** — deep Terraform patterns from the creators of Terragrunt
- **tfsec** (aquasecurity.github.io/tfsec) — static security scanning for Terraform HCL
- **Checkov** (checkov.io) — policy-as-code for IaC; enforces security and compliance rules

---

## 17.8 Progress Tracker

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | Complete |
| 4 | [Docker](../04-Docker/00-index.md) | Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | Complete |
| 6 | [Cloud Fundamentals (AWS)](../06-Cloud-Fundamentals-AWS/00-index.md) | Complete |
| 7 | Infrastructure as Code (Terraform) | Complete — you are here |
| 8 | Kubernetes Basics | Up next |
| 9 | Advanced Kubernetes | Coming soon |
| 10 | Monitoring & Logging | Coming soon |
| 11 | Security (DevSecOps) | Coming soon |

Continue to: **[Kubernetes Basics](../08-Kubernetes-Basics/00-index.md)**

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="16-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../08-Kubernetes-Basics/00-index.md">Next: Kubernetes Basics →</a>
</div>
