# Chapter 11 — Serverless (Lambda & API Gateway)

## Learning Objectives

By the end of this chapter you will be able to:
- Explain what serverless means and when to choose it over EC2
- Deploy, configure, and invoke Lambda functions from the CLI
- Connect Lambda to event sources: S3, EventBridge, SQS, and API Gateway
- Build an HTTP API with API Gateway and Lambda
- Understand cold starts and apply best practices to minimise their impact
- Use Lambda Layers to share libraries across functions
- Run containers without managing servers using ECS Fargate

---

## 11.1 What Is Serverless?

Serverless means you write code and AWS manages the servers, OS, runtime, scaling, and availability. You pay only for execution time — no idle cost (versus EC2, which runs and bills 24/7).

Lambda is the core serverless compute service. It runs your function in response to an event and then stops.

**Why serverless matters for DevOps:**

- Automate AWS operations by responding to events (CloudWatch alarm, S3 upload, CloudTrail event)
- Build glue code between services (S3 → Lambda → DynamoDB)
- Create APIs without managing web servers
- Replace cron jobs with EventBridge scheduled rules

Note: "serverless" does not mean no servers exist — AWS runs servers; you just do not manage them.

---

## 11.2 Lambda Fundamentals

```python
# handler.py — the simplest Lambda function
import json

def handler(event, context):
    print(f"Received event: {json.dumps(event)}")

    # event:   the trigger payload (S3 notification, API Gateway request, etc.)
    # context: runtime info (function name, memory, remaining time)

    name = event.get('name', 'World')
    return {
        'statusCode': 200,
        'body': json.dumps({'message': f'Hello, {name}!'})
    }
```

**Lambda limits to know:**

| Limit | Value |
|---|---|
| Max execution time | 15 minutes |
| Memory | 128 MB – 10 GB |
| Deployment package | 50 MB zipped, 250 MB unzipped |
| Container image size | Up to 10 GB |
| Concurrent executions | 1,000 per region (soft limit, raiseable) |
| /tmp storage | 512 MB – 10 GB |

More memory also means more CPU — Lambda does not let you configure CPU directly.

---

## 11.3 Deploying Lambda

```bash
# Package and deploy
zip function.zip handler.py

aws lambda create-function \
  --function-name my-api \
  --runtime python3.12 \
  --handler handler.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::123456789:role/lambda-execution-role \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables='{DB_HOST=prod-db.xxxxx.rds.amazonaws.com}'

# Update code
zip function.zip handler.py
aws lambda update-function-code \
  --function-name my-api \
  --zip-file fileb://function.zip

# Invoke manually
aws lambda invoke \
  --function-name my-api \
  --payload '{"name": "Alice"}' \
  output.json
cat output.json

# Stream logs in real time (CloudWatch Logs: /aws/lambda/my-api)
aws logs tail /aws/lambda/my-api --follow
```

---

## 11.4 Lambda Triggers (Event Sources)

| Trigger | When Lambda Fires | Use Case |
|---|---|---|
| S3 | Object created/deleted | Image processing, ETL, notifications |
| API Gateway | HTTP request | REST APIs, webhooks |
| EventBridge (scheduled) | Cron expression | Replacing cron jobs, scheduled tasks |
| DynamoDB Streams | Record inserted/updated | Replicate to other systems, audit log |
| SNS | Message published | Fan-out processing |
| SQS | Messages in queue | Batch processing, decoupling |
| CloudWatch Events | AWS service events | Respond to EC2 state change |
| ALB | HTTP request | Higher throughput than API GW for HTTP |

```bash
# EventBridge scheduled rule: run every day at 2:00 AM UTC
aws events put-rule \
  --name daily-cleanup \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED

aws events put-targets \
  --rule daily-cleanup \
  --targets Id=1,Arn=arn:aws:lambda:us-east-1:123:function:cleanup-function

# Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
  --function-name cleanup-function \
  --statement-id EventBridgeSchedule \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:123:rule/daily-cleanup
```

---

## 11.5 API Gateway

```bash
# Create HTTP API (simpler, cheaper than REST API)
aws apigatewayv2 create-api \
  --name my-api \
  --protocol-type HTTP

# Create Lambda integration
aws apigatewayv2 create-integration \
  --api-id abc123 \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:us-east-1:123:function:my-api \
  --payload-format-version 2.0

# Create route: POST /users → Lambda
aws apigatewayv2 create-route \
  --api-id abc123 \
  --route-key "POST /users" \
  --target integrations/xyz789

# Deploy to a stage
aws apigatewayv2 create-stage \
  --api-id abc123 \
  --stage-name prod \
  --auto-deploy

# Resulting endpoint:
# https://abc123.execute-api.us-east-1.amazonaws.com/prod/users
```

**REST API vs HTTP API:**

| Feature | REST API | HTTP API |
|---|---|---|
| Cost | Higher | ~70% cheaper |
| Latency overhead | ~6 ms | ~1 ms |
| Features | Full (request validation, caching, usage plans) | Lightweight |
| OIDC/OAuth2 | Via Cognito authorizer | Native support |
| Best for | Enterprise APIs with complex requirements | Most modern APIs |

Default to HTTP API unless you need REST API-specific features.

---

## 11.6 Lambda with S3 — Image Processing Example

```python
import boto3
from PIL import Image
import io

s3 = boto3.client('s3')
DEST_BUCKET = 'my-thumbnails-bucket'

def handler(event, context):
    for record in event['Records']:
        src_bucket = record['s3']['bucket']['name']
        src_key    = record['s3']['object']['key']

        # Download original image
        response   = s3.get_object(Bucket=src_bucket, Key=src_key)
        image_data = response['Body'].read()

        # Resize to 200x200
        img = Image.open(io.BytesIO(image_data))
        img.thumbnail((200, 200))

        # Upload thumbnail
        buffer = io.BytesIO()
        img.save(buffer, format='JPEG')
        buffer.seek(0)

        thumb_key = f"thumbnails/{src_key}"
        s3.put_object(
            Bucket=DEST_BUCKET,
            Key=thumb_key,
            Body=buffer,
            ContentType='image/jpeg'
        )
        print(f"Created thumbnail: s3://{DEST_BUCKET}/{thumb_key}")
```

This pattern — S3 event triggers Lambda, Lambda processes and writes to another bucket — is one of the most common serverless patterns in production.

---

## 11.7 Lambda Layers

A Layer is a zip archive of shared libraries or utilities that multiple functions can reference. It avoids duplicating large dependencies in every function package.

```bash
# Package Pillow as a layer
mkdir -p python/lib/python3.12/site-packages
pip install Pillow -t python/lib/python3.12/site-packages/
zip -r pillow-layer.zip python/

aws lambda publish-layer-version \
  --layer-name pillow \
  --zip-file fileb://pillow-layer.zip \
  --compatible-runtimes python3.12

# Attach layer to function
aws lambda update-function-configuration \
  --function-name my-image-processor \
  --layers arn:aws:lambda:us-east-1:123:layer:pillow:1
```

Each function can reference up to 5 layers. AWS also publishes official layers (e.g., the AWS SDK for newer runtimes).

---

## 11.8 Lambda Cold Starts

A cold start occurs when Lambda initialises a new execution environment — either on the first invocation or after an idle period. This adds 100 ms to 1 second of latency before your handler runs.

Code outside `handler()` runs during initialisation, not on every invocation. Use this to your advantage:

```python
import boto3

# Runs once per container (cold start) — NOT per invocation
dynamodb = boto3.resource('dynamodb')
table    = dynamodb.Table('users')

def handler(event, context):
    # Fast — reuses the already-connected table client
    response = table.get_item(Key={'userId': event['userId']})
    return response['Item']
```

**Strategies to reduce cold start impact:**

| Strategy | Detail |
|---|---|
| Move init outside handler | Connection pools, config loading, SDK clients |
| Choose a faster runtime | Python and Node.js cold-start faster than Java |
| Reduce package size | Smaller zip = faster init |
| Provisioned Concurrency | Pre-warms N environments — eliminates cold starts, costs extra |

---

## 11.9 Serverless Architectures

**Traditional:**
```
Browser → ALB → EC2 fleet → RDS
Problem: 24/7 cost even at 3 AM with 0 traffic
```

**Serverless:**
```
Browser → CloudFront → S3 (static files)
         → API Gateway → Lambda → DynamoDB
Cost: $0/month at 0 traffic; scales to millions of requests automatically
```

**For DevOps automation:**
```
EventBridge → Lambda → AWS SDK calls
Replace 90% of cron-based scripts with zero-maintenance serverless functions
```

Lambda's free tier is 1 million requests and 400,000 GB-seconds per month — most automation workloads fit entirely within the free tier.

---

## 11.10 Amazon ECS & Fargate (Container Serverless)

Lambda is for short-lived functions (maximum 15 minutes). For longer-running workloads or existing Docker containers, use ECS Fargate — containers without managing servers.

```bash
# Run a container task with Fargate
aws ecs run-task \
  --cluster prod-cluster \
  --task-definition my-app:5 \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={
    subnets=[subnet-private-1a],
    securityGroups=[sg-app],
    assignPublicIp=DISABLED
  }'
```

| | Lambda | ECS Fargate |
|---|---|---|
| Max runtime | 15 minutes | Unlimited |
| Unit of code | Function | Docker container |
| Scaling | Automatic, per-request | Task-based |
| Cold start | Yes | Slower start, but persistent |
| Best for | Event-driven automation, APIs | Microservices, batch jobs |

---

## Summary

- **Lambda** runs your code on demand — no servers to provision, patch, or scale.
- Initialise expensive objects (SDK clients, DB connections) outside `handler()` to avoid repeating the cost on every invocation.
- **API Gateway HTTP API** is the standard way to expose Lambda as an HTTP endpoint — it is 70% cheaper than REST API for most use cases.
- **EventBridge scheduled rules** are the serverless replacement for cron jobs.
- **Lambda Layers** share libraries across functions without duplicating them in every deployment package.
- Cold starts are real but manageable — use Provisioned Concurrency for latency-sensitive paths.
- **ECS Fargate** handles workloads that outgrow Lambda's 15-minute limit.

---

## Knowledge Check

1. What is the maximum execution timeout for a Lambda function?
2. You need to run a daily report job at 3 AM UTC. Which AWS service schedules the Lambda invocation?
3. Why should you initialise a DynamoDB client outside the `handler()` function?
4. What is the main difference between REST API and HTTP API in API Gateway, and when would you choose REST API?
5. Your Lambda needs the Pillow library (15 MB). Describe two ways to make it available at runtime.

---

## Hands-on Exercise

1. Create a Lambda function (Python 3.12) triggered by S3 uploads. Log the filename and file size to CloudWatch.
2. Create an EventBridge scheduled rule that invokes a Lambda function every 5 minutes. Verify it fires by checking CloudWatch Logs.
3. Build a 2-endpoint HTTP API with API Gateway and Lambda:
   - `GET /health` → returns `{"status": "ok"}`
   - `POST /echo` → returns the request body unchanged

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="10-security.md">← Previous: AWS Security Services</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="12-additional-services.md">Next: Additional AWS Services →</a>
</div>
