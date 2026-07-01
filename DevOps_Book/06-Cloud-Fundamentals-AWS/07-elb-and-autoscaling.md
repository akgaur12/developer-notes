# Chapter 7 — ELB & Auto Scaling

## Learning Objectives

By the end of this chapter you will be able to:

- Explain why load balancers are essential for production workloads
- Distinguish between ALB, NLB, and GWLB and choose the right one
- Create an Application Load Balancer with listeners and target groups
- Configure path-based and host-based routing rules
- Build an Auto Scaling Group backed by a launch template
- Implement target tracking, step scaling, and scheduled scaling policies
- Perform zero-downtime rolling deployments using instance refresh
- Cut compute costs with mixed On-Demand / Spot instance policies

---

## 7.1 Why Load Balancing?

A single server is a single point of failure: one crash and the service is gone, one traffic spike and it falls over.

A load balancer solves both problems:

- **Distributes traffic** across multiple instances so no single server is overloaded
- **Health checks** detect unhealthy instances and stop sending traffic to them automatically
- **Combined with Auto Scaling**: the fleet grows under load and shrinks when quiet — you pay only for what you need

```
Internet traffic
       │
  ┌────▼────┐
  │   ELB   │  ← health checks every 30 s
  └──┬──┬───┘
     │  │
 ┌───▼┐ ┌▼───┐
 │ EC2│ │ EC2│  ← registered targets
 └────┘ └────┘
```

---

## 7.2 ELB Types

| Type | OSI Layer | Protocols | Use Case |
|---|---|---|---|
| ALB (Application LB) | Layer 7 | HTTP / HTTPS / gRPC | Web apps, microservices, path & host routing |
| NLB (Network LB) | Layer 4 | TCP / UDP / TLS | Ultra-high performance, static IP, non-HTTP |
| GWLB (Gateway LB) | Layer 3 | IP | Inline network appliances (firewalls, IDS) |
| CLB (Classic LB) | L4 + L7 | HTTP / HTTPS / TCP | Legacy — do not use for new deployments |

**Rule of thumb:** default to ALB for web workloads; use NLB when you need sub-millisecond latency or a static Elastic IP address.

---

## 7.3 Application Load Balancer (ALB)

```bash
# Create ALB (internet-facing, across two public subnets)
aws elbv2 create-load-balancer \
  --name prod-alb \
  --type application \
  --subnets subnet-public-1a subnet-public-1b \
  --security-groups sg-alb

# Create target group (where the ALB forwards traffic)
aws elbv2 create-target-group \
  --name api-servers \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-12345678 \
  --target-type instance \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Create HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Register EC2 instances as targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-1234567890abcdef0 Id=i-0987654321fedcba0
```

Key concepts:

- **Listener**: watches for incoming connections on a port/protocol
- **Target group**: logical group of backends (instances, IPs, Lambda functions)
- **Health check**: periodic HTTP probe; only healthy targets receive traffic

---

## 7.4 ALB Path-Based and Host-Based Routing

ALB listener rules let you route different requests to different target groups without extra load balancers.

```bash
# Path-based routing: /api/* → api-target-group, all else → frontend-target-group
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --priority 10 \
  --conditions '[{"Field":"path-pattern","Values":["/api/*"]}]' \
  --actions '[{"Type":"forward","TargetGroupArn":"arn:...api-tg"}]'

# Host-based routing: api.myapp.com → api-tg, www.myapp.com → frontend-tg
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --priority 20 \
  --conditions '[{"Field":"host-header","Values":["api.myapp.com"]}]' \
  --actions '[{"Type":"forward","TargetGroupArn":"arn:...api-tg"}]'
```

Rules are evaluated in priority order (lowest number first). The default action on the listener acts as the catch-all.

---

## 7.5 ALB Access Logs and Monitoring

```bash
# Enable access logs — one compressed log file per minute, written to S3
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:... \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-logs \
    Key=access_logs.s3.prefix,Value=prod-alb
```

Key CloudWatch metrics to monitor:

| Metric | What it means |
|---|---|
| `RequestCount` | Total requests received |
| `TargetResponseTime` | Backend latency (watch p99) |
| `HTTPCode_Target_5XX_Count` | Application errors |
| `UnHealthyHostCount` | Targets failing health checks |
| `ActiveConnectionCount` | Current open connections |

---

## 7.6 Auto Scaling Groups (ASG)

An ASG automatically manages a fleet of EC2 instances. A **launch template** defines what each instance looks like; the ASG decides how many.

```bash
# Step 1: create a launch template
aws ec2 create-launch-template \
  --launch-template-name prod-app-lt \
  --launch-template-data '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.medium",
    "SecurityGroupIds": ["sg-app"],
    "IamInstanceProfile": {"Name": "ec2-app-role"},
    "UserData": "'$(base64 -w 0 user-data.sh)'"
  }'

# Step 2: create the ASG and attach it to the target group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name prod-app-asg \
  --launch-template LaunchTemplateName=prod-app-lt,Version='$Latest' \
  --min-size 2 \
  --desired-capacity 3 \
  --max-size 10 \
  --vpc-zone-identifier "subnet-private-1a,subnet-private-1b" \
  --target-group-arns arn:aws:elasticloadbalancing:... \
  --health-check-type ELB \
  --health-check-grace-period 300
```

- `min-size`: floor — never fewer instances than this
- `desired-capacity`: current target count
- `max-size`: ceiling — never more than this
- `health-check-grace-period`: seconds to wait before checking health on a new instance (allow boot time)

---

## 7.7 Scaling Policies

### Target Tracking (recommended default)

Keeps a metric at a target value. AWS creates and manages the CloudWatch alarms automatically.

```bash
# Keep average CPU utilisation at 50%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name prod-app-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

- `ScaleInCooldown 300`: wait 5 minutes between scale-in events (avoids thrashing)
- `ScaleOutCooldown 60`: scale out quickly when demand spikes

### Scheduled Scaling

```bash
# Scale to 10 instances at 8 am Mon–Fri (morning traffic surge)
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name prod-app-asg \
  --scheduled-action-name scale-up-morning \
  --recurrence "0 8 * * MON-FRI" \
  --desired-capacity 10

# Scale back to 3 at 8 pm Mon–Fri
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name prod-app-asg \
  --scheduled-action-name scale-down-evening \
  --recurrence "0 20 * * MON-FRI" \
  --desired-capacity 3
```

### Step Scaling

Define discrete steps: add 2 instances when CPU > 70 %, add 4 when CPU > 90 %, remove 1 when CPU < 30 %. More granular than target tracking; useful when the response needs to be non-linear.

---

## 7.8 Rolling Deployments with Instance Refresh

When you update a launch template (new AMI, new user-data), use instance refresh to replace instances gradually without downtime.

```bash
# Replace instances in rolling batches, keeping at least 80% healthy at all times
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name prod-app-asg \
  --preferences '{
    "MinHealthyPercentage": 80,
    "InstanceWarmup": 120
  }'
```

What happens:

1. AWS terminates ~20 % of instances at a time
2. New instances boot from the updated launch template version
3. Each batch must pass ELB health checks before the next batch is replaced
4. If something goes wrong: `cancel-instance-refresh`, then revert the launch template version

---

## 7.9 Cost Optimisation: Mixed Instance Policies

Combine a guaranteed On-Demand baseline with cheaper Spot instances for the rest.

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name cost-optimized-asg \
  --mixed-instances-policy '{
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "capacity-optimized"
    },
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {"LaunchTemplateName": "prod-app-lt"},
      "Overrides": [
        {"InstanceType": "t3.medium"},
        {"InstanceType": "t3a.medium"},
        {"InstanceType": "t2.medium"}
      ]
    }
  }'
```

What this gives you:

- First 2 instances: always On-Demand (SLA-safe baseline)
- Additional capacity: 80 % Spot, 20 % On-Demand
- `capacity-optimized`: AWS picks the Spot pool least likely to be interrupted
- Typical saving: ~60 % vs all On-Demand

---

## Summary

- ELB types: ALB (Layer 7, web), NLB (Layer 4, high throughput), GWLB (inline appliances)
- ALB rules enable path-based and host-based routing within a single load balancer
- Launch templates define instance configuration; ASGs manage fleet size
- Target tracking is the easiest scaling policy — set a target metric and AWS does the rest
- Instance refresh gives you rolling deployments without manual intervention
- Mixed instance policies dramatically reduce cost by blending Spot into your ASG

---

## Knowledge Check

1. What is the key difference between an ALB and an NLB?
2. You need `/api/*` to go to one set of servers and `/` to another on the same domain. Which ALB feature do you use?
3. An ASG has `min=2, desired=4, max=8`. CPU climbs to 90 %. What happens?
4. What does `health-check-grace-period` prevent?
5. You have a stateless application that can tolerate interruptions. How do you cut ASG cost by ~60 %?

---

## Hands-on Exercise

1. Create an ALB in your VPC across two public subnets.
2. Create a target group with an HTTP health check on `/health`.
3. Create a launch template using the Amazon Linux 2 AMI with a simple web server in user data.
4. Create an ASG with `min=2, desired=2, max=5` and attach it to the target group.
5. Add a CPU target tracking policy targeting 50 %.
6. Use `stress` or `ab` to generate load and watch instances scale out.
7. Terminate one instance manually and watch the ALB stop routing to it, then the ASG replace it.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-rds.md">← Previous: RDS & Database Services</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-cloudfront-and-route53.md">Next: CloudFront & Route53 →</a>
</div>
