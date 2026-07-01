# Infrastructure as Code (Terraform) — Complete Course Index

> **DevOps Learning Path — Topic 7 of 11**

---

## Course Overview

Terraform is the industry-standard tool for managing cloud infrastructure as code. Instead of clicking in the AWS console or writing fragile bash scripts, you declare your infrastructure in HCL (HashiCorp Configuration Language), and Terraform calculates what needs to change and applies it safely.

This course takes you from zero Terraform knowledge to running production infrastructure pipelines. You'll manage the entire AWS stack from Topic 6 — VPCs, EC2, RDS, ALB — in code.

---

## What You'll Be Able to Do

- Write HCL to declare any AWS resource
- Understand Terraform state and manage it safely
- Build reusable modules for your organisation
- Manage multiple environments (dev/staging/prod) without code duplication
- Store remote state in S3 with DynamoDB locking
- Integrate Terraform into CI/CD pipelines (plan on PR, apply on merge)
- Use Terraform Cloud for team collaboration
- Write and run Terraform tests
- Follow industry best practices for large-scale IaC

---

## Prerequisites

- **Cloud Fundamentals AWS (Topic 6)** — you need to know what AWS resources you're managing
- **Linux Fundamentals (Topic 1)** — assumed
- **Git & Version Control (Topic 3)** — assumed

---

## Estimated Learning Timeline

**3–4 weeks**

---

## Learning Milestones

| Milestone | Chapters | Skills Unlocked |
|-----------|----------|-----------------|
| Foundations | 01–03 | Understand IaC concepts, write basic HCL, manage state |
| Core Skills | 04–07 | Variables, modules, remote backends, environments |
| AWS Mastery | 08–12 | Provision full AWS stacks in Terraform |
| Professional | 13–17 | Advanced HCL, CI/CD, testing, best practices, interview ready |

---

## Full Chapter Index

| # | Chapter | Topics |
|---|---------|--------|
| 01 | [Introduction to IaC & Terraform](./01-introduction.md) | IaC concepts, Terraform vs alternatives, how Terraform works, the core loop |
| 02 | [Installation, Setup & First Resource](./02-installation-and-setup.md) | Install Terraform, project structure, providers, first S3 bucket, state basics |
| 03 | [HCL Fundamentals](./03-hcl-fundamentals.md) | Syntax, block types, references, type constraints, built-in functions, conditionals |
| 04 | [Providers & Resources](./04-providers-and-resources.md) | Provider ecosystem, resource lifecycle, meta-arguments, depends_on, count, for_each |
| 05 | [Variables, Outputs & Data Sources](./05-variables-outputs-data.md) | Input variables, output values, data sources, local values, tfvars files |
| 06 | [State Management](./06-state-management.md) | State file deep dive, terraform state commands, import, moving state, sensitive state |
| 07 | [Remote Backends & State Locking](./07-remote-backends.md) | S3 backend, DynamoDB locking, partial configuration, state migration |
| 08 | [Modules](./08-modules.md) | Module structure, inputs/outputs, public registry, versioning, calling modules |
| 09 | [Workspaces & Environments](./09-workspaces-and-environments.md) | Terraform workspaces, environment patterns, workspace vs directory layout |
| 10 | [Building AWS Networking with Terraform](./10-aws-networking.md) | VPC, subnets, routing, NAT gateway, security groups, full network stack |
| 11 | [Building AWS Compute with Terraform](./11-aws-compute.md) | EC2, AMI data sources, user data, ALB, Auto Scaling Groups, launch templates |
| 12 | [Building AWS Databases with Terraform](./12-aws-databases.md) | RDS, subnet groups, parameter groups, secrets management, backups |
| 13 | [Advanced HCL: Loops, Conditionals & Dynamic Blocks](./13-advanced-hcl.md) | for expressions, count vs for_each, dynamic blocks, templatefile, locals patterns |
| 14 | [CI/CD Integration & Atlantis](./14-cicd-integration.md) | GitHub Actions workflows, plan on PR, apply on merge, Atlantis, Terraform Cloud |
| 15 | [Best Practices & Patterns](./15-best-practices.md) | Naming conventions, tagging strategy, module design, secrets handling, drift detection |
| 16 | [Hands-On Projects](./16-projects.md) | Project 1: Three-tier AWS architecture. Project 2: Multi-environment setup. Project 3: Full CI/CD pipeline |
| 17 | [Course Summary](./17-course-summary.md) | Recap, what's next, interview prep, further reading |

---

## DevOps Roadmap Series

| # | Topic | Status |
|---|-------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/) | ✅ Complete |
| 2 | [Networking Basics](../02-Networking-Basics/) | ✅ Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/) | ✅ Complete |
| 4 | Docker | ✅ Complete |
| 5 | CI/CD Pipelines | ✅ Complete |
| 6 | [Cloud Fundamentals AWS](../06-Cloud-Fundamentals-AWS/) | ✅ Complete |
| 7 | **Infrastructure as Code (Terraform)** | 📍 You are here |
| 8 | Kubernetes Basics | Coming soon |
| 9 | Monitoring & Observability | Coming soon |
| 10 | Security & Compliance | Coming soon |
| 11 | Advanced DevOps Patterns | Coming soon |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction.md">Next: Introduction to IaC & Terraform →</a>
</div>
