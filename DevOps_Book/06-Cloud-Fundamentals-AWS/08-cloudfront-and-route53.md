# Chapter 8 — CloudFront & Route53

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what a CDN is and why CloudFront reduces latency and origin load
- Create a CloudFront distribution backed by S3 or an ALB
- Secure an S3 origin using Origin Access Control (OAC)
- Manage cache behaviours and perform targeted cache invalidations
- Configure Route53 hosted zones and common record types
- Choose the right Route53 routing policy for your use case
- Issue and attach free SSL/TLS certificates with ACM
- Combine all three services into a production static-site + API architecture

---

## 8.1 What Is CloudFront?

CloudFront is AWS's Content Delivery Network (CDN). It operates from 400+ edge locations worldwide. When a user requests content:

1. The request hits the nearest edge location (often < 10 ms away)
2. If the content is cached there, it is returned immediately (cache **hit**)
3. If not, CloudFront fetches it from the **origin** (S3, ALB, EC2, API Gateway), caches it, and returns it (cache **miss**)

Benefits beyond caching:

- **DDoS protection** via AWS Shield Standard (always on, no extra cost)
- **WAF integration** — attach an AWS WAF web ACL to block bad traffic at the edge
- **HTTPS termination** at the edge — TLS handshake happens close to the user
- **Lambda@Edge / CloudFront Functions** — run code at the edge for A/B testing, auth, URL rewrites

---

## 8.2 CloudFront Distribution

```bash
# Create a distribution serving static content from S3
aws cloudfront create-distribution --distribution-config '{
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "my-s3-origin",
      "DomainName": "my-bucket.s3.us-east-1.amazonaws.com",
      "S3OriginConfig": {"OriginAccessIdentity": ""}
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "my-s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]},
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "PriceClass": "PriceClass_100",
  "Enabled": true,
  "Comment": "My static site CDN"
}'
```

Key fields:

| Field | Meaning |
|---|---|
| `ViewerProtocolPolicy: redirect-to-https` | HTTP requests are 301-redirected to HTTPS |
| `CachePolicyId` | Managed cache policy (the value above is the AWS-managed "CachingOptimized") |
| `Compress: true` | Gzip/Brotli compress eligible responses automatically |
| `PriceClass_100` | Only US/Europe/Asia edge locations (cheapest); use `PriceClass_All` for global |

---

## 8.3 Origin Access Control (OAC) — Securing S3

Without OAC, the S3 bucket must be public — any user who discovers the S3 URL can bypass CloudFront entirely (bypassing caching, WAF, and cost savings).

With OAC, the bucket is private. Only CloudFront can read from it via a signed request.

**Bucket policy to grant CloudFront access:**

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

The `Condition` ties the permission to your specific distribution — even other CloudFront distributions cannot access the bucket.

> OAC replaces the older OAI (Origin Access Identity) approach. Use OAC for all new distributions.

---

## 8.4 CloudFront Caching and Cache Invalidation

### Cache Behaviours

You can apply different caching rules per URL path pattern:

| Path Pattern | Cache TTL | Rationale |
|---|---|---|
| `/static/*` | 1 year (31536000 s) | Filenames are content-addressed (fingerprinted), safe to cache forever |
| `/api/*` | 0 (no cache) | Dynamic — must always go to origin |
| `/` (default) | 1 hour | Changes infrequently |

### Cache Invalidation

```bash
# Invalidate everything after a deployment
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"
```

Cost note: the first 1,000 invalidation paths per month are free; after that $0.005 per path.

**Better approach:** use versioned/fingerprinted filenames (`app.abc123.js`). The old file stays cached harmlessly; browsers fetch the new filename immediately via the updated HTML. Zero invalidations needed.

---

## 8.5 CloudFront with ALB (Dynamic Content)

CloudFront is not only for static files. Placing it in front of an ALB provides edge benefits for dynamic workloads too.

```
Browser → CloudFront → ALB → EC2 / ECS / Lambda
```

What you gain:

- **SSL termination at the edge**: TLS handshake completes within milliseconds for users globally
- **DDoS absorption**: volumetric attacks are absorbed at 400+ edges before reaching your ALB
- **Partial caching**: even caching GET responses for 30–60 seconds dramatically cuts origin requests for popular endpoints
- **Lambda@Edge**: inspect and modify requests/responses without round-tripping to the origin

---

## 8.6 What Is Route53?

Route53 is AWS's managed authoritative DNS service. When a browser resolves `api.myapp.com`, Route53 answers with the IP address (or alias target).

Key capabilities beyond basic DNS:

- **Health checks**: Route53 probes your endpoints and can automatically remove unhealthy ones from DNS responses
- **Traffic policies**: weighted, latency-based, geolocation, failover, and more
- **Domain registration**: buy and manage domains directly in AWS

---

## 8.7 Route53 Record Types

```bash
# A record: maps a hostname to an IPv4 address
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONEID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "1.2.3.4"}]
      }
    }]
  }'
```

**Alias records** — point a name at an AWS resource (ALB, CloudFront, S3 website):

```json
{
  "Name": "myapp.com",
  "Type": "A",
  "AliasTarget": {
    "HostedZoneId": "Z35SXDOTRQ7X7K",
    "DNSName": "my-alb-12345.us-east-1.elb.amazonaws.com",
    "EvaluateTargetHealth": true
  }
}
```

Why use Alias over CNAME?

- Alias works at the **zone apex** (`myapp.com`) — CNAMEs cannot
- Alias queries are **free** (CNAMEs count as paid queries)
- With `EvaluateTargetHealth: true`, Route53 inherits the ALB's health status

Common record types summary:

| Type | Maps | Example |
|---|---|---|
| A | Hostname → IPv4 | `api.myapp.com → 1.2.3.4` |
| AAAA | Hostname → IPv6 | `api.myapp.com → 2001:db8::1` |
| CNAME | Hostname → Hostname | `www → myapp.com` (not at apex) |
| Alias | Hostname → AWS resource | `myapp.com → alb.amazonaws.com` |
| MX | Mail exchange | Email routing |
| TXT | Arbitrary text | Domain ownership verification, SPF |

---

## 8.8 Route53 Routing Policies

| Policy | How It Works | Use Case |
|---|---|---|
| Simple | Single record, single value | Single server |
| Weighted | Split traffic by percentage weight | A/B testing, blue/green migrations |
| Latency | Route to the region with lowest latency | Multi-region applications |
| Failover | Primary → secondary if health check fails | Disaster recovery |
| Geolocation | Route by user's country or continent | Compliance, localisation |
| Geoproximity | Route by geographic distance, bias-adjustable | Fine-grained traffic shifting |
| Multi-value | Return up to 8 IPs, health-check filtered | Simple client-side load balancing |

```bash
# Weighted routing — canary release: 90% to v1, 10% to v2
# Record 1: Name=api.myapp.com, Weight=90, Value=v1-alb.amazonaws.com
# Record 2: Name=api.myapp.com, Weight=10, Value=v2-alb.amazonaws.com
# Gradually shift weight to 100/0 once v2 is confirmed stable
```

---

## 8.9 ACM — AWS Certificate Manager

ACM issues free, auto-renewing SSL/TLS certificates for use with AWS services.

```bash
# Request a wildcard certificate
aws acm request-certificate \
  --domain-name myapp.com \
  --subject-alternative-names "*.myapp.com" \
  --validation-method DNS \
  --region us-east-1   # CloudFront certificates MUST be in us-east-1
```

Validation workflow:

1. ACM gives you a CNAME record to add to your DNS
2. Add it to Route53 (or let ACM do it automatically if Route53 manages your domain)
3. ACM validates ownership and issues the certificate — usually within a few minutes
4. Certificate **auto-renews 60 days before expiry** — no manual action ever needed

> Important: certificates attached to **CloudFront** must be requested in `us-east-1` regardless of where your origin lives. Certificates for ALBs must be in the same region as the ALB.

---

## 8.10 Full Architecture: Static Site + API

```
                          ┌──────────────────┐
  myapp.com  ──Route53──▶ │   CloudFront     │──▶ S3 (static files: HTML/CSS/JS)
                          └──────────────────┘
                                   │ (OAC)

                          ┌──────────────────┐
  api.myapp.com ─Route53─▶│      ALB         │──▶ EC2 ASG (private subnets)
                          └──────────────────┘
```

Layer responsibilities:

| Layer | Component | Characteristic |
|---|---|---|
| DNS | Route53 | Alias to CloudFront / ALB, health checks |
| CDN | CloudFront | Edge caching, TLS, DDoS, WAF |
| Static storage | S3 | Infinitely scalable, private (OAC) |
| Load balancer | ALB | Path routing, health checks |
| Compute | EC2 ASG | Auto-scales, private subnets |
| TLS certificates | ACM | Free, auto-renewing |

Typical latencies:

- Static assets (cache hit): ~1–5 ms (served from edge)
- API calls: ~10–50 ms (regional, dependent on application)

---

## Summary

- CloudFront caches content at 400+ edges globally — reduces latency and origin load
- OAC locks your S3 bucket to CloudFront only — never make the bucket public
- Use versioned filenames instead of invalidations for near-zero CDN cost
- Route53 Alias records are the correct way to point an apex domain at an ALB or CloudFront
- Routing policies enable weighted canary releases, latency-based global routing, and automatic failover
- ACM certificates are free and auto-renewing — always use them instead of self-signed certs

---

## Knowledge Check

1. What is the difference between a CloudFront cache hit and a cache miss?
2. Why should you use OAC instead of making the S3 bucket public?
3. Why can't you use a CNAME record for the zone apex (`myapp.com`)?
4. You are running your app in `us-east-1` and `eu-west-1`. Which Route53 routing policy sends each user to the region with the best response time?
5. Your CloudFront distribution uses a custom domain. Where (which AWS region) must the ACM certificate be requested?

---

## Hands-on Exercise

1. Take the S3 static website you created in Chapter 5.
2. Create a CloudFront distribution in front of it with OAC — make the bucket private.
3. Set `ViewerProtocolPolicy` to `redirect-to-https`.
4. Request an ACM certificate in `us-east-1` for your domain and attach it to the distribution.
5. Create a Route53 Alias A record pointing your domain at the CloudFront distribution.
6. Upload a new `index.html` and perform a cache invalidation. Verify the new content appears.
7. Check the CloudFront access logs in S3 and identify your request.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-elb-and-autoscaling.md">← Previous: ELB & Auto Scaling</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-cloudwatch.md">Next: CloudWatch & Observability →</a>
</div>
