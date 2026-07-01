# Chapter 1 — Introduction to IaC & Terraform

---

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the problems that Infrastructure as Code solves
- Describe the difference between declarative and imperative IaC
- Compare Terraform to alternative IaC tools and know why Terraform dominates
- Explain the Terraform core loop (write → plan → apply)
- Describe Terraform's internal architecture (Core, Providers, State)
- Understand the team workflow: PR → plan → review → merge → apply

---

## 1.1 The Problem with Manual Infrastructure

Before IaC, infrastructure was managed manually. Here is what that looked like in practice.

**"Works on my machine"**

An engineer SSH'd into the production server and installed dependencies by hand. Nobody wrote it down. Three months later, the application breaks on a new server because it's missing a library that was installed manually and never documented. Nobody knows exactly what's running in production.

**Snowflake servers**

Each server was configured slightly differently over time: different versions, different config file edits, different cron jobs added ad-hoc. When you need to scale out and add a new server, you can't reproduce the original. The new server behaves differently, and debugging takes days.

**Undocumented change**

Someone SSH'd into the production web server at 2am during an incident and edited `/etc/nginx/nginx.conf` to fix a timeout. It worked. They forgot to document it. Two weeks later, an automated deployment overwrites the file with the version from the repository — the timeout returns, the site goes down, and nobody knows why.

**Disaster recovery failure**

The company's disaster recovery plan said: "Restore from backup and reconfigure." When a catastrophic failure actually happened, the "reconfigure" step was a 200-step document that was two years out of date. Recovery took four days.

**Configuration drift**

What's running in production doesn't match the architecture diagram. The diagram shows three servers; production has five. Two were added six months ago for a load test and never removed. AWS is billing for them every month.

These are not edge cases. They are the normal state of manually managed infrastructure at any organisation operating for more than a year.

---

## 1.2 What Is Infrastructure as Code?

Infrastructure as Code (IaC) means describing your infrastructure in code files — version-controlled in Git — and applying changes automatically through tooling rather than manually.

```
Manual Infrastructure                  Infrastructure as Code
─────────────────────────              ──────────────────────────────
Click in AWS console           →       Write a .tf file
SSH in and run commands        →       Run terraform apply
"I think it's configured"      →       The code IS the configuration
"Who changed that?"            →       git log — author, date, reason
"Recover from scratch"         →       git clone + terraform apply
Two different environments     →       Same code, different tfvars
```

**Benefits of IaC:**

- **Reproducibility** — same code produces the same infrastructure, every time, in any account or region
- **Version history** — every change is a git commit with an author, timestamp, and commit message explaining why
- **Code review** — infrastructure changes go through PR review like application code; a second pair of eyes catches mistakes before they hit production
- **Automation** — no manual steps means no human error; CI/CD applies changes automatically
- **Documentation** — the code IS the documentation and is always up to date (unlike a wiki)
- **Disaster recovery** — rebuild the entire infrastructure from code in minutes, not days

---

## 1.3 IaC Approaches: Declarative vs Imperative

There are two fundamental approaches to IaC:

| Approach | You Say | Tool Figures Out | Example Tools |
|----------|---------|------------------|---------------|
| **Declarative** | "I want 3 EC2 instances" | How to create/modify/delete to reach the goal | Terraform, CloudFormation, Pulumi |
| **Imperative** | "Create instance 1, create instance 2, create instance 3" | Nothing — you write every step | Bash + AWS CLI, Ansible |

**Declarative wins for infrastructure.** You describe the desired end state; Terraform figures out the plan.

- If 2 instances already exist → Terraform creates 1 more
- If you reduce from 3 to 2 → Terraform destroys 1
- If nothing changed → Terraform does nothing

You never have to write "check if it already exists, skip if it does." Terraform handles idempotency automatically.

With an imperative script:

```bash
# Imperative (fragile): what if one already exists?
aws ec2 run-instances --image-id ami-xxx --count 1
aws ec2 run-instances --image-id ami-xxx --count 1
aws ec2 run-instances --image-id ami-xxx --count 1
# If run twice → 6 instances, not 3
```

With Terraform (declarative):

```hcl
# Declarative: just say what you want
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-xxx"
  instance_type = "t3.micro"
}
# Run twice → still 3 instances. Terraform reconciles to desired state.
```

---

## 1.4 Terraform vs Alternatives

| Tool | Approach | Language | Cloud Support | Key Notes |
|------|----------|----------|---------------|-----------|
| **Terraform** | Declarative | HCL | Multi-cloud | Most popular, huge ecosystem, OSS (BSL since 2023) |
| **OpenTofu** | Declarative | HCL | Multi-cloud | Open-source fork of Terraform (Linux Foundation, truly OSS) |
| **AWS CloudFormation** | Declarative | YAML/JSON | AWS only | Free, native AWS, deeply integrated, verbose |
| **AWS CDK** | Declarative | Python/TS/Go | AWS only | Code compiles to CloudFormation; excellent for developers |
| **Pulumi** | Declarative | Python/TS/Go | Multi-cloud | Real programming languages instead of HCL; newer, growing |
| **Ansible** | Imperative | YAML | Multi-cloud | Config management + IaC, procedural, better for app config |

**Why Terraform dominates:**

1. **Largest community** — Stack Overflow answers, blog posts, tutorials, YouTube videos
2. **Terraform Registry** — 3,000+ official and community providers; 15,000+ public modules
3. **Multi-cloud** — one tool manages AWS, GCP, Azure, Kubernetes, GitHub, Datadog, PagerDuty, Cloudflare, and more
4. **State management** — tracks exactly what exists; calculates the minimal set of changes needed
5. **Mature tooling** — `tflint`, `tfsec`, `checkov`, Atlantis, Terraform Cloud, Spacelift, env0

**The OpenTofu note:** In August 2023, HashiCorp changed Terraform's license from MPL 2.0 (open-source) to BSL 1.1, which restricts commercial use by competitors. The community forked it as OpenTofu under the Linux Foundation. For most organisations using Terraform internally (not selling it as a product), the license change has no impact. This course covers Terraform; the syntax is 100% compatible with OpenTofu.

---

## 1.5 How Terraform Works: The Core Loop

```
Write          →        Plan          →        Apply
─────────               ──────                 ───────
Edit .tf files    terraform plan          terraform apply
                  (read-only, safe)       (makes changes)

                  Shows a diff:
                  + add this resource
                  ~ update that attribute
                  - delete this resource
                  -/+ destroy and recreate
```

The three Terraform commands you'll use 90% of the time:

```bash
terraform init    # Download providers, initialise the backend
terraform plan    # Show what will change (read-only — safe to run anytime)
terraform apply   # Actually make the changes in the cloud
```

The **plan** step is critically important. Always read the plan output before applying. A plan showing `-/+ aws_db_instance.main` means your database will be destroyed and recreated — data loss. Catching that before apply is why the plan step exists.

---

## 1.6 Terraform Architecture

```
Your Code (*.tf files)
      │
      ▼
┌─────────────────────────────────────┐
│           Terraform Core            │
│  - Reads all .tf files              │
│  - Reads current state file         │
│  - Calculates diff                  │
│    (desired state vs actual state)  │
│  - Generates execution plan         │
└─────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│        Terraform Providers          │
│  (plugins — downloaded on init)     │
│                                     │
│  aws provider      → AWS API        │
│  google provider   → GCP API        │
│  azurerm provider  → Azure API      │
│  kubernetes provider → K8s API      │
│  github provider   → GitHub API     │
└─────────────────────────────────────┘
      │
      ▼
Cloud APIs (AWS, GCP, Azure, etc.)
```

**The State File (`terraform.tfstate`)**

The state file is a JSON file that records everything Terraform has created. It is the source of truth for "what currently exists."

```json
{
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_bucket",
      "instances": [
        {
          "attributes": {
            "id": "my-unique-bucket-name-2024",
            "arn": "arn:aws:s3:::my-unique-bucket-name-2024",
            "region": "us-east-1"
          }
        }
      ]
    }
  ]
}
```

Critical points about state:
- Terraform reads state to know what currently exists before calculating a plan
- If you delete a resource from your `.tf` file, Terraform sees it in state and knows to destroy it
- Losing the state file = Terraform has no idea what it created = dangerous
- Never manually edit the state file
- In production: always store state remotely (Chapter 7)

---

## 1.7 Terraform Versions and the BSL Change

| Date | Event |
|------|-------|
| 2014 | Terraform 0.1 released by HashiCorp under MPL 2.0 (open-source) |
| 2023-08 | HashiCorp switches to BSL 1.1 — restricts use by "competitive offerings" |
| 2023-09 | OpenTofu fork announced under Linux Foundation |
| 2024-01 | OpenTofu 1.6.0 released — generally available, fully MPL 2.0 |
| Today | Both Terraform and OpenTofu are widely used; syntax is identical |

For internal use (managing your own infrastructure), there is no practical difference. This course uses Terraform. All code examples work unchanged with OpenTofu — just replace `terraform` with `tofu` in commands.

---

## 1.8 The Terraform Workflow in a Team

This is how professional teams manage infrastructure safely. You'll implement this fully in Chapter 14.

```
1. Developer writes or modifies .tf files
            │
            ▼
2. git push → Pull Request opened
            │
            ▼
3. CI pipeline runs: terraform plan
   → Posts plan output as a PR comment
   → Team can see exactly what will change before approving
            │
            ▼
4. Team reviews plan output in PR review
   "Wait — this shows -/+ on the RDS instance, that's a destroy!"
            │
            ▼
5. PR approved and merged
            │
            ▼
6. CI pipeline runs: terraform apply
   → Changes deployed to the target environment
            │
            ▼
7. State updated in remote backend (S3 + DynamoDB locking)
```

This workflow means:
- No one applies Terraform from their laptop
- Every change is reviewed and approved
- The plan shows exactly what will happen — no surprises
- The state is safely stored and locked during operations

---

## Summary

Infrastructure as Code solves the fundamental problems of manual infrastructure: configuration drift, undocumented changes, inability to reproduce environments, and disaster recovery failures.

Terraform is the dominant IaC tool because it is declarative, multi-cloud, has a massive ecosystem, and handles state management automatically. The core loop — write HCL, run plan, apply — is simple, safe, and the foundation of everything else in this course.

---

## Knowledge Check

1. What is configuration drift, and how does IaC prevent it?
2. What is the difference between declarative and imperative IaC? Give one example of each.
3. What does `terraform plan` do, and why is it important to read it before applying?
4. What is the Terraform state file, and what happens if you lose it?
5. Why does Terraform need a "provider" such as the AWS provider?

---

## Hands-On Exercise

Browse the Terraform Registry at [registry.terraform.io](https://registry.terraform.io).

1. Find the official `terraform-aws-modules/vpc/aws` module
2. Read the module's README and identify:
   - What input variables does it accept?
   - What output values does it expose?
   - What AWS resources does it create?
3. Find a second module of your choice (e.g., for EKS, RDS, or EC2)
4. Write down, in plain English, what AWS infrastructure each module would deploy

This gives you intuition for how modules work before you write your own.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-installation-and-setup.md">Next: Installation, Setup & First Resource →</a>
</div>
