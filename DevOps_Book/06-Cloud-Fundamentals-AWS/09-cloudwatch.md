# Chapter 9 — CloudWatch & Observability

## Learning Objectives

By the end of this chapter you will be able to:

- Describe the AWS observability stack and the role of each service
- Query CloudWatch metrics for EC2, ALB, and RDS resources
- Publish custom application metrics
- Create CloudWatch Alarms with SNS notifications
- Stream application logs to CloudWatch using the CloudWatch Agent
- Write CloudWatch Insights queries to analyse log data
- Enable CloudTrail for a complete API audit log
- Build a CloudWatch Dashboard for a production service
- Define a practical set of alarms for the most important resources

---

## 9.1 AWS Observability Stack

Observability answers three questions: what is my system doing (metrics), what did it say (logs), and how did a request travel through it (traces)?

| Service | Pillar | Purpose |
|---|---|---|
| CloudWatch Metrics | Metrics | Numeric time-series data — CPU, request rate, error count |
| CloudWatch Logs | Logs | Log storage, search, alerting, and analytics |
| CloudWatch Alarms | Alerting | Trigger an action when a metric crosses a threshold |
| CloudWatch Dashboards | Visibility | Shared visual dashboards for your team |
| CloudTrail | Audit | Every AWS API call — who did what, when, from where |
| AWS X-Ray | Traces | End-to-end request tracing across microservices |
| AWS Config | Compliance | Configuration history and compliance rules |

Start with metrics and alarms. Add logs. Add tracing when you have multiple services and need to track a single request across them.

---

## 9.2 CloudWatch Metrics

AWS services emit metrics automatically. EC2 free default metrics: CPU, network in/out, disk read/write operations.

```bash
# Retrieve hourly average CPU for an EC2 instance over the last 24 hours
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --period 3600 \
  --statistics Average
```

### Custom Metrics

Your application can emit its own metrics using the `put-metric-data` API or the CloudWatch Agent.

```bash
# Publish a custom metric from a script or application
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name RequestCount \
  --value 42 \
  --unit Count \
  --dimensions Environment=production,Service=api
```

Common custom metrics to emit:

- Queue depth / backlog size
- Active user sessions
- Business KPIs (orders per minute, payment failures)
- Memory utilisation (not available by default — requires CloudWatch Agent)

> **Note:** EC2 does not emit memory or disk space metrics by default. You must install and configure the CloudWatch Agent to get them.

---

## 9.3 CloudWatch Alarms

An alarm watches a single metric and transitions between three states: `OK`, `ALARM`, and `INSUFFICIENT_DATA`. When it enters `ALARM`, it can send an SNS notification, trigger an Auto Scaling action, or stop/reboot an EC2 instance.

```bash
# Alarm: CPU > 80% for 10 minutes (two 5-minute periods) → notify ops-team topic
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-i-1234567890 \
  --alarm-description "CPU above 80% for 10 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789:ops-team \
  --ok-actions arn:aws:sns:us-east-1:123456789:ops-team
```

- `period`: measurement window in seconds (300 = 5 minutes)
- `evaluation-periods`: how many consecutive periods must breach before alarming (reduces false positives)
- `ok-actions`: send a recovery notification when the alarm clears

```bash
# Billing alarm — must be created in us-east-1; requires billing alerts enabled
aws cloudwatch put-metric-alarm \
  --alarm-name billing-10-dollars \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum \
  --period 86400 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789:billing-alerts
```

Enable billing alerts first: AWS Console → Billing → Billing preferences → "Receive CloudWatch Billing Alerts".

---

## 9.4 CloudWatch Logs

CloudWatch Logs stores log data from EC2 instances, Lambda functions, ECS containers, and any custom source.

```bash
# Create a log group
aws logs create-log-group --log-group-name /myapp/api

# Set a 30-day retention policy (default is never expire — costs accumulate)
aws logs put-retention-policy \
  --log-group-name /myapp/api \
  --retention-in-days 30
```

### Streaming Logs from EC2 with the CloudWatch Agent

Install the agent on your instance, then provide a config file:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [{
          "file_path": "/var/log/app/app.log",
          "log_group_name": "/myapp/api",
          "log_stream_name": "{instance_id}",
          "timestamp_format": "%Y-%m-%d %H:%M:%S"
        }]
      }
    }
  }
}
```

Place this at `/etc/amazon/amazon-cloudwatch-agent/config.json`, then start the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/etc/amazon/amazon-cloudwatch-agent/config.json -s
```

### Querying Logs with CloudWatch Insights

```bash
# Start an Insights query — returns a queryId
aws logs start-query \
  --log-group-name /myapp/api \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message
    | filter @message like /ERROR/
    | sort @timestamp desc
    | limit 50'

# Retrieve results
aws logs get-query-results --query-id <queryId>
```

---

## 9.5 CloudWatch Insights Query Language

Insights uses a pipe-based query language similar to Unix. Each stage filters or transforms the results of the previous stage.

```
# Count errors grouped by error message (find the most frequent errors)
fields @timestamp, level, message
| filter level = "ERROR"
| stats count() by message
| sort count desc
| limit 20
```

```
# Latency percentiles — p95 and p99 in 5-minute buckets
fields @timestamp, duration_ms
| stats pct(duration_ms, 95) as p95, pct(duration_ms, 99) as p99 by bin(5m)
```

```
# Request rate over time (requests per minute)
filter @message like /GET \/api/
| stats count() as requests by bin(1m)
```

```
# Find all requests from a specific IP address
fields @timestamp, @message
| filter @message like /192.168.1.100/
| sort @timestamp desc
```

Key functions:

| Function | Description |
|---|---|
| `stats count()` | Row count |
| `stats avg(field)` | Average |
| `stats pct(field, 99)` | Percentile |
| `bin(5m)` | Round timestamp to 5-minute bucket |
| `filter field like /pattern/` | Regex match |
| `parse @message "... *" as field` | Extract field from unstructured text |

---

## 9.6 CloudTrail — API Audit Log

CloudTrail records every AWS API call made in your account: the API action, who called it, when, from which IP, and what the response was.

```bash
# Create a multi-region trail that archives to S3 with log file integrity validation
aws cloudtrail create-trail \
  --name prod-audit-trail \
  --s3-bucket-name my-audit-logs-bucket \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name prod-audit-trail
```

Without a trail, CloudTrail only keeps 90 days of event history in the console. With a trail, events are archived to S3 indefinitely (or per your S3 lifecycle policy).

### Example: Investigating an Incident

```bash
# Find out who deleted an S3 object in the last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteObject \
  --start-time 2024-01-14T00:00:00Z \
  --end-time 2024-01-15T00:00:00Z \
  --query 'Events[*].[EventTime,Username,EventName,Resources]'
```

Typical investigations:

- "Who changed this security group?" → filter `EventName=AuthorizeSecurityGroupIngress`
- "Why did my IAM role permissions change?" → filter `EventName=PutRolePolicy`
- "Which user launched these EC2 instances?" → filter `EventName=RunInstances`

> CloudTrail is the first place to look after a security incident.

---

## 9.7 SNS — Simple Notification Service

SNS is the glue between CloudWatch Alarms and the people or systems that need to act on them. Alarms publish to an SNS topic; subscribers (email, Lambda, SQS, PagerDuty webhook) receive the notification.

```bash
# Create a topic
aws sns create-topic --name ops-alerts

# Subscribe an email address
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:ops-alerts \
  --protocol email \
  --notification-endpoint ops@mycompany.com

# Subscribe a Lambda function (e.g., to forward alerts to Slack)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:ops-alerts \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456789:function:sns-to-slack

# Publish a message manually (useful for testing or deploy notifications)
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789:ops-alerts \
  --message "Deploy completed: myapp v1.2.3 to production" \
  --subject "Deploy Success"
```

Common patterns:

- **Email**: simple, no setup cost; delayed by a few seconds
- **SNS → Lambda → Slack**: real-time formatted messages in your team channel
- **SNS → SQS**: fan-out to multiple queues; durable (message survives Lambda/worker restart)
- **SNS → PagerDuty/OpsGenie**: on-call escalation with silencing and acknowledgement

---

## 9.8 CloudWatch Dashboard

Dashboards give your team a shared, real-time view of system health without navigating individual resource pages.

```bash
aws cloudwatch put-dashboard \
  --dashboard-name prod-overview \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "API Request Rate",
          "metrics": [
            ["AWS/ApplicationELB", "RequestCount",
             "LoadBalancer", "app/prod-alb/abc123",
             {"stat": "Sum", "period": 60}]
          ],
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "5xx Error Count",
          "metrics": [
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count",
             "LoadBalancer", "app/prod-alb/abc123",
             {"stat": "Sum", "period": 60, "color": "#d62728"}]
          ],
          "view": "timeSeries"
        }
      }
    ]
  }'
```

Recommended widgets for a production service overview:

1. Request rate (ALB `RequestCount`)
2. Error rate (ALB `HTTPCode_Target_5XX_Count`)
3. P99 latency (ALB `TargetResponseTime` p99)
4. Unhealthy host count (ALB `UnHealthyHostCount`)
5. ASG instance count (`GroupInServiceInstances`)
6. EC2 CPU heatmap
7. RDS connections and storage

---

## 9.9 Key Metrics to Monitor

Use this as a starting checklist when instrumenting a new service:

```
EC2 / Application Servers
  CPUUtilization            > 80%      → alarm (scale out or investigate)
  MemoryUtilization         > 85%      → alarm (custom metric, CloudWatch Agent required)
  StatusCheckFailed         >= 1       → alarm (instance hardware/OS problem)

ALB
  TargetResponseTime p99    > 1 s      → alarm (latency degradation)
  HTTPCode_Target_5XX_Count > 0        → alarm (application errors)
  UnHealthyHostCount        > 0        → alarm (instances failing health checks)
  RequestCount                         → track for capacity planning

RDS
  CPUUtilization            > 80%      → alarm
  FreeStorageSpace          < 10 GB    → alarm (disk fill can crash the database)
  DatabaseConnections       > 80% max  → alarm (connection exhaustion)
  ReadLatency / WriteLatency           → baseline and alert on deviation

Auto Scaling
  GroupInServiceInstances              → compare with GroupDesiredCapacity
  GroupDesiredCapacity                 → watch for unexpected scale-out events

Billing
  EstimatedCharges          > $X       → alarm (set per your expected budget)
```

Set `ScaleInCooldown` and `ScaleOutCooldown` on your ASG policies so that a brief spike does not trigger a scale event that causes more alarms. The goal is signal, not noise.

---

## Summary

- The AWS observability stack covers metrics (CloudWatch), logs (CloudWatch Logs), traces (X-Ray), and audit (CloudTrail)
- EC2 does not emit memory or disk metrics by default — install the CloudWatch Agent
- Alarms require `evaluation-periods > 1` to avoid false positives from transient spikes
- CloudWatch Insights provides a SQL-like language for querying logs at scale
- CloudTrail is your security and compliance record — always enable a multi-region trail
- SNS decouples alarm delivery from the notification channel (email, Slack, PagerDuty)
- A shared dashboard reduces mean time to detection (MTTD) because the team sees the same data

---

## Knowledge Check

1. Which AWS service would you use to find out who terminated an EC2 instance last night?
2. Why does CloudWatch not show memory utilisation for EC2 by default?
3. An alarm has `period=300, evaluation-periods=3`. How long must a metric stay above the threshold before the alarm fires?
4. What is the CloudWatch Insights function to calculate the 99th percentile of a field?
5. You want your team to receive Slack messages when an alarm fires. What AWS services do you chain together to achieve this?

---

## Hands-on Exercise

1. Launch an EC2 instance and install the CloudWatch Agent. Configure it to stream `/var/log/messages` to a log group `/lab/syslog` with 7-day retention.
2. Create an SNS topic `lab-alerts` and subscribe your email address to it.
3. Create a CloudWatch Alarm that fires when CPU > 70 % for two consecutive 5-minute periods. Send notifications to `lab-alerts`.
4. Enable CloudTrail with a trail writing to an S3 bucket. In the console, look up the `RunInstances` event for the instance you launched.
5. Write a CloudWatch Insights query that counts the number of log lines containing the word "error" per 5-minute window over the last hour.
6. Create a CloudWatch Dashboard with at least three widgets: CPU utilisation, the log metric filter count, and the alarm state widget.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-cloudfront-and-route53.md">← Previous: CloudFront & Route53</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-security.md">Next: AWS Security Services →</a>
</div>
