# Chapter 1 — Introduction to Cloud & AWS

## Learning Objectives

By the end of this chapter you will be able to:

- Explain cloud computing and its five essential characteristics
- Compare IaaS, PaaS, SaaS, and FaaS service models
- Describe AWS regions, availability zones, and edge locations
- Set up an AWS free-tier account securely with MFA and billing alerts
- Configure the AWS CLI and run your first API call

---

## 1.1 What Is Cloud Computing?

**Before the cloud**: organisations purchased physical servers, racked them in a data centre, managed the hardware lifecycle, and massively overprovisioned for peak traffic that arrived only occasionally.

**Cloud computing**: rent compute, storage, and networking from a provider on-demand, pay only for what you use, and scale up or down in minutes instead of months.

The US National Institute of Standards and Technology (NIST) defines five essential characteristics:

| Characteristic | Meaning |
|----------------|---------|
| On-demand self-service | Provision resources instantly, no human interaction with the provider |
| Broad network access | Access from any device over the internet |
| Resource pooling | Provider serves multiple tenants from shared infrastructure |
| Rapid elasticity | Scale up or down automatically to match demand |
| Measured service | Usage is monitored, metered, and billed transparently |

---

## 1.2 Cloud Service Models

| Model | You Manage | Provider Manages | Examples |
|-------|-----------|-----------------|---------|
| **IaaS** — Infrastructure as a Service | OS, runtime, apps | Hardware, hypervisor, network | EC2, GCE, Azure VMs |
| **PaaS** — Platform as a Service | App code, data | OS, runtime, scaling | Heroku, App Engine, Elastic Beanstalk |
| **SaaS** — Software as a Service | Nothing (just use it) | Everything | Gmail, Salesforce, Slack |
| **FaaS** — Functions as a Service | Function code | Runtime, scaling, infra | Lambda, Cloud Functions |

The rule of thumb: moving from IaaS → PaaS → SaaS, you give up control but gain simplicity and speed. Most DevOps work lives at IaaS and PaaS.

---

## 1.3 Cloud Deployment Models

- **Public cloud** — AWS, GCP, Azure. Multi-tenant, on-demand, no upfront hardware cost.
- **Private cloud** — Your own data centre with cloud-like APIs (VMware vSphere, OpenStack). Full control, full cost.
- **Hybrid cloud** — On-premises systems connected to a public cloud via Direct Connect or VPN. Common during migrations.
- **Multi-cloud** — Using two or more cloud providers. Avoids vendor lock-in; adds operational complexity.

---

## 1.4 Why AWS?

- **Market leader**: 33% cloud market share (2024), more than GCP and Azure combined.
- **Breadth**: 200+ services across compute, storage, networking, AI/ML, IoT, and more.
- **Global reach**: 33 geographic regions, 105 availability zones (as of 2024).
- **Ecosystem**: every major DevOps tool (GitHub Actions, Terraform, Kubernetes, Datadog) has native AWS integration.
- **Free tier**: 12 months of limited free usage — enough to complete every exercise in this course.
- **Certifications**: AWS certs are the most in-demand cloud credentials globally. SAA-C03 is the gold standard entry point.

---

## 1.5 AWS Global Infrastructure

Understanding the physical layout of AWS is essential for designing highly available and low-latency applications.

### Regions

A **region** is a geographically isolated area containing multiple data centres. Each region is completely independent — a failure in us-east-1 does not affect eu-west-1.

Common regions:
- `us-east-1` — N. Virginia (oldest, largest, cheapest)
- `us-west-2` — Oregon
- `eu-west-1` — Ireland
- `ap-southeast-1` — Singapore
- `ap-northeast-1` — Tokyo

**Choosing a region**: pick the region closest to your users unless compliance or cost requirements dictate otherwise. Not all services are available in all regions.

### Availability Zones (AZs)

An **AZ** is one or more discrete data centres within a region, each with independent power, cooling, and networking. AZs in a region are connected by low-latency private fibre.

Deploying across multiple AZs is the primary mechanism for high availability in AWS.

```
Region: us-east-1 (N. Virginia)
├── AZ: us-east-1a  ← deploy primary resources here
├── AZ: us-east-1b  ← replicate here for HA
├── AZ: us-east-1c
└── AZ: us-east-1d
```

### Edge Locations

**Edge locations** are CDN points-of-presence used by CloudFront to cache content close to end users. There are 400+ edge locations globally — far more than regions. When a user in Mumbai requests your website hosted in us-east-1, CloudFront serves cached content from the nearest edge location.

### Local Zones

**Local Zones** extend AWS infrastructure into metro areas (e.g., Los Angeles, Dallas) for single-digit millisecond latency to end users in those cities. Used for latency-sensitive workloads like gaming and live streaming.

---

## 1.6 Setting Up Your AWS Account

Follow these steps to create a secure account before any hands-on work.

### Step 1 — Create the account

Go to [aws.amazon.com](https://aws.amazon.com) and sign up. You need an email address, a phone number, and a credit card. AWS will not charge you while you stay within free-tier limits.

### Step 2 — Enable MFA on the root account immediately

The root account has unrestricted access to everything. Compromise of the root account means full loss of control of your AWS environment.

1. Sign in as root → Account menu → Security credentials
2. Under "Multi-factor authentication (MFA)" → Assign MFA device
3. Use a virtual MFA app (Google Authenticator, Authy) or a hardware key

### Step 3 — Create a billing alarm

Never be surprised by an unexpected AWS bill.

1. AWS Console → Billing → Billing preferences → Enable billing alerts
2. CloudWatch → Alarms → Create alarm → Billing → Total Estimated Charge
3. Set threshold: $10 (or whatever your budget is)
4. Send alert to your email via SNS

### Step 4 — Create an IAM admin user

Never use the root account for daily work. Create a dedicated IAM admin user.

1. IAM → Users → Create user
2. Add to the `Administrators` group (attach `AdministratorAccess` managed policy)
3. Enable console access with a strong password
4. Enable MFA on this user too
5. Generate access keys for CLI use (store them securely)

### Step 5 — Install and configure the AWS CLI

```bash
# Install AWS CLI v2 on Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
# aws-cli/2.x.x Python/3.x.x ...

# Configure with your IAM user's access keys
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json

# Verify — returns your IAM user ARN
aws sts get-caller-identity
```

---

## 1.7 AWS Pricing Model

AWS pricing has several dimensions. Understanding them prevents bill shock.

### Pricing options for EC2 (and similar compute)

| Option | Cost | Commitment | Best For |
|--------|------|-----------|---------|
| On-Demand | Highest hourly rate | None | Variable or unpredictable workloads, learning |
| Reserved Instances | 40–75% cheaper | 1 or 3 years | Steady-state production workloads |
| Spot Instances | Up to 90% cheaper | None (can be interrupted) | Batch jobs, CI runners, stateless services |
| Savings Plans | ~66% cheaper | Flexible spend commitment | Mix of instance types/regions |

### Free Tier highlights (sufficient for this entire course)

| Service | Free Tier Allowance | Duration |
|---------|-------------------|----------|
| EC2 | 750 hours/month t2.micro | 12 months |
| S3 | 5 GB storage, 20,000 GET, 2,000 PUT requests | 12 months |
| RDS | 750 hours/month db.t2.micro | 12 months |
| Lambda | 1 million invocations/month | Permanent |
| CloudWatch | 10 custom metrics, 10 alarms | Permanent |
| DynamoDB | 25 GB storage, 25 RCU, 25 WCU | Permanent |

### Cost estimation tools

- **AWS Pricing Calculator** — estimate costs before you build: [calculator.aws](https://calculator.aws)
- **Cost Explorer** — visualise and analyse your actual spend
- **Budgets** — set spend thresholds and receive alerts
- **Trusted Advisor** — cost optimisation recommendations

---

## 1.8 AWS Console vs CLI vs SDK vs IaC

You will interact with AWS in four different ways throughout this course and your career.

| Method | Best For | Example |
|--------|---------|---------|
| **Console** (web UI) | Learning, one-off tasks, visual exploration | Click to launch EC2 |
| **AWS CLI** | Scripting, automation, daily operator work | `aws ec2 describe-instances` |
| **SDK** (boto3, JS, Go, Java) | Application integration, programmatic access | Python script creating an S3 bucket |
| **IaC** (Terraform, CDK, CloudFormation) | Reproducible, version-controlled infrastructure | Declare an entire VPC in code |

This course focuses on the **Console** for understanding and the **CLI** for automation. Topic 7 (Infrastructure as Code — Terraform) goes deep on IaC.

---

## Summary

- Cloud computing delivers on-demand compute, storage, and networking with pay-as-you-go pricing.
- AWS is the market leader with 200+ services, 33 regions, and 105 AZs.
- Regions are independent geographic areas; AZs are redundant data centres within a region. Deploy across multiple AZs for high availability.
- Always secure the root account with MFA, create a dedicated IAM admin user, and set a billing alarm before doing anything else.
- The AWS CLI is your primary tool for automation; configure it with `aws configure`.

---

## Knowledge Check

1. What are the five NIST essential characteristics of cloud computing?
2. What is the difference between IaaS and PaaS? Give an AWS example of each.
3. Why should you deploy resources across multiple AZs?
4. What is the difference between a Reserved Instance and a Spot Instance?
5. Why should you never use the AWS root account for daily work?

---

## Hands-On Exercise

Complete the following before moving to Chapter 2:

1. Create an AWS free-tier account at [aws.amazon.com](https://aws.amazon.com).
2. Enable MFA on the root account using Google Authenticator or Authy.
3. Create a CloudWatch billing alarm at $10 (or your chosen budget).
4. Create an IAM admin user, add it to an `Administrators` group, and enable MFA.
5. Generate CLI access keys for your IAM admin user.
6. Install AWS CLI v2 on your local machine.
7. Run `aws configure` with your access key, `us-east-1`, and `json` output.
8. Run `aws sts get-caller-identity` — confirm you see your IAM user ARN, not root.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-iam.md">Next: IAM — Identity & Access Management →</a>
</div>
