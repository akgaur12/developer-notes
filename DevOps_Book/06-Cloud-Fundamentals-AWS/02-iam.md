# Chapter 2 — IAM — Identity & Access Management

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the difference between IAM users, groups, and roles
- Write IAM policies in JSON with correct Effect, Action, and Resource fields
- Apply the policy evaluation logic (explicit deny always wins)
- Create roles for EC2 instances and cross-account access
- List the IAM security best practices expected in production environments

---

## 2.1 What Is IAM?

**IAM (Identity and Access Management)** is the AWS service that controls who can do what on which AWS resources. Every API call made to AWS — from the console, CLI, or SDK — is authenticated and authorised through IAM.

Key facts:
- **Global service** — IAM is not region-specific; users, groups, and roles are available across all regions.
- **Free** — there is no charge for IAM usage.
- **Foundation** — every other AWS service depends on IAM for access control. Getting IAM right is the most important security task in any AWS account.

---

## 2.2 IAM Principals: Users, Groups, Roles

### Users

An IAM **user** represents an individual person or an application that needs long-term credentials (password for the console, or access keys for the CLI/SDK).

```bash
# Create a user
aws iam create-user --user-name alice

# List all users
aws iam list-users

# Create CLI access keys for the user
aws iam create-access-key --user-name alice
# Returns: AccessKeyId and SecretAccessKey — store the secret; you cannot retrieve it again
```

### Groups

A **group** is a collection of IAM users. Attach policies to the group, and all members inherit those permissions. This is the correct way to manage permissions for teams.

```bash
# Create a group
aws iam create-group --group-name developers

# Add a user to the group
aws iam add-user-to-group --user-name alice --group-name developers

# Attach a managed policy to the group
aws iam attach-group-policy \
  --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
```

### Roles

A **role** is an IAM identity that can be **assumed** by a trusted entity (an AWS service, another account, a federated user, or even another IAM user). Roles issue **temporary credentials** via STS — no long-lived access keys needed.

Common role use cases:

| Use Case | Who Assumes the Role | Why |
|----------|---------------------|-----|
| EC2 instance role | The EC2 instance | Access S3, Secrets Manager, etc. without hardcoded keys |
| Lambda execution role | The Lambda function | Write logs to CloudWatch, read from DynamoDB |
| Cross-account role | Another AWS account | Allow Account B to access resources in Account A |
| CI/CD OIDC role | GitHub Actions / GitLab CI | Deploy to AWS from pipelines without storing static keys |
| Developer assume-role | IAM user in same account | Temporary elevation to a higher-privilege role |

---

## 2.3 IAM Policies

Policies are JSON documents that define permissions. They are attached to users, groups, or roles.

### Policy structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadWrite",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

| Field | Description |
|-------|-------------|
| `Version` | Always `"2012-10-17"` — the current policy language version |
| `Sid` | Optional statement identifier for documentation |
| `Effect` | `"Allow"` or `"Deny"` |
| `Action` | AWS API call(s) — e.g., `"s3:PutObject"`, `"ec2:*"`, `"*"` |
| `Resource` | ARN of the target resource, or `"*"` for all |
| `Condition` | Optional — restrict by IP, MFA status, time, tags, etc. |

### Policy types

| Type | Description | When to Use |
|------|-------------|-------------|
| **AWS Managed** | Pre-built by AWS, maintained by AWS | Quick starts, common roles |
| **Customer Managed** | You create and control | Custom permissions, reusable across multiple identities |
| **Inline** | Embedded directly in one user/group/role | Avoid — hard to audit, cannot be reused |

### Useful AWS managed policies

- `AdministratorAccess` — full access to everything (use only for break-glass accounts)
- `PowerUserAccess` — full access except IAM management
- `ReadOnlyAccess` — read-only across all services
- `AmazonS3ReadOnlyAccess` — read-only on S3
- `AmazonEC2FullAccess` — full control of EC2

### Condition example — require MFA

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```

This deny overrides any allow when the user has not authenticated with MFA — a standard pattern for protecting sensitive actions.

---

## 2.4 Policy Evaluation Logic

AWS evaluates all applicable policies before allowing or denying a request. The logic:

```
1. Is there an explicit DENY in any policy?  → DENY  (always wins — cannot be overridden)
2. Is there an explicit ALLOW in any policy? → ALLOW
3. Neither?                                  → DENY  (implicit deny — default)
```

**Practical implication**: if 10 policies grant `s3:DeleteObject` and one policy denies it, the request is DENIED. Deny always wins. Use denies sparingly and intentionally.

---

## 2.5 IAM Roles in Practice

### Creating a role for EC2 to access S3

```bash
# Step 1: Create the role with a trust policy that allows EC2 to assume it
aws iam create-role \
  --role-name ec2-s3-read \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Step 2: Attach the S3 read-only managed policy
aws iam attach-role-policy \
  --role-name ec2-s3-read \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Step 3: Create an instance profile (required wrapper for EC2)
aws iam create-instance-profile --instance-profile-name ec2-s3-read

# Step 4: Add the role to the instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-s3-read \
  --role-name ec2-s3-read

# Step 5: Launch EC2 with the instance profile attached
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --iam-instance-profile Name=ec2-s3-read \
  --key-name my-keypair \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678
```

Once the EC2 instance is running with this profile, any code on that instance can call `aws s3 ls` or use the AWS SDK — the instance automatically receives rotating temporary credentials via the metadata service. No access keys needed in the code.

### Cross-account role assumption

```bash
# In Account A: create a role that Account B can assume
# Trust policy:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT_B_ID:root"},
    "Action": "sts:AssumeRole"
  }]
}

# In Account B: assume the role to get temporary credentials
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_A_ID:role/cross-account-role \
  --role-session-name my-session

# Returns: AccessKeyId, SecretAccessKey, SessionToken (expire in 1 hour by default)
```

---

## 2.6 IAM Best Practices

These are the practices AWS and security auditors check for in production environments:

```
□ Never use the root account for daily work — create IAM users or use SSO
□ Enable MFA on the root account AND all IAM users with console access
□ Grant least privilege — start with read-only, add write only when needed
□ Use roles for applications and services — never hardcode access keys in code
□ Rotate access keys every 90 days (or eliminate them in favour of roles/SSO)
□ Organise users into groups; assign permissions to groups, not individuals
□ Enable CloudTrail to audit every API call across all regions
□ Use IAM Access Analyzer to detect resources shared with external entities
□ Use AWS Config to detect IAM drift and policy violations
□ Never commit access keys to source code — use Secrets Manager or parameter store
□ Set a strong password policy: minimum 14 chars, MFA required, 90-day rotation
```

---

## 2.7 AWS Security Token Service (STS)

STS issues temporary security credentials — the engine behind IAM roles.

```bash
# Assume a role and get temporary credentials
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/my-role \
  --role-session-name my-session \
  --duration-seconds 3600

# Example response (abbreviated):
# {
#   "Credentials": {
#     "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
#     "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
#     "SessionToken": "AQoDYXdzEJr...",
#     "Expiration": "2024-01-01T12:00:00Z"
#   }
# }

# Check who you are currently authenticated as
aws sts get-caller-identity
# {
#   "UserId": "AIDAIOSFODNN7EXAMPLE",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/alice"
# }
```

Temporary credentials automatically expire — a major security advantage over long-lived access keys.

---

## 2.8 IAM Identity Center (SSO)

**IAM Identity Center** (formerly AWS SSO) provides centralised single sign-on for AWS organisations. It is the recommended approach for teams and companies.

How it works:
1. Connect your company's identity provider — Okta, Azure AD, Google Workspace, or the built-in directory.
2. Define permission sets (bundles of IAM policies).
3. Assign users or groups to AWS accounts with specific permission sets.
4. Users log in with their company credentials at `https://<your-org>.awsapps.com/start`.
5. No IAM users required — no access keys to rotate.

Benefits over individual IAM users:
- Centralised user lifecycle management (add/remove from IdP, access automatically updated)
- Single audit trail across all accounts
- Temporary credentials only
- Works with AWS Organizations for multi-account environments

---

## 2.9 Resource-Based Policies

Some AWS resources support their own attached policies, separate from IAM identity policies. These define who can access the resource, including cross-account principals.

Examples:
- **S3 bucket policy** — controls access to a specific bucket and its objects
- **Lambda resource policy** — controls who can invoke a specific function
- **SQS queue policy** — controls who can send to or receive from a queue
- **KMS key policy** — controls who can use or manage a specific encryption key

```json
// S3 bucket policy — allow cross-account read
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::999888777666:root"
    },
    "Action": [
      "s3:GetObject",
      "s3:ListBucket"
    ],
    "Resource": [
      "arn:aws:s3:::my-shared-bucket",
      "arn:aws:s3:::my-shared-bucket/*"
    ]
  }]
}
```

For cross-account access: **both** the identity policy (in Account B) and the resource policy (in Account A) must grant access. The effective permission is the intersection.

---

## Summary

- IAM controls authentication and authorisation for every AWS API call.
- Users are long-lived identities; roles issue temporary credentials — prefer roles for applications.
- Policies are JSON documents with Effect, Action, and Resource. Explicit deny always overrides allow.
- Default is implicit deny — nothing is permitted unless explicitly allowed.
- Best practices: least privilege, MFA everywhere, roles instead of access keys, CloudTrail for auditing.
- IAM Identity Center (SSO) is the recommended approach for teams; it eliminates long-lived IAM user access keys.

---

## Knowledge Check

1. What is the difference between an IAM user and an IAM role?
2. In IAM policy evaluation, what happens when there is both an explicit Allow and an explicit Deny for the same action?
3. What is an instance profile and why is it needed for EC2?
4. Why should applications use IAM roles instead of hardcoded access keys?
5. What does STS do, and what kind of credentials does it return?

---

## Hands-On Exercise

Complete the following before moving to Chapter 3:

1. Create an IAM user named `s3-read-test` with no console access and no group memberships.
2. Attach a custom inline policy that allows `s3:GetObject` and `s3:ListBucket` on a specific bucket ARN only.
3. Generate CLI access keys for `s3-read-test`.
4. Configure a second CLI profile: `aws configure --profile s3-test`.
5. Test: `aws s3 ls s3://your-bucket --profile s3-test` (should work).
6. Test: `aws s3 cp s3://your-bucket/file.txt . --profile s3-test` (should work).
7. Test: `aws s3 rm s3://your-bucket/file.txt --profile s3-test` (should fail — Access Denied).
8. Create the `ec2-s3-read` role from section 2.5, including the instance profile.
9. Enable MFA on your IAM admin user if not already done.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./01-introduction.md">← Previous: Introduction to Cloud & AWS</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./03-ec2.md">Next: EC2 — Elastic Compute Cloud →</a>
</div>
