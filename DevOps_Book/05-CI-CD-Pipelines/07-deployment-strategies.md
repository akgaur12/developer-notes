# Chapter 7 — Deployment Strategies

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the trade-offs between rolling, blue/green, and canary deployments
- Choose the right deployment strategy for a given service and team
- Implement the expand/contract pattern for zero-downtime database migrations
- Write smoke tests that automatically trigger a rollback on failure

---

## 7.1 Why Deployment Strategy Matters

The goal of any deployment is to ship new code to users with **zero downtime** and the ability to **roll back** quickly if something goes wrong.

Different strategies make different trade-offs across three dimensions:

| Dimension | What it means |
|---|---|
| **Speed of rollout** | How fast does 100% of traffic see the new version? |
| **Blast radius** | If v2 is broken, how many users are affected before you notice? |
| **Infrastructure cost** | How much extra capacity does the deploy process require? |

The right choice depends on:

- **Criticality of the service** — a payment processor needs more caution than an internal dashboard
- **Team maturity** — canary deployments require robust monitoring to be effective
- **Infrastructure capability** — blue/green requires double the resources

---

## 7.2 Rolling Deployment

Deploy the new version gradually, replacing old instances one by one:

```
Before: [v1] [v1] [v1] [v1]
Step 1: [v2] [v1] [v1] [v1]
Step 2: [v2] [v2] [v1] [v1]
Step 3: [v2] [v2] [v2] [v1]
After:  [v2] [v2] [v2] [v2]
```

**Key characteristics:**

- At any point during rollout, some users get v1 and some get v2
- Requires backward-compatible API changes and DB migrations (both versions run simultaneously)
- No extra infrastructure needed — instances are replaced in-place

**Kubernetes** uses rolling updates by default:

```yaml
# deployment.yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # allow 1 extra pod during update
      maxUnavailable: 0  # never reduce below desired count
```

```bash
# Trigger a rolling update
kubectl set image deployment/myapp app=myapp:v2

# Watch progress
kubectl rollout status deployment/myapp

# Rollback
kubectl rollout undo deployment/myapp
```

**Docker Swarm:**

```bash
docker service update --image myapp:v2 myapp
docker service rollback myapp    # rollback if needed
```

---

## 7.3 Blue/Green Deployment

Run two identical environments and switch all traffic at once:

```
Blue (active, v1) ←── all traffic
Green (idle, v2)  ←── deploy here, test here
              ↓ (switch load balancer)
Blue (idle, v1)   ←── keep for instant rollback
Green (active, v2)←── all traffic
```

**Key characteristics:**

- **Zero downtime** — the traffic switch is instantaneous (DNS or load balancer rule)
- **Instant rollback** — flip the load balancer back to blue
- **Cost** — requires double the infrastructure while the deploy is in progress
- **Testing in production** — you can run smoke tests against the green environment before any user traffic reaches it

**Implementation options:**

- AWS ALB weighted target groups
- nginx `upstream` block swap
- Route 53 weighted routing records

```yaml
# GitHub Actions: blue/green with AWS ECS
- name: Deploy to green
  run: |
    aws ecs update-service \
      --cluster prod \
      --service myapp-green \
      --task-definition $NEW_TASK_DEF

- name: Run smoke tests on green
  run: ./smoke-test.sh https://green.internal.myapp.com

- name: Switch traffic to green
  run: |
    aws elbv2 modify-listener \
      --listener-arn $LISTENER_ARN \
      --default-actions \
        '[{"Type":"forward","TargetGroupArn":"$GREEN_TG_ARN"}]'

- name: Rollback on failure
  if: failure()
  run: |
    aws elbv2 modify-listener \
      --listener-arn $LISTENER_ARN \
      --default-actions \
        '[{"Type":"forward","TargetGroupArn":"$BLUE_TG_ARN"}]'
```

---

## 7.4 Canary Deployment

Send a small percentage of real traffic to the new version first, then gradually increase it:

```
v1: 95% traffic ──────────────────────────
v2:  5% traffic ─────
        ↓ (metrics look good after 10 min)
v1: 50% traffic ───────────────
v2: 50% traffic ───────────────
        ↓ (metrics still good)
v1:  0% traffic
v2: 100% traffic ──────────────────────────
```

**Key characteristics:**

- **Limits blast radius** — only a small fraction of users encounter a regression
- **Uses real traffic** — catches issues that synthetic tests miss
- **Requires monitoring** — you need error rate, latency, and business metrics to know when to advance or abort
- **Slower rollout** — intentionally takes time to reach 100%

**nginx weighted upstream canary:**

```nginx
upstream backend {
    server v1.internal weight=95;
    server v2.internal weight=5;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

**Kubernetes with Argo Rollouts:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
```

---

## 7.5 Feature Flags

Deploy code without enabling it; enable the feature separately from the deploy:

```javascript
// In application code
if (featureFlag.isEnabled('new-checkout', userId)) {
  return newCheckoutFlow(cart);
} else {
  return oldCheckoutFlow(cart);
}
```

**Why this matters:**

- **Decouple deploy from release** — code ships dark, feature turns on when the team is ready
- **Gradual rollout** — enable for 1% → 10% → 50% → 100% of users
- **Instant kill switch** — disable without deploying anything
- **A/B testing** — show different variants to different user segments

**Tooling options:**

| Tool | Type | Notes |
|---|---|---|
| LaunchDarkly | SaaS | Full-featured, team-oriented |
| GrowthBook | Open source / SaaS | Good for A/B testing |
| Unleash | Open source | Self-hosted, enterprise-ready |
| Environment variables | DIY | Simplest; requires redeploy to change |

---

## 7.6 Database Migrations in CD

Database migrations are the hardest part of zero-downtime deployments. The fundamental problem:

```
Problem: app v2 expects a new DB column
         app v1 doesn't know about that column
         during a rolling deploy, both v1 and v2 run simultaneously
         → v2 breaks if column missing; v1 breaks if column is NOT NULL
```

**Solution: Expand / Contract (3-phase migration)**

```
Phase 1 — Expand (deploy this first, before v2):
  ALTER TABLE orders ADD COLUMN new_status VARCHAR(50) DEFAULT 'pending';
  (nullable with default → v1 ignores it, v2 can write to it)

Phase 2 — Backfill (run as a job, both versions work fine):
  UPDATE orders SET new_status = status WHERE new_status IS NULL;

Phase 3 — Contract (deploy after all v1 instances are retired):
  ALTER TABLE orders DROP COLUMN status;
  ALTER TABLE orders ALTER COLUMN new_status SET NOT NULL;
```

```yaml
# Run migration BEFORE updating the app image
- name: Run DB migration
  run: |
    docker run --rm \
      -e DATABASE_URL=${{ secrets.DATABASE_URL }} \
      myapp:${{ github.sha }} \
      python manage.py migrate

- name: Deploy new version
  run: kubectl set image deployment/myapp app=myapp:${{ github.sha }}

- name: Wait for rollout
  run: kubectl rollout status deployment/myapp --timeout=5m
```

**Golden rules for DB migrations in CD:**

1. Always run migrations before deploying the new app version
2. Every migration must be backward-compatible with the current production app version
3. Never rename or drop a column in the same deploy that removes code using it
4. Test the migration on a production-size dataset before shipping

---

## 7.7 Rollback Strategies

Always be able to roll back within 5 minutes. Here is how to do it on common platforms:

```bash
# Kubernetes — undo last rollout
kubectl rollout undo deployment/myapp

# Kubernetes — roll back to a specific revision
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3

# Docker Swarm
docker service rollback myapp

# AWS ECS — redeploy a previous task definition
aws ecs update-service \
  --cluster prod \
  --service myapp \
  --task-definition myapp:42    # last known-good version number

# Helm
helm history myapp
helm rollback myapp 2
```

**What makes rollback reliable:**

- Immutable image tags — never overwrite `:latest` in production; always use commit SHA or version tags
- Keep the previous deployment artifact available (ECS task definition, Helm release, etc.)
- Smoke tests that fail fast — a 5-minute smoke test window beats a 30-minute one
- Database rollback plan — migrations run forward only; design them so the old app still works if you have to rollback the app but not the schema

---

## 7.8 Smoke Tests and Health Checks Post-Deploy

A smoke test runs immediately after deployment and triggers an automatic rollback if it fails:

```bash
#!/bin/bash
# smoke-test.sh
BASE_URL="${1:-https://api.myapp.com}"

echo "Testing $BASE_URL..."

check() {
  local url="$1"
  local expected="$2"
  local code
  code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
  if [ "$code" != "$expected" ]; then
    echo "FAIL: $url returned $code (expected $expected)"
    exit 1
  fi
  echo "OK: $url → $code"
}

check "$BASE_URL/health"          "200"
check "$BASE_URL/api/v1/status"   "200"
check "$BASE_URL/api/v1/version"  "200"

echo "All smoke tests passed."
```

Wire it into the pipeline with automatic rollback on failure:

```yaml
- name: Deploy
  run: kubectl set image deployment/myapp app=myapp:${{ github.sha }}

- name: Wait for rollout
  run: kubectl rollout status deployment/myapp --timeout=5m

- name: Smoke test
  run: ./smoke-test.sh https://api.myapp.com

- name: Rollback on failure
  if: failure()
  run: kubectl rollout undo deployment/myapp
```

**What to check in a smoke test:**

- `/health` — app is alive and dependencies (DB, cache) are reachable
- One or two key user-facing endpoints — the homepage, the login endpoint, the main API
- A known response value — assert on content, not just HTTP status code
- Response time — a 200 that takes 30 seconds is also a failure

---

## Summary

| Strategy | Downtime | Rollback speed | Infrastructure cost | When to use |
|---|---|---|---|---|
| Rolling | None | ~5 min | None extra | Most services; requires backward-compat |
| Blue/Green | None | Instant | 2x during deploy | Critical services; when you can afford cost |
| Canary | None | Instant abort | Small overhead | High-traffic services with good monitoring |
| Feature flag | None | Instant toggle | None | Feature releases decoupled from deployments |

Key principles to remember:

- Migrations must run **before** new app code; they must be **backward-compatible** with old app code
- Smoke tests must be wired to **automatic rollback** — a manual process will be skipped under pressure
- Use **immutable image tags** in production; never deploy `:latest`
- The expand/contract pattern solves schema changes in zero-downtime deployments

---

## Knowledge Check

1. During a rolling deployment, what is the primary constraint on how you write database migrations?
2. What is the main advantage of blue/green over rolling deployment? What is the main cost?
3. Describe a scenario where a canary deployment catches a bug that a blue/green deployment would not catch until it was too late.
4. What does the "expand" phase of the expand/contract migration pattern do, and why must it be deployed before the new application version?
5. You deploy v2 and your smoke test fails. What should happen next, automatically?

---

## Hands-on Exercise

**Goal:** Implement a blue/green deployment locally with Docker and nginx.

**Steps:**

1. Create two simple web services (`v1` and `v2`) using Docker. `v1` returns `{"version": "1"}`, `v2` returns `{"version": "2"}`.
2. Write an nginx config with two upstream blocks (`blue` and `green`) and a single `server` block pointing at one of them.
3. Write a shell script `deploy-green.sh` that:
   - Starts the v2 container as `green`
   - Runs the smoke test against `green` directly
   - If tests pass: rewrites the nginx config to point at `green` and reloads nginx (`nginx -s reload`)
   - If tests fail: stops the green container and exits with an error
4. Write a `rollback.sh` that switches nginx back to `blue` within 30 seconds.

**Success criteria:** Traffic switches atomically (no requests drop) and rollback completes in under 30 seconds.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="06-testing-strategies.md">← Previous: Testing Strategies</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="08-gitlab-ci.md">Next: GitLab CI →</a>
</div>
