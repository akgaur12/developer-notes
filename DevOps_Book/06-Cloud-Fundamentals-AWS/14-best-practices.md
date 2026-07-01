# Chapter 14 — Best Practices & Well-Architected Framework

## Learning Objectives

By the end of this chapter you will be able to:
- Describe the six pillars of the AWS Well-Architected Framework and the question each answers
- Apply operational excellence, reliability, and security best practices to a real workload
- Identify the top cost-waste patterns and fix them
- Define a tagging strategy and enforce it with AWS Config
- Understand multi-account strategy and when to use it
- Prevent and detect infrastructure drift

---

## 14.1 AWS Well-Architected Framework

Six pillars of a well-designed AWS system:

| Pillar | Core Concern | Key Question |
|---|---|---|
| Operational Excellence | Operations and improvement | Can we detect and respond to events? |
| Security | Protecting data and systems | Is data protected and access controlled? |
| Reliability | Recovering from failure | Will the system recover automatically? |
| Performance Efficiency | Using resources efficiently | Are we using the right resource types? |
| Cost Optimization | Avoiding unnecessary cost | Are we paying only for what we use? |
| Sustainability | Environmental impact | Are we minimising our carbon footprint? |

**AWS Well-Architected Tool:** free service that guides you through a review against all six pillars and produces a report with prioritised recommendations. Access it in the AWS Console under Management & Governance.

---

## 14.2 Operational Excellence

```
Principles:
✓ Everything as code (infrastructure, runbooks, configuration)
✓ Make frequent, small, reversible changes
✓ Refine operations procedures frequently
✓ Anticipate failure — run pre-mortems
✓ Learn from all operational failures

Practices:
- Use CloudFormation/Terraform for all infrastructure (no manual console changes in prod)
- Tag all resources: Environment, Owner, Project, CostCenter
- Use Parameter Store / Secrets Manager (never hardcode config)
- Enable CloudTrail in all regions
- Create runbooks in Markdown, store in Git
- Use CloudWatch Dashboards for operational visibility
```

---

## 14.3 Reliability

Design for failure — everything will fail eventually.

**Multi-AZ deployments:**
- EC2: spread across 2+ AZs in an ASG
- RDS: Multi-AZ for automatic failover
- ElastiCache: Multi-AZ Redis replication group
- ELB: spans multiple AZs automatically

**Recovery objectives:**
- **RTO** (Recovery Time Objective): max acceptable downtime (e.g., 1 hour)
- **RPO** (Recovery Point Objective): max acceptable data loss (e.g., 1 hour of transactions)

**Backup strategy:**
```
- EC2:        AMI snapshots before changes + EBS snapshots on schedule
- RDS:        automated backups (daily) + manual snapshots before migrations
- S3:         versioning + Cross-Region Replication for critical data
- DynamoDB:   point-in-time recovery (PITR) enabled
```

**Circuit breaker pattern:**

If a dependency (database, external API) is failing, stop sending requests and return a fallback rather than queuing thousands of failed requests that cascade into a total outage.

---

## 14.4 Security Best Practices

```
Identity:
□ Use IAM roles, not long-lived access keys
□ Least privilege — review and reduce over time
□ MFA everywhere
□ Rotate credentials regularly (or use roles to avoid rotation entirely)

Network:
□ Resources in private subnets, exposed only through ALB/API Gateway
□ Security groups as tight as possible (reference SG IDs, not CIDR blocks)
□ VPC Flow Logs enabled for forensics
□ No direct SSH from internet — use AWS Systems Manager Session Manager instead

Data:
□ Encryption at rest on all services (S3, EBS, RDS, DynamoDB)
□ Encryption in transit (TLS everywhere, enforce HTTPS-only on ALB)
□ Secrets in Secrets Manager, not env vars in code
□ Enable S3 versioning on critical buckets

Detection:
□ GuardDuty enabled
□ AWS Config rules running
□ CloudTrail enabled and alerting on suspicious activity
□ AWS Security Hub: unified dashboard aggregating GuardDuty, Config, Inspector findings
```

---

## 14.5 Performance Efficiency

**Selection:**
- **Compute:** instance type matches workload
  - CPU-bound → c-family; memory-bound → r-family; general-purpose → m-family
- **Storage:** gp3 default; io2 only for high-IOPS databases
- **Database:** right engine for the use case (don't use RDS for time-series data)
- **Network:** enhanced networking (ENA) enabled on supported instances

**Review:**
- CloudWatch metrics: are resources over/under-provisioned?
- AWS Compute Optimizer: ML-based instance right-sizing recommendations
- CloudFront: cache hit ratio > 80% for static content?

**Caching:**
- CloudFront for static and cacheable dynamic content
- ElastiCache Redis for database query results and session storage
- DAX (DynamoDB Accelerator) for DynamoDB microsecond latency

---

## 14.6 Cost Optimization

**Visibility:**
- Cost Explorer: understand where money goes by service, tag, or account
- AWS Budgets: alert when costs exceed a threshold
- Tag everything (Environment, Project, Owner) — enables per-project cost attribution

**Right-sizing:**
- Start small, scale up when needed (cheaper than starting large and scaling down)
- AWS Compute Optimizer: identifies over-provisioned EC2, Lambda, EBS
- Shut down dev/test environments nights and weekends (saves ~70% on dev costs)

**Pricing models:**
- Reserved Instances / Savings Plans for predictable workloads (40–66% savings)
- Spot Instances for fault-tolerant batch and CI workloads (up to 90% savings)
- Graviton instances (AWS ARM): same performance, ~20% cheaper (m6g vs m5)

**Top waste items:**

| # | Waste | Notes |
|---|---|---|
| 1 | Stopped EC2 instances | Still paying for attached EBS volumes |
| 2 | Unattached EBS volumes | Status = `available`, paying for nothing |
| 3 | Old EBS snapshots | Accumulate silently; set lifecycle policies |
| 4 | Idle NAT Gateways | $0.045/hr even with 0 traffic |
| 5 | Unused Elastic IPs | $0.005/hr when not attached to a running instance |
| 6 | S3 data not tiered | Old data not moved to Glacier/Intelligent-Tiering |
| 7 | Unused load balancers | Provisioned and forgotten after a project ends |

---

## 14.7 Tagging Strategy

```bash
# Mandatory tags for every resource
aws ec2 create-tags --resources i-1234567890 --tags \
  Key=Environment,Value=production \
  Key=Project,Value=myapp \
  Key=Owner,Value=team-backend \
  Key=CostCenter,Value=CC-1234 \
  Key=ManagedBy,Value=terraform

# Enforce tagging with AWS Config rule
# Rule: required-tags — flags resources missing mandatory tags
```

Recommended minimum tag set:

| Tag Key | Example Values | Purpose |
|---|---|---|
| `Environment` | `production`, `staging`, `dev` | Filter by environment |
| `Project` | `myapp`, `data-pipeline` | Cost attribution per project |
| `Owner` | `team-backend`, `john.doe` | Know who to contact |
| `CostCenter` | `CC-1234` | Finance chargeback |
| `ManagedBy` | `terraform`, `manual` | Drift detection |

---

## 14.8 Multi-Account Strategy

As organisations grow, use AWS Organizations and separate accounts:

```
Management Account (root — billing only, minimal workloads)
├── Security Account (CloudTrail, GuardDuty, Config aggregator)
├── Shared Services Account (ECR, shared tooling, Active Directory)
└── Workloads OU
    ├── Dev Account
    ├── Staging Account
    └── Production Account
```

**Benefits:**
- Blast radius isolation — a compromised dev account cannot touch production
- Separate billing — per-account cost visibility
- Independent permission boundaries — prod can have stricter SCPs

**AWS Control Tower** automates the multi-account setup with landing zone best practices, pre-built guardrails, and an account vending machine.

---

## 14.9 Infrastructure Drift

**Problem:**

An engineer clicks in the console to "fix" something quickly. Now the running infrastructure no longer matches the IaC definition. The next `terraform apply` may revert or break the change unexpectedly.

**Prevention:**

```
□ All changes go through IaC (Terraform/CDK) — no manual console changes in production
□ AWS Config detects drift (compares current state to recorded baseline)
□ Read-only console access for engineers; deploy via CI/CD only
□ Terraform drift detection: run terraform plan in CI daily; alert on unexpected changes
□ Use Service Control Policies (SCPs) to block direct console modifications in prod accounts
```

---

## 14.10 Well-Architected Review Checklist

```
Reliability:
□ Multi-AZ deployment for all stateful services
□ Automated backups tested (can you actually restore from them?)
□ Health checks configured on all ALB target groups
□ Auto Scaling configured with appropriate min/max

Security:
□ No resources with 0.0.0.0/0 inbound except ALB on port 443
□ S3 Block Public Access enabled on all buckets
□ All data encrypted at rest and in transit
□ No hardcoded credentials anywhere in code or config

Cost:
□ Unused resources cleaned up
□ Reserved Instances / Savings Plans for predictable workloads
□ Billing alarm and budget alerts configured
□ Cost allocated with tags

Operations:
□ CloudWatch alarms for key metrics
□ Runbooks exist for common operational tasks
□ CloudTrail enabled
□ Incident response process documented
```

---

## Summary

- The AWS Well-Architected Framework's six pillars give a structured lens for evaluating any system design.
- Operational excellence means all changes go through code, events are detected automatically, and post-incident reviews happen after every failure.
- Reliability requires multi-AZ, tested backups, defined RTOs/RPOs, and circuit breakers for dependencies.
- Security means least-privilege roles, private subnets, encryption everywhere, and secrets in Secrets Manager.
- The top cost optimisation wins are right-sizing, Reserved Instances/Savings Plans, shutting down idle resources, and tagging for attribution.
- A tagging strategy and AWS Config enforcement are prerequisites for mature cost and compliance management.
- Multi-account structures isolate blast radius and simplify billing; AWS Control Tower automates the setup.
- Infrastructure drift is prevented by treating the console as read-only and running IaC plans on a schedule.

---

## Knowledge Check

1. Which Well-Architected pillar is primarily concerned with **data loss and recovery time** after a failure?
2. What is the difference between RTO and RPO? Give a concrete example of each.
3. Name three specific resource types where **encryption at rest** should be enabled in every production workload.
4. You find that your AWS bill has doubled. List three AWS tools you would use to investigate and attribute the increase.
5. A developer made a manual change in the AWS Console to unblock a production incident. What should happen next, and how would you prevent it from becoming a recurring pattern?

---

## Hands-On Exercise

1. Open the **AWS Well-Architected Tool** in the AWS Console and create a new workload review for a real or imaginary application.
2. Complete the questionnaire for at least two pillars (Security and Reliability recommended).
3. Identify the top 3 **High Risk Items** the tool flags.
4. For each High Risk Item, document:
   - What the risk is
   - The recommended remediation
   - Whether the fix is already in place or needs action
5. Implement a fix for at least one High Risk Item and verify the tool updates the risk count.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="13-aws-cli-and-sdk.md">← Previous: AWS CLI & SDK</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="15-projects.md">Next: Hands-On Projects →</a>
</div>
