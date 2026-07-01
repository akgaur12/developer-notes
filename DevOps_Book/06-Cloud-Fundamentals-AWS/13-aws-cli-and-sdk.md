# Chapter 13 — AWS CLI & SDK

## Learning Objectives

By the end of this chapter you will be able to:
- Install and configure the AWS CLI with multiple named profiles
- Filter and transform CLI output using JMESPath `--query` expressions
- Handle paginated results correctly
- Automate AWS tasks with boto3 (Python) using clients, resources, paginators, and waiters
- Use the AWS SDK from other languages (JavaScript, Go)
- Understand when to move from CLI scripting to Infrastructure as Code

---

## 13.1 AWS CLI Deep Dive

```bash
# AWS CLI v2 — installation recap
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version

# Multiple profiles (different accounts or regions)
aws configure --profile dev
aws configure --profile prod

# Use a specific profile
aws s3 ls --profile prod
export AWS_PROFILE=prod   # set for the session

# Named profiles in ~/.aws/config
[profile dev]
region = us-east-1
output = json

[profile prod]
region = us-east-1
role_arn = arn:aws:iam::999888777:role/prod-admin
source_profile = default
```

---

## 13.2 CLI Output Formats and Filtering

```bash
# Output formats
aws ec2 describe-instances --output json    # default — full JSON
aws ec2 describe-instances --output text    # tab-separated, good for scripts
aws ec2 describe-instances --output table   # human-readable table
aws ec2 describe-instances --output yaml    # YAML

# --query: JMESPath expressions to filter/project output
# Get only instance IDs and states
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,PublicIpAddress]' \
  --output table

# Get running instance IDs only
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[?State.Name==`running`].InstanceId' \
  --output text

# Get the latest AMI ID for Amazon Linux 2
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text
```

---

## 13.3 CLI Pagination

```bash
# Many commands return paginated results
# --no-paginator returns only the first page (wrong!)
# Correct: use --paginator or pipe through aws paginator

# Method 1: auto-pagination (default with AWS CLI v2)
aws s3api list-objects-v2 --bucket my-bucket

# Method 2: manual pagination
aws ec2 describe-instances \
  --max-items 10 \
  --starting-token <NextToken from previous call>

# Method 3: pipe all pages through jq
aws s3api list-objects-v2 --bucket my-bucket \
  --query 'Contents[*].Key' \
  --output text
```

---

## 13.4 Useful CLI One-Liners

```bash
# Get your current account ID
aws sts get-caller-identity --query Account --output text

# List all running EC2 instances with Name tag
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].{Name:Tags[?Key==`Name`]|[0].Value,ID:InstanceId,Type:InstanceType,IP:PublicIpAddress}' \
  --output table

# List all S3 buckets with their sizes
for bucket in $(aws s3 ls | awk '{print $3}'); do
  echo -n "$bucket: "
  aws s3 ls s3://$bucket --recursive --summarize | grep "Total Size"
done

# Delete all stopped EC2 instances (CAREFUL!)
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=stopped" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text | \
  xargs aws ec2 terminate-instances --instance-ids

# Find all unattached EBS volumes (paying for nothing)
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[*].[VolumeId,Size,VolumeType,CreateTime]' \
  --output table

# List Lambda functions with their runtimes
aws lambda list-functions \
  --query 'Functions[*].[FunctionName,Runtime,MemorySize,Timeout]' \
  --output table
```

---

## 13.5 CLI Environment Variables

```bash
# Override config with env vars (useful in CI/CD)
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json

# In CI/CD (GitHub Actions): set as secrets, not env vars in code
# Never commit AWS credentials to git
```

---

## 13.6 AWS SDK — boto3 (Python)

```python
# Install: pip install boto3

import boto3

# Session with profile
session = boto3.Session(profile_name='prod')
ec2 = session.client('ec2', region_name='us-east-1')

# List running instances
response = ec2.describe_instances(
    Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
)

for reservation in response['Reservations']:
    for instance in reservation['Instances']:
        name = next(
            (tag['Value'] for tag in instance.get('Tags', []) if tag['Key'] == 'Name'),
            'unnamed'
        )
        print(f"{instance['InstanceId']} | {instance['InstanceType']} | {name}")

# Resource abstraction (higher-level, more Pythonic)
s3 = boto3.resource('s3')
bucket = s3.Bucket('my-bucket')
for obj in bucket.objects.all():
    print(obj.key, obj.size)

# Upload with progress callback
import os
def upload_progress(bytes_transferred):
    print(f"\rUploaded {bytes_transferred} bytes", end="")

s3.upload_file(
    'large-file.zip',
    'my-bucket',
    'backups/large-file.zip',
    Callback=upload_progress
)
```

---

## 13.7 boto3 Patterns for DevOps Automation

```python
# Pattern 1: Paginator (never miss results)
import boto3

ec2 = boto3.client('ec2')
paginator = ec2.get_paginator('describe_instances')

all_instances = []
for page in paginator.paginate(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]):
    for reservation in page['Reservations']:
        all_instances.extend(reservation['Instances'])

print(f"Total running instances: {len(all_instances)}")

# Pattern 2: Waiter (wait for resource to reach desired state)
ec2.start_instances(InstanceIds=['i-1234567890abcdef0'])
waiter = ec2.get_waiter('instance_running')
waiter.wait(InstanceIds=['i-1234567890abcdef0'])
print("Instance is running")

# Pattern 3: Resource tagging automation
def tag_untagged_instances():
    ec2 = boto3.resource('ec2')
    for instance in ec2.instances.all():
        tags = {t['Key']: t['Value'] for t in (instance.tags or [])}
        if 'Environment' not in tags:
            instance.create_tags(Tags=[
                {'Key': 'Environment', 'Value': 'unknown'},
                {'Key': 'AutoTagged', 'Value': 'true'}
            ])
            print(f"Tagged {instance.id}")

# Pattern 4: Error handling
import botocore

try:
    s3 = boto3.client('s3')
    s3.get_object(Bucket='my-bucket', Key='nonexistent.txt')
except botocore.exceptions.ClientError as e:
    error_code = e.response['Error']['Code']
    if error_code == 'NoSuchKey':
        print("File not found")
    elif error_code == '403':
        print("Access denied")
    else:
        raise
```

---

## 13.8 AWS SDK in Other Languages

```javascript
// JavaScript / Node.js (AWS SDK v3)
import { EC2Client, DescribeInstancesCommand } from "@aws-sdk/client-ec2";

const client = new EC2Client({ region: "us-east-1" });
const command = new DescribeInstancesCommand({
    Filters: [{ Name: "instance-state-name", Values: ["running"] }]
});
const response = await client.send(command);
```

```go
// Go
import (
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

client := s3.NewFromConfig(cfg)
result, err := client.ListBuckets(ctx, &s3.ListBucketsInput{})
```

---

## 13.9 AWS CloudShell

- Browser-based shell in the AWS Console — pre-authenticated, no setup needed
- AWS CLI v2 pre-installed, Python + boto3, Node.js, git
- 1 GB persistent storage per region
- Great for: quick one-off tasks, accessing AWS from any machine, learning

**Access:** AWS Console → top navigation bar → CloudShell icon (`>_`)

---

## 13.10 Infrastructure as Code Preview

What you can do with CLI scripting:

```bash
#!/bin/bash
# Deploy a complete web app stack with bash + CLI
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --query Subnet.SubnetId --output text)
# ... 100 more lines ...
```

**Problem:** imperative, not idempotent, hard to manage changes. This is why Terraform (Topic 7) exists — declare what you want, Terraform figures out the plan and executes it.

---

## Summary

- The AWS CLI supports multiple named profiles, rich `--query` (JMESPath) filtering, and four output formats.
- Always use auto-pagination or paginators in scripts — never assume one page is enough.
- boto3 offers both `client` (low-level, full API) and `resource` (high-level, Pythonic) abstractions.
- Use **paginators** to handle large result sets and **waiters** to block until a resource reaches its target state.
- AWS SDKs exist for every major language; credential resolution order is the same across all of them (env vars → `~/.aws/credentials` → IAM role).
- CloudShell gives you a pre-authenticated terminal anywhere without local setup.
- CLI scripting is a stepping stone; complex infrastructure belongs in Terraform or CDK.

---

## Knowledge Check

1. What JMESPath expression retrieves only the `InstanceId` of instances where `State.Name` equals `running`?
2. What happens if you call `aws ec2 describe-instances` without handling pagination in a large account?
3. What is the difference between `boto3.client('s3')` and `boto3.resource('s3')`?
4. How does a boto3 **waiter** differ from writing your own polling loop?
5. Why should you never set `AWS_ACCESS_KEY_ID` in a GitHub Actions workflow file?

---

## Hands-On Exercise

Write a boto3 script that:

1. Lists all EC2 instances **across all regions** (use `ec2.describe_regions()` first).
2. Uses **paginators** correctly in every describe call.
3. Outputs a cost-saving report with two sections:
   - Stopped instances that still have attached EBS volumes (you are paying for the volumes).
   - Unattached EBS volumes (`status = available`).
4. Formats the report as a tab-aligned table printed to stdout.

Stretch goal: add a `--dry-run` flag and a `--delete` flag so the script can optionally terminate stopped instances and delete unattached volumes.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="12-additional-services.md">← Previous: Additional AWS Services</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="14-best-practices.md">Next: Best Practices & Well-Architected →</a>
</div>
