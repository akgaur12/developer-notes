# Chapter 11 — Building AWS Compute with Terraform

## Learning Objectives

By the end of this chapter you will be able to:

- Build EC2 Launch Templates with IMDSv2, gp3 volumes, and user data via `templatefile()`
- Configure Auto Scaling Groups with rolling instance refresh and target-tracking scaling policies
- Provision an Application Load Balancer with HTTPS listener, HTTP-to-HTTPS redirect, and health checks
- Define IAM roles and instance profiles for EC2 using `aws_iam_policy_document` data sources
- Package and deploy Lambda functions with EventBridge triggers
- Create CloudWatch alarms and SNS topics for operational alerting

---

## 11.1 EC2 Launch Templates

A Launch Template defines everything about an EC2 instance: AMI, instance type, security groups, IAM profile, storage, and user data. Auto Scaling Groups reference the template and spin up identical instances from it.

```hcl
# Always use a data source to find the latest AMI — never hard-code AMI IDs
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_launch_template" "app" {
  name_prefix   = "${local.name_prefix}-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  vpc_security_group_ids = [var.app_security_group_id]

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 20
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/user_data.tpl", {
    environment = var.environment
    app_version = var.app_version
    db_secret   = var.db_secret_arn
  }))

  metadata_options {
    http_tokens   = "required"   # IMDSv2 — prevents SSRF metadata attacks
    http_endpoint = "enabled"
  }

  tag_specifications {
    resource_type = "instance"
    tags          = merge(local.common_tags, { Name = "${local.name_prefix}-app" })
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**Key points:**

- `name_prefix` (not `name`) lets Terraform create a new template before destroying the old one, which ASGs require for zero-downtime updates
- `http_tokens = "required"` enforces IMDSv2 — this closes the SSRF-to-metadata-service attack vector that compromised the Capital One breach
- `templatefile()` renders a `.tpl` file with variables substituted in — cleaner than embedding a giant bash script as a heredoc
- `gp3` is newer than `gp2` and provides higher baseline throughput at the same cost

**Example user_data.tpl:**

```bash
#!/bin/bash
yum update -y
aws secretsmanager get-secret-value \
  --secret-id "${db_secret}" \
  --region $(curl -s http://169.254.169.254/latest/meta-data/placement/region) \
  --query SecretString --output text > /etc/app/db_credentials.json
systemctl start app-${app_version}
```

---

## 11.2 Auto Scaling Group

The ASG uses the launch template to maintain a fleet of EC2 instances and integrates with the ALB target group for health checks.

```hcl
resource "aws_autoscaling_group" "app" {
  name                = "${local.name_prefix}-asg"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  health_check_grace_period = 300

  min_size         = var.min_capacity
  max_size         = var.max_capacity
  desired_capacity = var.min_capacity

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  # Replace instances rolling (not all at once)
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 80
      instance_warmup        = 120
    }
  }

  dynamic "tag" {
    for_each = merge(local.common_tags, { Name = "${local.name_prefix}-instance" })
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }

  lifecycle {
    ignore_changes = [desired_capacity]   # let ASG manage its own count
  }
}

# Target tracking scaling policy — scale out when CPU exceeds 60%
resource "aws_autoscaling_policy" "cpu_target" {
  name                   = "${local.name_prefix}-cpu-target"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value       = 60.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

**Key points:**

- `health_check_type = "ELB"` means the ASG trusts the ALB's health check — if the ALB marks an instance unhealthy (failed `/health` endpoint), the ASG replaces it
- `ignore_changes = [desired_capacity]` prevents Terraform from resetting the ASG's count to `min_capacity` every apply after the scaling policy has scaled it up
- `instance_refresh` with `min_healthy_percentage = 80` means: when the launch template changes, replace instances 20% at a time — avoids a full fleet replacement
- `dynamic "tag"` block iterates over the tags map and sets `propagate_at_launch = true` so instances inherit tags

---

## 11.3 Application Load Balancer

```hcl
resource "aws_lb" "main" {
  name               = "${local.name_prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [var.alb_security_group_id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = var.environment == "prod"

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    prefix  = "alb"
    enabled = true
  }

  tags = local.common_tags
}

resource "aws_lb_target_group" "app" {
  name        = "${local.name_prefix}-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "instance"

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200"
  }

  deregistration_delay = 30

  tags = local.common_tags
}

# HTTPS listener — terminate TLS at the ALB
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# HTTP listener — redirect to HTTPS
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

**Key points:**

- `enable_deletion_protection = var.environment == "prod"` — Terraform conditional expression: prevents accidental `terraform destroy` in production
- `ssl_policy = "ELBSecurityPolicy-TLS13-1-2-2021-06"` — supports TLS 1.3 and removes weak cipher suites
- `deregistration_delay = 30` — ALB waits only 30 seconds (not the default 300) before sending no more traffic to a deregistering instance, speeding up deployments
- Access logs to S3 are essential for debugging and security incident response

---

## 11.4 IAM Role for EC2

EC2 instances need an IAM role to call AWS APIs (Secrets Manager, S3, SSM) without embedding credentials. The pattern: role → policy attachment → instance profile.

```hcl
# Trust policy — allows EC2 service to assume this role
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "app" {
  name               = "${local.name_prefix}-app-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
  tags               = local.common_tags
}

# Attach AWS managed policy for SSM Session Manager (no SSH required)
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Custom policy for application-specific permissions
data "aws_iam_policy_document" "app_permissions" {
  statement {
    effect    = "Allow"
    actions   = ["secretsmanager:GetSecretValue"]
    resources = [var.db_secret_arn]
  }
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["${aws_s3_bucket.uploads.arn}/*"]
  }
}

resource "aws_iam_policy" "app" {
  name   = "${local.name_prefix}-app-policy"
  policy = data.aws_iam_policy_document.app_permissions.json
}

resource "aws_iam_role_policy_attachment" "app" {
  role       = aws_iam_role.app.name
  policy_arn = aws_iam_policy.app.arn
}

# Instance profile — wraps the role so EC2 can use it
resource "aws_iam_instance_profile" "app" {
  name = "${local.name_prefix}-app-profile"
  role = aws_iam_role.app.name
}
```

**Why `aws_iam_policy_document` over inline JSON strings?**

- Terraform validates the structure at plan time
- Supports variable interpolation cleanly
- Version-controlled, diff-friendly
- Can be combined with `data.aws_iam_policy_document` `source_policy_documents` to merge multiple documents

---

## 11.5 Lambda with Terraform

```hcl
# Package the Lambda source code into a zip on every apply
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/lambda_src"
  output_path = "${path.module}/lambda_function.zip"
}

resource "aws_lambda_function" "processor" {
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  function_name    = "${local.name_prefix}-processor"
  role             = aws_iam_role.lambda.arn
  handler          = "handler.lambda_handler"
  runtime          = "python3.12"
  timeout          = 30
  memory_size      = 256

  environment {
    variables = {
      ENVIRONMENT = var.environment
      DB_SECRET   = var.db_secret_arn
    }
  }

  tracing_config {
    mode = "Active"   # X-Ray tracing
  }

  tags = local.common_tags
}

# Always create an explicit log group to control retention
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${aws_lambda_function.processor.function_name}"
  retention_in_days = var.log_retention_days
  tags              = local.common_tags
}

# EventBridge rule — schedule the Lambda daily at 02:00 UTC
resource "aws_cloudwatch_event_rule" "daily" {
  name                = "${local.name_prefix}-daily-job"
  schedule_expression = "cron(0 2 * * ? *)"
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.daily.name
  target_id = "lambda"
  arn       = aws_lambda_function.processor.arn
}

# Permission — allow EventBridge to invoke the Lambda
resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.processor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily.arn
}
```

**Key points:**

- `source_code_hash` is the critical field — it tells Terraform to redeploy the Lambda when the zip contents change. Without it, Terraform would never update the function code after the first apply
- Always create the CloudWatch log group explicitly. If you let Lambda auto-create it, you cannot control the retention period and logs will be retained indefinitely
- `aws_lambda_permission` is easy to forget — without it, EventBridge gets an access denied when invoking the Lambda

---

## 11.6 CloudWatch Alarms

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${local.name_prefix}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU above 80% for 10 minutes"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}

resource "aws_cloudwatch_metric_alarm" "unhealthy_hosts" {
  alarm_name          = "${local.name_prefix}-unhealthy-hosts"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnHealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Average"
  threshold           = 0
  alarm_description   = "One or more ALB targets are unhealthy"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
    TargetGroup  = aws_lb_target_group.app.arn_suffix
  }
}

resource "aws_sns_topic" "alerts" {
  name = "${local.name_prefix}-alerts"
  tags = local.common_tags
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.ops_email
}
```

**Note on `ok_actions`:** Setting `ok_actions` to the same SNS topic means you get a notification when the alarm recovers too, not just when it fires. This is important for on-call workflows.

---

## Summary

- Launch Templates define what instances look like; ASGs define how many to run and where
- `instance_refresh` enables rolling deployments when the launch template changes — no downtime
- `ignore_changes = [desired_capacity]` prevents Terraform from fighting with the ASG scaling policies
- ALB handles TLS termination; always redirect HTTP → HTTPS with a 301
- IAM roles for EC2 use instance profiles — never embed credentials in user data
- Lambda deployments require `source_code_hash` to detect code changes
- CloudWatch alarms should fire on both alarm and OK state for clean on-call notifications

---

## Knowledge Check

1. Why is `http_tokens = "required"` in the Launch Template's `metadata_options` a security best practice?
2. What does `ignore_changes = [desired_capacity]` do in the ASG resource, and why is it necessary?
3. What is the difference between `name` and `name_prefix` in `aws_launch_template`, and which should you use?
4. Why must `aws_lambda_permission` be created when using EventBridge to trigger a Lambda?
5. What happens if you do not create `aws_cloudwatch_log_group` for a Lambda — will it fail?

---

## Hands-on Exercise

Write a complete Terraform configuration for the following compute stack:

- A Launch Template using the latest Amazon Linux 2 AMI, `t3.small` instance type, gp3 20GB encrypted root volume, and IMDSv2 enforced
- A user data template file that installs the AWS CLI and fetches a secret from Secrets Manager on startup
- An Auto Scaling Group with min=1, max=4, health check type ELB, rolling instance refresh at 80% minimum healthy
- A target-tracking scaling policy targeting 60% average CPU
- An Application Load Balancer in public subnets with:
  - HTTPS listener on port 443 with TLS 1.3 policy forwarding to a target group
  - HTTP listener on port 80 redirecting to HTTPS with HTTP 301
  - Target group health check on `/health` returning `200`
- An IAM role for the EC2 instances with permission to call `secretsmanager:GetSecretValue` on one specific secret ARN, plus the `AmazonSSMManagedInstanceCore` managed policy
- A CloudWatch CPU alarm triggering at 80% for 2 consecutive 5-minute periods, publishing to an SNS topic with an email subscription

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="10-aws-networking.md">← Previous: Building AWS Networking with Terraform</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="12-aws-databases.md">Next: Building AWS Databases with Terraform →</a>
</div>
