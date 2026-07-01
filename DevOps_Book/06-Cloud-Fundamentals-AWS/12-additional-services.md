# Chapter 12 — Additional AWS Services

## Learning Objectives

By the end of this chapter you will be able to:
- Decouple applications with SQS queues and dead-letter queues
- Fan out events to multiple consumers with SNS
- Route AWS and custom events with EventBridge rules
- Orchestrate multi-step workflows with Step Functions
- Push and pull container images with ECR
- Build a CI/CD pipeline with CodePipeline and CodeBuild
- Deploy applications without managing infrastructure using Elastic Beanstalk
- Add user authentication to an application with Cognito
- Analyse and control AWS spending with Cost Explorer and Budgets
- Identify and raise service quota limits before hitting them

---

## 12.1 Amazon SQS — Simple Queue Service

SQS is a fully managed message queue that decouples producers from consumers. Messages sit in the queue until a consumer reads and deletes them.

**Queue types:**

| Type | Delivery | Ordering | Use Case |
|---|---|---|---|
| Standard | At-least-once | Best-effort | High-throughput workloads |
| FIFO | Exactly-once | Strict | Payments, order processing |

```bash
# Create standard queue
aws sqs create-queue \
  --queue-name job-queue \
  --attributes VisibilityTimeout=300,MessageRetentionPeriod=86400

# Send message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/job-queue \
  --message-body '{"jobId": "abc123", "type": "resize-image", "key": "uploads/photo.jpg"}'

# Receive message (makes it invisible for 300 s — long enough to process it)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/job-queue

# Delete after successful processing
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/job-queue \
  --receipt-handle "AQEBwJnKyrHigUMZj..."
```

**SQS + Lambda**: Lambda polls the queue automatically, processes messages in batches, and deletes them on success.

**Dead-letter queue (DLQ)**: messages that fail processing N times are moved to a DLQ so you can investigate without losing them.

---

## 12.2 Amazon SNS — Fan-Out Pattern

SNS is a managed pub/sub service. One publish to a topic delivers to all subscribers simultaneously.

```
SNS Topic
├── SQS Queue A   (async processing — team A)
├── SQS Queue B   (async processing — team B)
├── Lambda        (real-time side effect)
└── Email / SMS   (human notifications)
```

SNS + SQS fan-out is a common pattern: SNS delivers to multiple SQS queues, and each SQS queue buffers messages independently for its consumer. If one consumer is slow or down, it does not affect the others.

---

## 12.3 Amazon EventBridge

EventBridge is a serverless event bus that routes events between AWS services, SaaS apps, and your own applications. It replaces CloudWatch Events and extends them to third-party sources.

```bash
# Rule: when an EC2 instance terminates → invoke a Slack notification Lambda
aws events put-rule \
  --name ec2-terminate-notify \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"],
    "detail": {"state": ["terminated"]}
  }'

# Rule: scheduled — runs every hour
aws events put-rule \
  --name hourly-report \
  --schedule-expression "rate(1 hour)"
```

EventBridge also supports **Pipes** (point-to-point event routing with filtering and transformation) and **Schemas** (auto-discovery and documentation of event structures).

---

## 12.4 AWS Step Functions

Step Functions orchestrates multi-step workflows as a visual state machine. It coordinates Lambda functions, ECS tasks, SQS, SNS, and AWS SDK calls with built-in retry, error handling, parallel execution, and wait states.

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:validate-order",
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:charge-payment",
      "Retry": [{"ErrorEquals": ["Lambda.ServiceException"], "MaxAttempts": 3}],
      "Catch": [{"ErrorEquals": ["PaymentFailed"], "Next": "HandlePaymentFailure"}],
      "Next": "FulfillOrder"
    },
    "FulfillOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:fulfill-order",
      "End": true
    },
    "HandlePaymentFailure": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:notify-customer",
      "End": true
    }
  }
}
```

Use Step Functions when you need to chain more than two or three Lambda functions and want visibility into execution history, retry logic, and error branches without embedding that logic in application code.

---

## 12.5 Amazon ECR — Elastic Container Registry

ECR is a managed private Docker registry integrated with IAM, ECS, EKS, and Lambda container images.

```bash
# Create private registry
aws ecr create-repository --repository-name myapp

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Build, tag, and push image
docker build -t myapp .
docker tag myapp:latest \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Scan image for OS and library vulnerabilities
aws ecr start-image-scan \
  --repository-name myapp \
  --image-id imageTag=latest

aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=latest \
  --query 'imageScanFindings.findingSeverityCounts'
```

Enable ECR lifecycle policies to automatically delete old or untagged images and keep storage costs under control.

---

## 12.6 AWS CodePipeline & CodeBuild

AWS-native CI/CD tools. Many teams use GitHub Actions (Chapter 5), but CodePipeline integrates directly with ECR, ECS, Elastic Beanstalk, and CloudFormation.

**buildspec.yml** — placed in the project root, it tells CodeBuild what to run:

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
  build:
    commands:
      - docker build -t $ECR_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker push $ECR_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - echo "Image pushed to ECR"
```

A typical CodePipeline has three stages: Source (GitHub or CodeCommit) → Build (CodeBuild) → Deploy (ECS, Beanstalk, or CloudFormation).

---

## 12.7 AWS Elastic Beanstalk

Elastic Beanstalk is a PaaS layer over EC2, ALB, Auto Scaling Groups, and optionally RDS. You deploy application code; Beanstalk provisions and manages the infrastructure.

Supported platforms: Python, Node.js, Java, Go, .NET, PHP, Ruby, Docker.

```bash
# Install the EB CLI
pip install awsebcli

# Initialise and deploy
eb init my-app --platform python-3.12 --region us-east-1
eb create prod-env
eb deploy

# Rolling deploy (zero-downtime)
eb deploy --timeout 30
```

Beanstalk is a good fit for teams that want to focus on application code and accept some limitations on infrastructure control. For full control, use ECS or EC2 directly.

---

## 12.8 Amazon Cognito — Auth as a Service

Cognito provides managed user authentication: signup, login, MFA, social login (Google, Facebook), and OAuth2/OIDC.

```bash
# Create a User Pool
aws cognito-idp create-user-pool \
  --pool-name my-app-users \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 12,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true
    }
  }' \
  --mfa-configuration OPTIONAL \
  --auto-verified-attributes email
```

**How it fits into an API:**

1. User logs in → Cognito issues a JWT (ID token + access token).
2. Client sends the JWT in the `Authorization` header.
3. API Gateway validates the JWT against Cognito — no custom auth code needed.

Cognito handles user storage, password hashing, MFA, forgot-password flows, and token refresh. For most applications, implementing this yourself is not worth it.

---

## 12.9 AWS Cost Explorer

```bash
# Query monthly costs broken down by service
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[*].[Keys[0],Metrics.BlendedCost.Amount]' \
  --output table
```

**Top cost-saving levers:**

1. Rightsize EC2 instances — use AWS Compute Optimizer for recommendations
2. Delete unused EBS volumes and snapshots
3. Use Reserved Instances or Savings Plans for steady-state workloads (up to 72% discount)
4. Enable S3 Intelligent-Tiering for infrequently accessed data
5. Delete idle NAT Gateways (each costs ~$32/month whether or not traffic flows)
6. Audit forgotten resources: unattached Elastic IPs, old load balancers, stale RDS snapshots

**AWS Budgets**: set a monthly spend threshold and receive alerts — or automatically stop resources — when the budget is exceeded.

---

## 12.10 AWS Service Quotas

Every AWS account has default limits on how many resources you can create per region. These are soft limits and can be raised.

```bash
# View current quotas for EC2
aws service-quotas list-service-quotas --service-code ec2

# Request a quota increase
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --desired-value 100
```

**Limits to know before they surprise you:**

| Service | Default Limit | Impact |
|---|---|---|
| EC2 On-Demand vCPUs | 32 per region | Hit quickly on any real project |
| Lambda concurrent executions | 1,000 per region | Shared across all functions |
| SES outbound email | 200/day (sandbox) | Request production access early |
| VPCs per region | 5 | Easy to exhaust in multi-env setups |

Request increases at least a week before you need them — AWS approval is not instant.

---

## Summary

- **SQS** decouples producers and consumers; pair it with a DLQ to capture failed messages without data loss.
- **SNS** fans a single publish out to multiple SQS queues, Lambda functions, and notification endpoints simultaneously.
- **EventBridge** routes AWS service events and custom events to targets with pattern-matching rules — the recommended replacement for cron + shell scripts.
- **Step Functions** visualises and coordinates multi-step workflows with built-in retry and error handling.
- **ECR** stores private Docker images with IAM-based access control and built-in vulnerability scanning.
- **CodePipeline + CodeBuild** provide AWS-native CI/CD tightly integrated with ECR, ECS, and CloudFormation.
- **Elastic Beanstalk** abstracts infrastructure management for teams that want PaaS simplicity.
- **Cognito** offloads user authentication entirely — no password hashing, token management, or MFA code to write.
- Review **Cost Explorer** monthly and act on Compute Optimizer recommendations before the bill grows.
- Request **Service Quota** increases proactively — default EC2 and Lambda limits are lower than most production workloads need.

---

## Knowledge Check

1. What is the difference between an SQS Standard queue and a FIFO queue? When would you choose FIFO?
2. You publish one message to an SNS topic. How many SQS queues subscribed to that topic will receive it?
3. Describe the fan-out pattern: why use SNS + SQS instead of publishing directly to multiple SQS queues?
4. What problem does Step Functions solve that chaining raw Lambda functions does not?
5. Your application sends transactional email but is hitting the SES sandbox limit of 200 emails/day. What must you do?

---

## Hands-on Exercise

1. Create an SQS queue with a dead-letter queue (max receives = 3).
2. Send 5 test messages to the queue from the CLI.
3. Create a Lambda function that reads from the queue in batches of 5 and logs the message body to CloudWatch.
4. Force one message to fail (raise an exception on a specific `jobId`) and verify it lands in the DLQ after 3 attempts.
5. Push a Docker image to ECR with vulnerability scanning enabled and review the scan findings.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="11-serverless.md">← Previous: Serverless (Lambda & API Gateway)</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="13-aws-cli-and-sdk.md">Next: AWS CLI & SDK →</a>
</div>
