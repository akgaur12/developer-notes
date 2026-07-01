# Chapter 10 — AWS Security Services

## Learning Objectives

By the end of this chapter you will be able to:
- Explain the AWS Shared Responsibility Model and know which party owns which layer
- Store and retrieve secrets securely using Secrets Manager and Parameter Store
- Encrypt and decrypt data with KMS
- Protect web applications with WAF and Shield
- Enable threat detection with GuardDuty
- Audit configuration compliance with AWS Config
- Apply a security hardening checklist to a new AWS account

---

## 10.1 AWS Shared Responsibility Model

```
AWS is responsible FOR the cloud:
├── Physical data centres (locks, guards, power)
├── Hardware (servers, switches, storage)
├── Hypervisor and virtualisation layer
└── Managed service infrastructure (RDS OS patches, Lambda runtime)

You are responsible IN the cloud:
├── IAM (who can access what)
├── Operating system (patches on EC2)
├── Application code and vulnerabilities
├── Data encryption (at rest and in transit)
├── Network security (VPC, security groups, NACLs)
└── Correct service configuration (S3 public access, security group rules)
```

The key insight: AWS secures the infrastructure you run on; you secure what you run on that infrastructure. If you misconfigure an S3 bucket as public, that is your responsibility — not AWS's.

---

## 10.2 AWS Secrets Manager

Never hardcode credentials. Store them in Secrets Manager.

```bash
# Store a secret
aws secretsmanager create-secret \
  --name prod/database/password \
  --secret-string '{"username":"dbadmin","password":"SuperSecret123!"}'

# Retrieve in your application
aws secretsmanager get-secret-value \
  --secret-id prod/database/password \
  --query SecretString \
  --output text | jq -r .password

# Auto-rotate: Secrets Manager can rotate RDS passwords automatically
aws secretsmanager rotate-secret \
  --secret-id prod/database/password \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123:function:rotate-db-password \
  --rotation-rules AutomaticallyAfterDays=30
```

In application code (boto3):

```python
import boto3, json

def get_db_password():
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId='prod/database/password')
    secret = json.loads(response['SecretString'])
    return secret['password']
```

---

## 10.3 AWS Systems Manager Parameter Store

Lighter-weight alternative to Secrets Manager for non-sensitive config:

```bash
# Store parameter (free tier: Standard parameters)
aws ssm put-parameter \
  --name /myapp/prod/db-host \
  --value "prod-db.xxxxx.us-east-1.rds.amazonaws.com" \
  --type String

# Store encrypted secret (uses KMS)
aws ssm put-parameter \
  --name /myapp/prod/api-key \
  --value "sk-supersecretkey" \
  --type SecureString

# Retrieve
aws ssm get-parameter \
  --name /myapp/prod/api-key \
  --with-decryption \
  --query Parameter.Value \
  --output text
```

**When to use which:**

| Use Case | Service |
|---|---|
| DB passwords, API keys needing rotation | Secrets Manager |
| Config values, feature flags, non-rotating values | Parameter Store |
| Free tier config storage | Parameter Store (Standard) |

---

## 10.4 AWS KMS — Key Management Service

```bash
# Create a customer-managed key (CMK)
aws kms create-key \
  --description "My application encryption key" \
  --key-usage ENCRYPT_DECRYPT

# Create an alias
aws kms create-alias \
  --alias-name alias/myapp-key \
  --target-key-id 12345678-1234-1234-1234-123456789012

# Encrypt data
aws kms encrypt \
  --key-id alias/myapp-key \
  --plaintext "my secret data" \
  --output text \
  --query CiphertextBlob | base64 --decode > encrypted.bin

# Decrypt
aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --output text \
  --query Plaintext | base64 --decode
```

KMS is used under the hood by: S3 SSE-KMS, EBS encryption, RDS encryption, Secrets Manager, and Parameter Store SecureString.

Always use `alias/` names in code — never hardcode key IDs. Aliases are stable even if you rotate the underlying key.

---

## 10.5 AWS WAF — Web Application Firewall

WAF protects ALB, CloudFront, and API Gateway from common web attacks:

- SQL injection
- Cross-site scripting (XSS)
- Bad bots and scrapers
- Rate limiting (DoS mitigation)
- Geo-blocking

```bash
# Create a rate-based rule (block IPs making > 2000 requests in 5 min)
aws wafv2 create-web-acl \
  --name prod-waf \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "RateLimitRule",
      "Priority": 1,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimitRule"
      }
    }
  ]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=prod-waf

# Associate with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:... \
  --resource-arn arn:aws:elasticloadbalancing:...
```

**AWS Managed Rule Groups**: pre-built rule sets for OWASP Top 10, bot control, and known bad IPs — enable with one click in the console. Start with these before writing custom rules.

---

## 10.6 AWS Shield — DDoS Protection

| Tier | Cost | Coverage |
|---|---|---|
| Shield Standard | Free | Layer 3/4 DDoS protection — automatic for all AWS customers |
| Shield Advanced | $3,000/month | Layer 7 protection, DDoS cost protection, 24/7 DRT access |

If you use CloudFront + ALB + Route 53, you already have Shield Standard coverage. Shield Advanced is for businesses where downtime has significant financial impact.

---

## 10.7 AWS GuardDuty — Threat Detection

GuardDuty analyses CloudTrail logs, VPC Flow Logs, and DNS logs without requiring any agents.

```bash
# Enable GuardDuty
aws guardduty create-detector --enable

# GuardDuty finding examples:
# UnauthorizedAccess:EC2/SSHBruteForce    — SSH brute force on your EC2
# Recon:IAMUser/MaliciousIPCaller         — IAM calls from known malicious IP
# CryptoCurrency:EC2/BitcoinTool          — EC2 instance mining Bitcoin
# Exfiltration:S3/MaliciousIPCaller       — S3 access from known threat actor

# List findings
aws guardduty list-findings \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)
```

Cost: approximately $4/month for most accounts. Enable it in every account — it is too cheap not to.

---

## 10.8 AWS Config — Compliance and Configuration History

AWS Config records all configuration changes to AWS resources and evaluates them against compliance rules.

```bash
# Built-in managed rules for common compliance checks:
#
# s3-bucket-public-access-prohibited  — All S3 buckets blocking public access?
# encrypted-volumes                   — All EBS volumes encrypted?
# root-account-mfa-enabled            — MFA enabled on root account?
# restricted-ssh                      — Security groups not allowing 0.0.0.0/0 on port 22?

# Enable Config recorder
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123:role/config-role \
  --recording-group allSupported=true,includeGlobalResourceTypes=true
```

Config also maintains a timeline of resource changes, so you can answer "what changed, and when?" during an incident.

---

## 10.9 Security Checklist for New AWS Accounts

Apply this checklist every time you set up a new account or review an existing one:

```
IAM:
[ ] MFA on root account
[ ] No root account access keys
[ ] IAM password policy (min 12 chars, require complexity)
[ ] Never use root for daily work

Networking:
[ ] No 0.0.0.0/0 inbound on port 22 (SSH) in any SG
[ ] No 0.0.0.0/0 inbound on port 3389 (RDP) in any SG
[ ] Production resources in private subnets

Data:
[ ] S3 Block Public Access enabled on all buckets
[ ] EBS default encryption enabled per region
[ ] RDS encrypted at rest
[ ] CloudTrail enabled and logging to S3

Monitoring:
[ ] GuardDuty enabled
[ ] AWS Config enabled with security rules
[ ] Billing alarm set
[ ] CloudWatch alarms for key metrics
```

---

## Summary

- The **Shared Responsibility Model** defines a clear line: AWS secures the infrastructure; you secure what runs on it.
- Use **Secrets Manager** for credentials that require automatic rotation; use **Parameter Store** for non-secret config and feature flags.
- **KMS** underpins encryption across S3, EBS, RDS, and other services — always reference keys by alias.
- **WAF** blocks application-layer attacks; **Shield** defends against network-layer DDoS.
- **GuardDuty** provides continuous threat detection with zero agent overhead for around $4/month.
- **AWS Config** tracks resource configuration history and flags compliance drift automatically.

---

## Knowledge Check

1. Under the Shared Responsibility Model, who is responsible for patching the operating system on an EC2 instance?
2. What is the difference between Secrets Manager and Parameter Store SecureString?
3. Why should you use `alias/myapp-key` instead of a raw key ID in application code?
4. What layer of attacks does Shield Standard protect against, and what does Shield Advanced add?
5. GuardDuty detected `CryptoCurrency:EC2/BitcoinTool` on one of your instances. What does this mean and what should you do?

---

## Hands-on Exercise

1. Enable GuardDuty in your AWS account and review the Findings dashboard.
2. Store a database password in Secrets Manager with the path `dev/database/password`. Retrieve it from the CLI.
3. Configure a WAF Web ACL with a rate-limiting rule (2000 requests per 5 minutes) and associate it with an ALB.
4. Enable AWS Config and activate the `restricted-ssh` and `s3-bucket-public-access-prohibited` managed rules. Review the compliance report.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="09-cloudwatch.md">← Previous: CloudWatch & Observability</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="11-serverless.md">Next: Serverless (Lambda & API Gateway) →</a>
</div>
