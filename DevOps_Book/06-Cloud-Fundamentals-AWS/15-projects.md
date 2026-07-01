# Chapter 15 — Hands-On Projects

## Learning Objectives

By the end of this chapter you will have:
- Deployed a secure, globally distributed static website on S3 + CloudFront
- Built a three-tier scalable web application with ALB, Auto Scaling, and RDS
- Developed a fully serverless REST API with API Gateway, Lambda, DynamoDB, and Cognito
- Constructed a complete CI/CD pipeline from GitHub push to ECS Fargate via GitHub Actions
- Designed a capstone production platform that demonstrates every skill in the course

---

## Project 1 — Static Website on S3 + CloudFront (Beginner)

**Goal:** Deploy a secure, globally distributed static website with HTTPS.

**Architecture:**
```
GitHub → manual upload → S3 (private) → CloudFront (HTTPS) → users globally
Custom domain via Route53 with ACM certificate
```

**Steps:**

1. Create an S3 bucket with **Block Public Access** fully enabled
2. Enable static website hosting on the bucket
3. Request an ACM certificate for your domain in `us-east-1` (required for CloudFront)
4. Create a CloudFront distribution with **Origin Access Control (OAC)** for S3
5. Set up a Route53 **Alias** record pointing to the CloudFront distribution
6. Upload an `index.html` and verify HTTPS access
7. Update the file and test cache invalidation

```bash
# Automated deploy script
#!/bin/bash
set -e

BUCKET="my-website-bucket"
DISTRIBUTION_ID="EDFDVBD6EXAMPLE"

# Build (if using static site generator)
npm run build

# Sync to S3 — cache static assets for 1 year, HTML for 5 minutes
aws s3 sync ./dist s3://$BUCKET/ \
  --delete \
  --cache-control "max-age=31536000" \
  --exclude "*.html"

aws s3 sync ./dist s3://$BUCKET/ \
  --delete \
  --cache-control "max-age=300" \
  --include "*.html" \
  --exclude "*"

# Invalidate CloudFront cache for HTML only
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/*.html"

echo "Deploy complete!"
```

**Success criteria:**
- Website loads over HTTPS
- HTTP automatically redirects to HTTPS
- CloudFront is serving content (check response headers for `x-cache: Hit from cloudfront`)
- Page load time < 500 ms from multiple continents (verify with WebPageTest)

---

## Project 2 — Scalable Web Application (Intermediate)

**Goal:** Three-tier application with ALB, Auto Scaling, RDS, and a complete monitoring setup.

**Architecture:**
```
Internet → Route53 → CloudFront → ALB (HTTPS)
                                        ↓
                                   EC2 ASG (2–10 instances)
                                    ↑            ↓
                                 Secrets      RDS PostgreSQL
                                 Manager      (Multi-AZ)
                                                   ↓
                                           ElastiCache Redis
```

**Step-by-step:**

1. **VPC:** 2 AZs, public/private/data subnets, IGW, NAT Gateway
2. **RDS:** PostgreSQL, Multi-AZ, in data subnets, security group allows app SG only
3. **ElastiCache:** Redis cluster, in data subnets
4. **EC2 AMI:** bake a custom AMI with your application installed (or use user data)
5. **Launch Template:** reference the AMI, instance role with Secrets Manager read access
6. **ASG:** min 2, desired 3, max 10; health check grace period 300 s; target tracking CPU at 60%
7. **ALB:** HTTPS listener, redirect HTTP → HTTPS, health check path `/health`
8. **CloudFront:** ALB as origin, HTTPS only
9. **CloudWatch:** alarms for CPU, error rate, RDS connections; SNS email notifications
10. **Secrets Manager:** DB credentials fetched at application startup via SDK

**Load testing to trigger Auto Scaling:**

```python
# locustfile.py — install locust: pip install locust
from locust import HttpUser, task, between

class WebUser(HttpUser):
    wait_time = between(0.1, 0.5)

    @task
    def homepage(self):
        self.client.get("/")

    @task(3)
    def api_call(self):
        self.client.get("/api/users")

# Run:
# locust --host=https://my-alb-url.com --headless -u 500 -r 50 --run-time 5m
```

**Watch in the AWS Console during the load test:**
- ASG activity tab: instances scaling from 3 to 8+
- ALB: `RequestCount` and `TargetResponseTime` graphs
- RDS: `DatabaseConnections` increasing

---

## Project 3 — Serverless API (Intermediate)

**Goal:** Build a fully serverless REST API with authentication, persistence, and monitoring.

**Architecture:**
```
Client → CloudFront → API Gateway (HTTP API)
                              ↓
                       Lambda function
                       ├── DynamoDB (data)
                       ├── S3 (file uploads via pre-signed URLs)
                       └── SNS (notifications)
Cognito User Pool → JWT → API Gateway authorizer
```

**Build steps:**

1. **DynamoDB table:** `users` with `userId` partition key; enable PITR
2. **Lambda:** CRUD operations (list, get, create, update, delete users)
3. **API Gateway HTTP API:** routes to Lambda, Cognito JWT authorizer on protected routes
4. **Cognito:** User Pool with email verification and password policy
5. **S3:** profile photo uploads via pre-signed URLs generated by Lambda
6. **CloudWatch:** Lambda Insights, custom metrics, alarm on error rate > 1%

```python
# Lambda handler for full CRUD API
import boto3
import json
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')

def handler(event, context):
    method = event['requestContext']['http']['method']
    path = event['requestContext']['http']['path']

    try:
        if method == 'GET' and path == '/users':
            return list_users()
        elif method == 'GET' and path.startswith('/users/'):
            user_id = path.split('/')[-1]
            return get_user(user_id)
        elif method == 'POST' and path == '/users':
            body = json.loads(event.get('body', '{}'))
            return create_user(body)
        elif method == 'DELETE' and path.startswith('/users/'):
            user_id = path.split('/')[-1]
            return delete_user(user_id)
        else:
            return {'statusCode': 404, 'body': json.dumps({'error': 'Not found'})}
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}

def list_users():
    result = table.scan()
    return {'statusCode': 200, 'body': json.dumps(result['Items'])}

def get_user(user_id):
    result = table.get_item(Key={'userId': user_id})
    if 'Item' not in result:
        return {'statusCode': 404, 'body': json.dumps({'error': 'User not found'})}
    return {'statusCode': 200, 'body': json.dumps(result['Item'])}

def create_user(data):
    user = {
        'userId': str(uuid.uuid4()),
        'email': data['email'],
        'name': data['name'],
        'createdAt': datetime.utcnow().isoformat()
    }
    table.put_item(Item=user)
    return {'statusCode': 201, 'body': json.dumps(user)}

def delete_user(user_id):
    table.delete_item(Key={'userId': user_id})
    return {'statusCode': 204, 'body': ''}
```

---

## Project 4 — CI/CD Pipeline to AWS (Advanced)

**Goal:** Complete GitOps pipeline from GitHub push to AWS deployment.

**Pipeline:**
```
GitHub push → GitHub Actions → Build Docker image → Push to ECR
                                     → Update ECS Fargate task definition
                                     → Deploy new task definition
                                     → Wait for service stability
                                     → Notify Slack on success/failure
```

**GitHub Actions workflow:**

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECS_SERVICE: myapp-service
  ECS_CLUSTER: production

permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — no long-lived secrets)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Update ECS service
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          TASK_DEF=$(aws ecs describe-task-definition \
            --task-definition myapp \
            --query taskDefinition)
          NEW_TASK_DEF=$(echo $TASK_DEF | jq \
            --arg IMAGE "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
            '.containerDefinitions[0].image = $IMAGE |
             del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)')
          NEW_TASK_ARN=$(aws ecs register-task-definition \
            --cli-input-json "$NEW_TASK_DEF" \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --task-definition $NEW_TASK_ARN
          aws ecs wait services-stable \
            --cluster $ECS_CLUSTER \
            --services $ECS_SERVICE
```

**IAM role for GitHub Actions using OIDC (no long-lived credentials):**

```bash
aws iam create-role \
  --role-name github-actions-deploy \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:myorg/myapp:ref:refs/heads/main"
        }
      }
    }]
  }'
```

Key insight: the `sub` condition restricts the role to a specific repository **and** branch. A fork or a feature branch cannot assume this role.

---

## Project 5 — Capstone: Multi-Tier Production Platform (Advanced)

**Goal:** Architect and deploy a complete, production-ready platform on AWS.

**Requirements:**
- Highly available: multi-AZ, auto-scaling, no single point of failure
- Secure: private subnets, WAF, GuardDuty, all data encrypted
- Observable: CloudWatch dashboards, alarms, distributed tracing with X-Ray
- Cost-optimised: Savings Plans, Spot instances for non-critical workloads
- IaC: everything in Terraform (Topic 7 extension)

**Components to build:**

| # | Component | Details |
|---|---|---|
| 1 | VPC | 3 AZs, public/private/data subnets, Transit Gateway for future account expansion |
| 2 | ECS Fargate cluster | Runs the API — no EC2 instances to manage or patch |
| 3 | RDS Aurora PostgreSQL | Writer + 2 read replicas, Multi-AZ, automated backups |
| 4 | ElastiCache Redis | Cluster mode enabled, in-transit and at-rest encryption |
| 5 | ALB + WAF | WAF with AWS managed rule groups, rate limiting |
| 6 | CloudFront | Static frontend, custom error pages, security headers |
| 7 | ECR | Image scanning on push, lifecycle policy keeping last 10 images |
| 8 | Secrets Manager | All credentials; automatic rotation for RDS |
| 9 | CloudWatch | Unified dashboard, composite alarms, Container Insights |
| 10 | CI/CD | GitHub Actions → ECR → ECS (from Project 4) |

**Architecture diagram:**

```
                         ┌─────────────────────────────────────────┐
                         │              AWS Cloud                   │
Internet                 │                                          │
   │                     │   CloudFront                             │
   ├──────────────────── │ ──────────────►  S3 (static frontend)   │
   │                     │      │                                   │
   │                     │      ▼                                   │
   └──────────────────── │ ──► WAF ──► ALB (public subnets)        │
                         │               │                          │
                         │               ▼                          │
                         │    ECS Fargate (private subnets)         │
                         │         │            │                   │
                         │         ▼            ▼                   │
                         │   Aurora RDS    ElastiCache Redis        │
                         │  (data subnets) (data subnets)           │
                         │                                          │
                         │   Secrets Manager ◄── application        │
                         │   CloudWatch ◄──── metrics/logs          │
                         └─────────────────────────────────────────┘
```

**This is the project to put in your portfolio.** It demonstrates every skill in this course. Once complete, convert it to Terraform (Topic 7) — you will provision and destroy the entire stack with a single command.

---

## Summary

These five projects mirror real production architectures used at scale:

| Project | Skill demonstrated | Difficulty |
|---|---|---|
| 1 — S3 + CloudFront website | Storage, CDN, DNS, HTTPS | Beginner |
| 2 — Three-tier web app | Compute, networking, databases, Auto Scaling | Intermediate |
| 3 — Serverless API | Lambda, API Gateway, DynamoDB, Cognito | Intermediate |
| 4 — CI/CD pipeline | GitHub Actions, ECR, ECS, OIDC IAM | Advanced |
| 5 — Capstone platform | Everything, multi-service integration, IaC | Advanced |

Work through them in order. Each one builds on the previous. Complete Project 4 before moving to Topic 7 (Terraform) — you will convert this entire stack into Terraform code and learn the value of declarative infrastructure directly by comparison with what you have already built by hand.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="14-best-practices.md">← Previous: Best Practices & Well-Architected</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="16-interview-preparation.md">Next: Interview Preparation →</a>
</div>
