# Chapter 16 — Interview Preparation

## Learning Objectives

By the end of this chapter you will be able to:

- Explain core AWS services confidently and concisely under interview pressure
- Work through architecture design scenarios systematically
- Debug common AWS connectivity and performance problems step-by-step
- Answer cost and security questions with professional depth
- Use the STAR method for behavioural questions
- Know which AWS certifications to target and in what order

---

## 16.1 How AWS Interviews Work

AWS DevOps / Cloud Engineer interviews typically have 3–4 rounds:

1. **Phone screen** — 30 min, basic AWS concepts, 2–3 questions
2. **Technical interview** — 60–90 min, architecture design, troubleshooting scenarios
3. **Hands-on / live coding** — AWS console/CLI tasks, or writing IaC
4. **System design** — design a production system on AWS (common at senior level)

What they are really testing:

- Can you explain AWS service concepts clearly (without reading the docs)?
- Have you actually deployed real systems, or just read about it?
- Can you debug and troubleshoot like a professional?
- Do you understand cost, security, and operational concerns — not just "make it work"?

---

## 16.2 Core AWS Concepts — Must Know

### IAM

**Q: "What's the difference between an IAM role and an IAM user?"**

A: A user has permanent credentials (password, access keys) associated with a person. A role has no permanent credentials — it is assumed by services (EC2, Lambda), applications, or other users to get temporary credentials via STS. Roles are the correct mechanism for giving AWS services permissions; users are for humans.

**Q: "How does EC2 access S3 securely without hardcoding credentials?"**

A: Create an IAM role with the required S3 policy, attach it to the EC2 instance profile. The EC2 instance retrieves temporary credentials from IMDS (`169.254.169.254`) automatically. The AWS SDK does this transparently. Never hardcode credentials.

**Q: "What happens if an IAM policy allows something but another policy denies it?"**

A: Explicit DENY always wins, regardless of how many Allow statements exist.

---

### Networking

**Q: "What's the difference between a security group and a NACL?"**

A: Security groups are stateful (return traffic automatically allowed), instance-level, allow-only rules. NACLs are stateless (must explicitly allow both directions), subnet-level, support both allow and deny. Use SGs for most protection; NACLs for subnet-level block rules (like blocking an IP range).

**Q: "A private EC2 instance needs to download packages from the internet. What do you need?"**

A: A NAT Gateway in a public subnet, with the private subnet's route table pointing `0.0.0.0/0` to the NAT Gateway. The NAT Gateway needs an Elastic IP. The public subnet must have an Internet Gateway.

**Q: "What's a VPC endpoint used for?"**

A: Access AWS services (S3, DynamoDB, etc.) from a VPC without going through the internet. Traffic stays within the AWS network. Gateway endpoints (S3, DynamoDB) are free; Interface endpoints (most other services) cost ~$0.01/hr.

---

### High Availability

**Q: "How would you make an EC2 web application highly available?"**

A: Place EC2 instances in multiple AZs using an Auto Scaling Group. Put an Application Load Balancer in front, spanning the same AZs. The ALB health checks remove unhealthy instances. For the database, use RDS Multi-AZ — the standby in a different AZ takes over automatically within 60–120 seconds.

**Q: "What's the difference between RDS Multi-AZ and Read Replicas?"**

A: Multi-AZ uses synchronous replication to a standby in another AZ — for HA/failover; the standby is NOT readable. Read Replicas use asynchronous replication to one or more copies — for scaling read traffic; replicas ARE readable but may have slight lag.

---

## 16.3 Architecture Scenarios

### Scenario: "Design a high-availability web application for 100k users/day"

```
Good answer structure:

1. Compute:      ECS Fargate or EC2 ASG in 2+ AZs, min 2 instances
2. Load balancer: ALB with health checks, HTTPS, HTTP→HTTPS redirect
3. Database:     RDS Aurora PostgreSQL Multi-AZ
                 (or Aurora Serverless if traffic is variable)
4. Caching:      ElastiCache Redis for sessions and hot queries
5. CDN:          CloudFront for static assets + API caching where applicable
6. DNS:          Route53 with health checks and latency routing (if multi-region)
7. Security:     WAF on ALB, security groups locking down inter-tier traffic
8. Monitoring:   CloudWatch alarms + dashboards; SNS alerts to on-call
9. Cost:         Reserved Instances for baseline; Spot for non-critical ASG capacity
```

### Scenario: "Your EC2 instance suddenly can't connect to S3. How do you debug?"

```bash
# Debugging checklist:

# 1. Check IAM: does the instance role have the required S3 policy?
aws iam simulate-principal-policy \
  --policy-source-arn <role-arn> \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# 2. Check bucket policy: is there a policy that denies access?
aws s3api get-bucket-policy --bucket my-bucket

# 3. Check network
#    - Is there a VPC endpoint? Is the endpoint policy too restrictive?
#    - If no endpoint: can the instance reach the internet? (NAT Gateway ok?)

# 4. Check S3 Block Public Access settings
#    For public operations, this might be blocking

# 5. Read the error message carefully:
#    403 Forbidden        → IAM or bucket policy issue
#    No route to host     → network/VPC configuration issue
```

### Scenario: "Your website is slow. CloudWatch shows high CPU on EC2. What do you do?"

```
Immediate:
1. Scale up: manually increase ASG desired capacity
2. Check if autoscaling is working (why didn't it scale automatically?)

Short-term:
3. Is CloudFront caching static assets? Check cache hit ratio.
4. Is ElastiCache reducing DB queries? Check cache hit rate.
5. Are all ALB instances healthy? (no traffic going to crashed instances)

Root cause:
6. Profile the application — is a specific endpoint causing the CPU spike?
7. Check RDS slow query log — is a bad query causing backend delay?
8. Check for DDoS — CloudWatch RequestCount spike + WAF rate rules
```

---

## 16.4 Cost Questions

**Q: "How would you reduce AWS costs in a startup?"**

A: "First I'd enable Cost Explorer and understand where the money goes. Then: rightsize EC2 instances (Compute Optimizer), convert steady workloads to Reserved Instances, use Spot for CI/CD runners and batch jobs, enable S3 Intelligent-Tiering, delete unattached EBS volumes and idle load balancers, and shut down dev environments on weekends."

**Q: "What's the cheapest way to run a simple web application?"**

A: S3 static website + Lambda + DynamoDB can cost ~$0/month on free tier or a few cents for low traffic. No EC2 means no 24/7 idle compute cost.

---

## 16.5 Security Questions

**Q: "An EC2 instance was compromised. What do you do?"**

```
Incident response steps:

1. Isolate:      Modify security group to block all inbound/outbound traffic
2. Snapshot:     Create EBS snapshot for forensics before any other action
3. Investigate:  Review CloudTrail, VPC Flow Logs, CloudWatch Logs for the period
4. Blast radius: What did the role have access to? Rotate any accessed secrets.
5. Terminate:    Terminate the instance — do not try to clean it (it is untrusted)
6. Root cause:   How did it get compromised? (exposed SSH? vulnerable app? stolen key?)
7. Remediate:    Fix the vulnerability, deploy clean instance from a known-good AMI
```

**Q: "How do you prevent an S3 data breach?"**

A: Block Public Access at account and bucket level. Encrypt at rest with SSE-KMS. Use bucket policies to restrict access to specific IAM roles only. Enable versioning (recover from accidental deletes/overwrites). S3 Object Lock for compliance data. GuardDuty monitors for access from malicious IPs. CloudTrail logs all API calls for audit.

---

## 16.6 Behavioural and Situational Questions

Common questions you should have answers prepared for:

- "Tell me about a time you reduced cloud costs significantly."
- "Describe a production incident and how you handled it."
- "How do you keep up with AWS releases?"
- "What's the most complex AWS architecture you've built?"
- "How would you migrate an on-premises application to AWS?"

**Prep tip:** use the STAR method (Situation, Task, Action, Result) for behavioural questions. Have 3–4 real examples ready before the interview.

---

## 16.7 Common Gotchas in Interviews

```
Mistakes that reveal inexperience:
  "I'd just use EC2 for everything"
    → shows unfamiliarity with managed services

  Forgetting about multi-AZ for HA
    → no single point of failure is day-one thinking

  "IAM user with admin access for the app"
    → should be roles, not users

  "I'd SSH in to fix it"
    → production should be immutable; redeploy, don't patch in place

  Not mentioning encryption and security until asked
    → security should be the default framing, not an afterthought

  No mention of cost
    → engineers who don't think about cost aren't ready for production


What impresses interviewers:
  Mentioning trade-offs
    → "ALB is cheaper but NLB gives a static IP if they need that"

  Asking clarifying questions
    → "Is this stateful or stateless? What's the RTO requirement?"

  Knowing service limits
    → "Lambda max is 15 min, so batch jobs over that need ECS or AWS Batch"

  Mentioning operational concerns
    → monitoring, alerting, runbooks, on-call rotation

  Bringing up IaC unprompted
    → "I'd manage this with Terraform so it's reproducible and reviewable"
```

---

## 16.8 Certification Roadmap

| Certification | Level | Focus | Good For |
|---|---|---|---|
| AWS Cloud Practitioner (CLF-C02) | Foundational | Cloud concepts, billing | Non-technical roles, first cert |
| AWS Solutions Architect Associate (SAA-C03) | Associate | Design AWS architectures | Most valuable, best known |
| AWS Developer Associate (DVA-C02) | Associate | Develop/deploy apps on AWS | Developers |
| AWS SysOps Associate (SOA-C02) | Associate | Operations, monitoring | Ops engineers |
| AWS DevOps Engineer Professional (DOP-C02) | Professional | CI/CD, IaC, operations | Senior DevOps |
| AWS Solutions Architect Professional (SAP-C02) | Professional | Complex architectures | Architects |

**Recommended path for DevOps:** SAA-C03 → DOP-C02

Time investment: approximately 6–8 weeks of study per certification.

---

## 16.9 Mock Interview Questions

Practice these out loud before your interview. Set a timer and speak your answer as if talking to an interviewer — not writing notes.

1. Explain VPC, subnets, and how traffic flows from the internet to a private EC2 instance.
2. Design a disaster recovery strategy for an RDS database with RTO = 1 hour and RPO = 15 minutes.
3. Walk me through setting up a CI/CD pipeline that deploys to ECS using GitHub Actions.
4. Your Lambda function is timing out. How do you diagnose and fix it?
5. How does CloudFront improve both performance and security?
6. Explain IAM policy evaluation logic with an example of a deny override.
7. What's the difference between SQS Standard and FIFO queues?
8. How do you estimate AWS costs before deploying a new service?

---

## Summary

AWS interviews test real operational knowledge, not just memorised definitions. Interviewers want to see that you understand trade-offs, think about security and cost by default, and can debug production problems systematically. Architecture scenario questions are your chance to show depth — walk through compute, networking, storage, database, caching, CDN, monitoring, and security in a structured way. Prepare 3–4 real STAR stories for behavioural rounds. The SAA-C03 certification is the single most recognised AWS credential and a natural next step after completing this course.

---

## Knowledge Check

1. What is the difference between a security group and a NACL, and when would you use each?
2. An EC2 instance has an IAM role with S3 read access, but `aws s3 ls` returns a 403. What are three possible causes?
3. How does RDS Multi-AZ differ from a Read Replica? Which one improves availability and which improves read performance?
4. A Lambda function is processing SQS messages but keeps timing out on large payloads. What are two ways to fix this?
5. Describe the STAR method and give an example structure for "Tell me about a time you reduced cloud costs."

---

## Hands-On Exercise

Do a 30-minute mock interview with yourself:

1. Pick 3 architecture scenarios from this chapter.
2. Set a timer for 10 minutes per scenario.
3. Answer each one out loud as if talking to an interviewer — no notes.
4. After each answer, write down what you forgot to mention.
5. Repeat the weakest scenario the next day until you can cover all areas without prompting.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="17-course-summary.md">Next: Course Summary →</a>
</div>
