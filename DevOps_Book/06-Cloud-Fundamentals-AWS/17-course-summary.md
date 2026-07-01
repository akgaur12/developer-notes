# Chapter 17 — Course Summary & Next Steps

## 17.1 What You've Learned

Congratulations on completing Cloud Fundamentals (AWS) — Topic 6 of the DevOps Learning Path.

This course covered the AWS services that every DevOps engineer uses every day, from the foundational IAM and VPC building blocks to serverless architecture patterns and production-grade operational practices.

Here is what you now know:

| Chapter | What You Can Do Now |
|---|---|
| 01 Cloud & AWS Intro | Set up an AWS account, understand pricing, use the CLI |
| 02 IAM | Design least-privilege policies, create roles for services, enable MFA |
| 03 EC2 | Launch, configure, and manage virtual machines at scale |
| 04 VPC | Design production-grade network architecture with proper isolation |
| 05 S3 | Store, serve, and secure any file type at any scale |
| 06 RDS | Run managed relational databases with HA and automatic backups |
| 07 ELB & Auto Scaling | Build self-healing, auto-scaling application tiers |
| 08 CloudFront & Route53 | Serve content globally, configure DNS, manage SSL certs |
| 09 CloudWatch | Monitor everything, alert on anomalies, audit all API calls |
| 10 Security | Protect systems with WAF, GuardDuty, KMS, Secrets Manager |
| 11 Serverless | Build event-driven systems without managing servers |
| 12 Additional Services | Use SQS, SNS, ECR, Step Functions, and cost tools |
| 13 CLI & SDK | Automate AWS operations with scripts and code |
| 14 Best Practices | Apply the Well-Architected Framework to real systems |
| 15 Projects | Deploy real production-grade architectures |
| 16 Interview Prep | Answer AWS interview questions confidently |

---

## 17.2 Completion Checklist

Work through this checklist before moving on. If any item feels shaky, go back to the relevant chapter.

```
Core Skills:
  [ ] Can explain the AWS shared responsibility model without notes
  [ ] Can create a production VPC from scratch (subnets, IGW, NAT, routes)
  [ ] Can launch and configure an EC2 instance via CLI only
  [ ] Can configure IAM roles with least-privilege policies
  [ ] Can design a multi-AZ, auto-scaling web application architecture
  [ ] Can set up CloudWatch alarms and SNS notifications
  [ ] Can deploy a Lambda function triggered by S3 and API Gateway
  [ ] Can write boto3 scripts to automate AWS operations

Projects:
  [ ] Completed Project 1: Static website on S3 + CloudFront
  [ ] Completed Project 2: Scalable web application with ALB + ASG + RDS
  [ ] Completed Project 3: Serverless API with Lambda + API Gateway + DynamoDB
  [ ] Completed Project 4: CI/CD pipeline deploying to ECS via GitHub Actions

Interview Readiness:
  [ ] Can answer "Design a HA web application" in full detail
  [ ] Can debug EC2→S3 connectivity issues systematically
  [ ] Know the difference between Multi-AZ and Read Replicas
  [ ] Know when to use SQS vs SNS vs EventBridge
  [ ] Understand IAM evaluation logic (explicit deny always wins)
```

---

## 17.3 AWS Services Quick Reference

```
Compute:
  EC2          — Virtual machines
  Lambda       — Serverless functions (max 15 min timeout)
  ECS Fargate  — Containerised workloads without managing EC2
  Beanstalk    — PaaS for application deployments

Storage:
  S3           — Object storage (unlimited scale, 11 9s durability)
  EBS          — Block storage attached to EC2
  EFS          — Shared NFS filesystem for multiple EC2 instances
  Glacier      — Long-term archive storage

Database:
  RDS          — Managed relational (PostgreSQL, MySQL, MariaDB, Oracle, SQL Server)
  Aurora       — AWS-optimised MySQL/PostgreSQL (up to 5x faster than RDS MySQL)
  DynamoDB     — Serverless NoSQL key-value / document
  ElastiCache  — In-memory caching (Redis / Memcached)

Networking:
  VPC          — Isolated virtual network
  ELB/ALB/NLB  — Load balancers (application / network)
  CloudFront   — CDN (400+ edge locations globally)
  Route53      — DNS + health checks + routing policies
  API Gateway  — REST / WebSocket / HTTP API management

Security:
  IAM          — Identity and access management
  KMS          — Key management and encryption
  Secrets Mgr  — Secrets storage and automatic rotation
  WAF          — Web application firewall
  GuardDuty    — Threat detection and anomaly alerting
  Shield       — DDoS protection (Standard free, Advanced paid)

Observability:
  CloudWatch   — Metrics, logs, alarms, dashboards
  CloudTrail   — API audit log (who did what, when, from where)
  X-Ray        — Distributed tracing for microservices

Messaging:
  SQS          — Message queue (decouple producers from consumers)
  SNS          — Pub/sub notifications (fan-out to multiple subscribers)
  EventBridge  — Event bus and scheduler

Developer Tools:
  CodePipeline — CI/CD orchestration
  CodeBuild    — Managed build environment
  ECR          — Container image registry
  CloudShell   — Browser-based CLI (no local setup needed)
```

---

## 17.4 Common Architecture Patterns

```
Static website:
  Route53 → CloudFront → S3 (private bucket, OAC policy)

REST API (serverless):
  Route53 → API Gateway → Lambda → DynamoDB

REST API (containerised):
  Route53 → CloudFront → ALB → ECS Fargate → Aurora + ElastiCache

CI/CD to AWS:
  GitHub → GitHub Actions (OIDC, no long-lived keys) → ECR → ECS / EKS

Event-driven processing:
  S3 upload → SNS → SQS → Lambda → DynamoDB

Scheduled jobs (replacing cron):
  EventBridge (schedule rule) → Lambda → AWS SDK
```

---

## 17.5 Key Numbers to Remember

```
Lambda:      max 15 min timeout, 10 GB memory, 1,000 default concurrent executions
S3:          11 nines durability, 99.99% availability, objects up to 5 TB
EC2:         t3.micro = 2 vCPU + 1 GB RAM; 750 hr/month on free tier
RDS:         Multi-AZ failover completes in ~60–120 seconds
SQS:         14-day max message retention; 256 KB max message size
Route53:     TTL 300 s (5 min) is typical for most records
CloudFront:  400+ edge locations globally
```

---

## 17.6 What's Next: Topic 7 — Infrastructure as Code (Terraform)

You have just built real AWS infrastructure manually — with the console and the CLI. The problem is now clear: you cannot reproduce it reliably, you cannot review it in Git, and one typo in the console can break production.

**Terraform solves this.** Everything you built in this course — the VPC, the ASG, the ALB, the RDS database — you will declare in code and deploy with a single command. Infrastructure becomes reviewable, testable, version-controlled, and reproducible across environments.

In Topic 7 you will:

- Write HCL (HashiCorp Configuration Language) to declare AWS resources
- Understand Terraform state and why it is the heart of IaC
- Use modules to make infrastructure reusable across teams
- Manage multiple environments (dev / staging / prod) with workspaces
- Store remote state in S3 with DynamoDB locking
- Integrate Terraform into CI/CD pipelines (plan on PR, apply on merge)
- Convert the Project 4 capstone stack into fully-managed Terraform code

---

## 17.7 The Bigger Picture

After Topic 7, you will be ready for Kubernetes. AWS provides EKS (Elastic Kubernetes Service) — a managed Kubernetes control plane. Topics 8 and 9 build on everything here: VPC networking, IAM roles, ECR container registry, ELB for ingress, CloudWatch for monitoring — all the AWS foundations you have now will be used directly in your Kubernetes clusters.

The full DevOps learning path:

```
Topics 1–4:   Local foundations (Linux, Networking, Git, Docker)
Topics 5–6:   Cloud and automation (CI/CD, AWS)        ← you are here
Topics 7–9:   Infrastructure at scale (Terraform, K8s, Advanced K8s)
Topics 10–11: Production operations (Monitoring, Security / DevSecOps)
```

You are now at the halfway point. The second half gets more powerful — and you have everything you need to tackle it.

---

## 17.8 Recommended Resources

### Books

- *AWS Certified Solutions Architect Study Guide* by Ben Piper — best SAA-C03 prep book
- *Amazon Web Services in Action* by Michael Wittig — practical hands-on reference

### AWS Official Resources

- **AWS Documentation** — genuinely excellent; read the "Getting Started" guide for each service
- **AWS Well-Architected Labs** — free hands-on labs for each pillar
- **AWS Free Tier** — keep experimenting; most of this course fits within free tier limits
- **AWS Skill Builder** — official learning platform with hands-on labs and practice exams

### Third-Party

- **A Cloud Guru (ACG)** — video courses, practice exams, cloud sandboxes
- **Tutorials Dojo** — best SAA-C03 practice exam questions (closer to real exam than ACG)
- **CloudCraft** (cloudcraft.co) — diagram AWS architectures visually before you build

---

## 17.9 Your Progress

You have now completed 6 of 11 courses in the DevOps Learning Path.

| # | Course | Status |
|---|---|---|
| 1 | Linux Fundamentals | Complete |
| 2 | Networking Basics | Complete |
| 3 | Git & Version Control | Complete |
| 4 | Docker | Complete |
| 5 | CI/CD Pipelines | Complete |
| 6 | Cloud Fundamentals (AWS) | **Complete — you are here** |
| 7 | Infrastructure as Code (Terraform) | Up next |
| 8 | Kubernetes Basics | Coming soon |
| 9 | Advanced Kubernetes | Coming soon |
| 10 | Monitoring & Logging | Coming soon |
| 11 | Security (DevSecOps) | Coming soon |

Continue to: **[Infrastructure as Code (Terraform) →](../07-Infrastructure-as-Code-Terraform/00-index.md)**

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="16-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../07-Infrastructure-as-Code-Terraform/00-index.md">Next: Infrastructure as Code (Terraform) →</a>
</div>
