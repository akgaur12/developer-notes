# Chapter 15 — Common Mistakes and Pitfalls

## Learning Objectives

By the end of this chapter, you will be able to:

- Identify the most common observability mistakes — metrics, logging, tracing, alerting, and SLOs — by recognizing their symptoms
- Understand the misunderstanding or organizational pressure that leads to each mistake
- Apply the correct pattern from earlier chapters of this course immediately, without having to look it up
- Recognize these mistakes in a dashboard, alerting rule, or logging statement before they cause an incident or a silent monitoring blind spot

---

## How to Read This Chapter

Each mistake is presented with four parts:

1. **The wrong pattern** — a config, query, or code snippet you will encounter in the wild
2. **Why it happens** — the misunderstanding or pressure that leads to it
3. **The correct fix** — a drop-in replacement, pointing back to the chapter that covers it in depth
4. **Impact / Prevention** — what breaks in production, and how to stop it happening again

This chapter is strictly about the **observability domain** — metrics, logs, traces, alerting, and SLOs. If you're looking for application-deployment mistakes (missing probes, `:latest` tags, no resource limits), those are covered in Kubernetes Basics, Chapter 16. If you're looking for platform/cluster-admin mistakes (broad RBAC grants, unverified NetworkPolicies, premature multi-cluster adoption), those are covered in Advanced Kubernetes, Chapter 15. This chapter assumes you already know and avoid both of those lists, and covers what goes wrong specifically in how a team observes what it's running.

---

## Mistake 1: Graphing a Raw Counter Instead of `rate()`

```promql
# WRONG — a raw counter, graphed directly on a dashboard panel
http_requests_total{job="checkout-api"}
```

**Why it happens:** The metric name itself (`http_requests_total`) sounds like it should just show "how many requests happened," so a new dashboard author drops it straight into a panel without applying any function. Grafana renders it happily, with no error, no warning — it just draws a line.

**The correct fix:** Wrap the counter in `rate()` over a sensible window (Chapter 4):

```promql
rate(http_requests_total{job="checkout-api"}[5m])
```

**Impact:** A raw counter graph only ever goes up, for the entire lifetime of the process, then drops instantly to zero and starts climbing again on every restart. This produces a permanently ascending line punctuated by sawtooth drops that carries zero usable information about the current traffic rate — you cannot tell from it whether traffic just spiked, just recovered, or has been flat for a week. Anyone glancing at this panel during an incident wastes time trying to interpret a graph shape that fundamentally cannot answer "is traffic elevated right now."

**Prevention:** Treat any dashboard panel built directly from a metric ending in `_total` or `_count`, with no `rate()`/`increase()`/`irate()` applied, as an automatic review rejection. Some teams enforce this with a linter over exported dashboard JSON that flags raw counter queries.

---

## Mistake 2: Unbounded Label Cardinality on Custom Metrics

```python
# WRONG — a raw Kubernetes Pod name (or a user ID) as a metric label
orders_total.labels(pod=os.environ["HOSTNAME"], user_id=user.id, status=status).inc()
```

**Why it happens:** Adding "just one more label" to an existing metric feels free — it's a one-line code change, and it makes a specific debugging question easier to answer *in that moment*. Nobody instruments a metric planning for it to cause an outage; the cardinality cost is invisible at write time and only shows up, gradually, in Prometheus's memory usage weeks later.

**The correct fix:** Never put an unbounded or ephemeral value (a Kubernetes Pod name, a Pod IP, a user ID, a request ID, a raw session token) on a metric label. Use stable, bounded labels instead, and push per-request or per-user identifiers into logs and trace spans, which are built for high-cardinality data (Chapters 2 and 13):

```python
# RIGHT — stable, bounded labels only
orders_total.labels(status=status).inc()
# the specific user_id, if needed for investigation, belongs in a
# structured log line or a trace span attribute, not a metric label
```

**Impact:** This is the single most damaging and common real-world Prometheus mistake. Kubernetes Pods are recreated constantly — every rolling deploy, every autoscaling event, every node failure creates new Pod identities — so a `pod` or `user_id` label creates a new, never-reused time series on essentially every restart or every distinct user. Over weeks of normal operation, this silently and continuously grows the number of active series Prometheus must hold in memory, with no single dramatic event to point to — until Prometheus eventually OOMs, or a managed observability vendor's cost-per-series bill spikes without warning. Chapter 13 walks through a full six-week version of exactly this incident.

**Prevention:** Review every new metric label at instrumentation time against the question "how many distinct values can this take over the metric's lifetime?" — if the answer is unbounded, it doesn't belong on a label. Back this up with a platform-level `metricRelabelings` drop rule (Chapter 13) as a safety net, and a standing alert on `prometheus_tsdb_head_series` so a regression is caught in hours, not weeks.

---

## Mistake 3: Alerting on a Raw Instantaneous Threshold With No `for:` Duration

```yaml
# WRONG — fires the instant a single scrape crosses the threshold
- alert: HighCPUUsage
  expr: instance_cpu_usage_percent > 90
  labels:
    severity: page
  # no `for:` field at all
```

**Why it happens:** The alerting rule looks complete — a metric, a threshold, a severity label — and Prometheus accepts it without complaint. It's easy to forget that a single noisy scrape (a brief garbage-collection pause, a momentary scheduling hiccup) can cross almost any threshold for exactly one scrape interval without anything actually being wrong.

**The correct fix:** Require the condition to hold continuously for a meaningful duration before firing, using `for:` (Chapter 7):

```yaml
- alert: HighCPUUsage
  expr: instance_cpu_usage_percent > 90
  for: 10m
  labels:
    severity: page
```

**Impact:** Without `for:`, the alert pages on a single-scrape blip that resolves itself before anyone can even look at it — the page arrives, the on-call engineer checks the dashboard, and the metric is already back to normal, with no actionable information left to investigate. Repeated often enough, this is one of the fastest ways to train a team to distrust and start ignoring pages altogether, which is precisely the alert-fatigue failure mode Chapter 7 warns about.

**Prevention:** Treat a missing `for:` field on any threshold-based alert as a review-blocking issue. Size the duration to the noise characteristics of the specific metric — a few minutes for something naturally noisy, shorter for something that should never fluctuate briefly at all.

---

## Mistake 4: No Inhibition or Grouping in Alertmanager

```yaml
# WRONG — every alert routes independently, with no grouping or
# inhibition rules configured at all
route:
  receiver: 'oncall-pager'
  # no group_by, no inhibit_rules anywhere in the config
```

**Why it happens:** Alerting rules (Prometheus) and alert routing (Alertmanager) are configured separately, and it's easy to get every individual alerting rule right while never coming back to configure how Alertmanager should group or suppress related alerts that fire together. Grouping and inhibition aren't needed to make a single alert work correctly in isolation — the gap only becomes visible when many alerts fire simultaneously from one root cause.

**The correct fix:** Configure `group_by` so related alerts are batched into one notification, and `inhibit_rules` so a higher-severity root-cause alert suppresses the lower-severity symptom alerts it would otherwise trigger (Chapter 7):

```yaml
route:
  receiver: 'oncall-pager'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m

inhibit_rules:
  - source_matchers: ['alertname="NodeDown"']
    target_matchers: ['alertname=~"PodNotReady|HighLatency"']
    equal: ['cluster', 'node']
```

**Impact:** A single node going down can simultaneously trigger dozens of individual Pod-not-ready alerts, several service-level latency alerts, and a node-health alert — all genuinely true, all caused by the exact same root event. Without grouping and inhibition, the on-call engineer receives fifty near-duplicate pages in the space of a minute for one actual problem, making it harder (not easier) to see the actual root cause underneath the noise, and training the team to associate "page storm" with "probably just one thing" — which is exactly the habit that causes a genuinely severe, unrelated multi-alert incident to be dismissed too quickly.

**Prevention:** Configure `group_by` and at least the most obvious `inhibit_rules` (a node-down alert inhibiting that node's dependent Pod/service alerts) as a mandatory part of any new Alertmanager route, not an optional refinement added later. Test it by deliberately triggering a root-cause scenario in staging and confirming one notification arrives, not fifty.

---

## Mistake 5: Setting an SLO at 100%

```yaml
# WRONG — an SLO with no error budget at all
slo:
  name: checkout-api-availability
  target: 100%
  window: 30d
```

**Why it happens:** 100% sounds like the obviously correct target — who would deliberately aim for less than perfect? It's an intuitive but fundamentally mistaken instinct, because it misunderstands what an SLO is *for* (Chapter 12): an error budget is meant to be a deliberately-sized allowance for imperfection, used to make informed decisions about risk, not a scorecard to maximize.

**The correct fix:** Set an SLO based on what users actually need and what's realistically achievable given your architecture's genuine failure modes (network blips, dependency hiccups, deploys) — commonly 99.9% or 99.95% for most services, reserving 99.99%+ for services with a specific, justified business need for it:

```yaml
slo:
  name: checkout-api-availability
  target: 99.9%
  window: 30d
  # 99.9% over 30 days = ~43 minutes of allowed downtime —
  # a real, usable error budget for weighing risk decisions against
```

**Impact:** A 100% target has a zero-size error budget by definition, which means literally any error, anywhere, is a budget breach — a state that is either quietly ignored (because treating every single error as a crisis is unsustainable, so the team stops reacting to the "breach" at all) or actively enforced (in which case no deploy, no risky change, no experiment is ever approved, because anything with any chance of causing an error is prohibited). Either outcome defeats the entire purpose of having an error budget in the first place: making an explicit, informed tradeoff between reliability and feature velocity.

**Prevention:** Set SLO targets from actual user expectations and business impact, not from an instinct toward perfection. Revisit the target periodically using real historical performance and real user feedback, and treat "our SLO is unrealistic and getting ignored" as a signal to adjust the target, not a signal to try harder to hit an impossible one.

---

## Mistake 6: Logging Unstructured Free-Text Instead of Structured JSON

```text
# WRONG — a human-readable, unstructured log line
2026-07-01 09:14:02 ERROR Payment failed for order 9f2c1a with provider stripe, timeout after 30s
```

**Why it happens:** Free-text log lines are what `print()`/`console.log()` produce by default, and they're perfectly readable to a human `tail`-ing a single file on a single machine — which is exactly the environment most engineers first learn logging in. The gap only shows up once logs are centralized (Chapter 8) across many services and many replicas, at a volume no human reads line by line.

**The correct fix:** Emit structured JSON with consistent field names across services, so a log aggregation backend can filter, aggregate, and alert on log content rather than requiring a human to read prose (Chapter 8):

```json
{
  "timestamp": "2026-07-01T09:14:02.881Z",
  "level": "error",
  "service": "checkout-api",
  "message": "payment provider timeout",
  "order_id": "9f2c1a",
  "provider": "stripe",
  "timeout_seconds": 30
}
```

**Impact:** Free-text logs are effectively unqueryable at scale beyond a simple substring `grep`. Answering "how many payment timeouts happened against Stripe specifically, in the last hour, across all 40 replicas" requires either a fragile regular expression that breaks the next time someone slightly rewords the log message, or manually reading through raw log output — neither of which scales past a handful of services. Structured fields turn the same question into a direct, reliable query (`provider="stripe" AND message="payment provider timeout"`) that keeps working even as unrelated parts of the message wording change.

**Prevention:** Standardize on a structured logging library across all services from day one, with a shared, documented set of common field names (`service`, `level`, `trace_id`, and so on) so every team's logs are queryable the same way, rather than each service inventing its own ad hoc format.

---

## Mistake 7: Logging Sensitive Data Into a Centralized, Widely-Queryable Log Store

```python
# WRONG — logs the full request payload, including sensitive fields, verbatim
logger.info(f"Processing payment request: {request.json}")
# request.json includes: {"card_number": "4111111111111111",
#                          "cvv": "123", "password": "hunter2"}
```

**Why it happens:** Logging the full request object is a fast, convenient way to get maximum debugging context with minimal code, and it's easy to forget that "the full request" includes fields that should never leave the service boundary in plaintext. Centralized logging (Chapter 8) makes this worse than a local log file would have: once ingested, that sensitive data is now searchable by every engineer with query access to the logging backend, indefinitely, for as long as retention keeps it.

**The correct fix:** Redact or omit sensitive fields before logging, as a matter of course, not as an afterthought — the same "never hardcode secrets" discipline from earlier courses in this series, now applied to log *content* rather than source code or config:

```python
# RIGHT — explicitly redact sensitive fields before logging anything
safe_payload = redact_fields(request.json, fields=["card_number", "cvv", "password"])
logger.info("Processing payment request", extra={"payload": safe_payload})
# logs: {"payload": {"card_number": "[REDACTED]", "cvv": "[REDACTED]",
#                     "password": "[REDACTED]", "order_id": "9f2c1a"}}
```

**Impact:** This is a real security and compliance exposure, not a style nitpick. A centralized log store is, by design, queryable by a much broader set of engineers than the specific service's own team — anyone with general log-search access can now find plaintext passwords, card numbers, or other PII simply by searching for them. Beyond the immediate exposure, this can constitute a genuine compliance violation (PCI-DSS for card data, GDPR/CCPA for PII) with real regulatory and financial consequences, and unlike a code-level secret leak, a log-level leak can span months of historical data across every ingestion the sensitive field ever appeared in.

**Prevention:** Build field-level redaction into shared logging middleware/libraries so individual engineers don't have to remember to redact manually on every log call. Add automated scanning of the logging pipeline for known-sensitive field-name patterns (`password`, `ssn`, `card_number`, `cvv`, `token`) as a CI or pipeline-level check, and treat any confirmed sensitive-data leak into logs as a security incident requiring both a fix and a retention-scoped purge of the exposed data.

---

## Mistake 8: Running Elasticsearch Without Index Lifecycle Management

```json
// WRONG — indices created daily, forever, with no retention policy
// applied at all; disk usage grows monotonically
{
  "index_patterns": ["app-logs-*"],
  "settings": { "number_of_shards": 3 }
  // no ILM policy attached — nothing ever deletes an old index
}
```

**Why it happens:** A fresh Elasticsearch cluster (Chapter 9) has plenty of headroom, and index lifecycle management is genuinely optional configuration — logs flow in and are searchable without it, so nothing looks broken. The gap is invisible until disk usage, which has been climbing since day one, finally crosses a threshold that puts the cluster itself at risk.

**The correct fix:** Configure Index Lifecycle Management (ILM) from the very start of any Elasticsearch deployment, with an explicit policy for how long data stays in each tier and when it's deleted:

```json
{
  "policy": {
    "phases": {
      "hot":    { "min_age": "0ms",  "actions": { "rollover": { "max_age": "1d", "max_size": "50gb" } } },
      "warm":   { "min_age": "7d",   "actions": { "shrink": { "number_of_shards": 1 } } },
      "delete": { "min_age": "30d",  "actions": { "delete": {} } }
    }
  }
}
```

**Impact:** Without ILM, Elasticsearch keeps every index it ever created indefinitely, and disk usage grows without bound as long as logs keep flowing — which they always do. Once disk usage crosses Elasticsearch's watermark thresholds, the cluster starts refusing writes to protect itself, which for a centralized logging pipeline means new logs are silently dropped or backed up at the ingestion layer, at precisely the moment (a disk-space-driven cluster problem) when logs are most needed to diagnose what's going wrong. In the worst case, an unmanaged disk fill can take down the entire Elasticsearch cluster, losing log visibility for every team that depends on it simultaneously.

**Prevention:** Treat ILM configuration as a mandatory, non-optional part of provisioning any Elasticsearch-backed logging pipeline (Chapter 9) — the same category of requirement as authentication or backup. Alert on disk usage and ILM policy health explicitly, well before Elasticsearch's own internal watermarks are reached.

---

## Mistake 9: Tracing That Silently Breaks Across an Async Boundary

```python
# WRONG — trace context is available on the HTTP handler, but is never
# propagated into the message published to the queue
def handle_order(request):
    with tracer.start_as_current_span("handle_order"):
        queue.publish("order.created", {"order_id": request.order_id})
        # no trace context attached to the message at all

# the consumer, running later, in a different process, starts a
# brand-new, disconnected trace with no link back to the original request
def process_order_message(message):
    with tracer.start_as_current_span("process_order"):
        ...
```

**Why it happens:** Distributed tracing (Chapter 11) auto-instruments HTTP and gRPC calls fairly reliably out of the box in most tracing libraries, which creates a false sense that "tracing is set up" once those show up correctly. Message queues, background job schedulers, and other async boundaries are not auto-instrumented the same way — the trace context (trace ID, span ID) has to be explicitly serialized into the message and explicitly extracted by the consumer, and this step is very easy to simply forget, since nothing errors when it's missing.

**The correct fix:** Explicitly propagate trace context across every async boundary, injecting it into the message on publish and extracting it on consume (Chapter 11):

```python
# RIGHT — inject the current trace context into the outgoing message
def handle_order(request):
    with tracer.start_as_current_span("handle_order"):
        headers = {}
        propagator.inject(headers)          # serializes trace_id/span_id
        queue.publish("order.created", {"order_id": request.order_id}, headers=headers)

def process_order_message(message):
    ctx = propagator.extract(message.headers)   # restores the original trace context
    with tracer.start_as_current_span("process_order", context=ctx):
        ...
```

**Impact:** The trace silently "breaks" at exactly the message-queue or background-job hop — the producer-side trace ends looking complete and successful, and the consumer-side trace starts fresh with no link back to the original request, as if the two were entirely unrelated events. During an incident where a background job is the actual source of a delay or failure, an engineer following a trace from the original user request hits a dead end at the queue boundary and has no way to follow the request's actual path through the async processing that happened next — precisely the cross-service visibility distributed tracing exists to provide, missing at the one hop where async processing makes debugging hardest without it.

**Prevention:** Explicitly test trace propagation across every async boundary in the system (message queues, cron/scheduled jobs, background workers) as part of onboarding a new async integration — don't assume auto-instrumentation covers it just because it covered your HTTP layer. Most OpenTelemetry SDKs provide queue-specific instrumentation packages (for Kafka, RabbitMQ, SQS, and similar) that handle this correctly if explicitly added; verify it's actually installed and wired up, not just available.

---

## Mistake 10: One Giant "Kitchen Sink" Dashboard

```text
# WRONG — a single Grafana dashboard, "Everything," with 60+ panels:
# node CPU, node memory, etcd latency, team-checkout's request rate,
# team-search's error rate, team-payments' latency histogram,
# a handful of business KPIs, all on one screen
```

**Why it happens:** It's organizationally easier, early on, for one person or one small team to build a single dashboard everyone can bookmark, rather than setting up per-team dashboard ownership from the start. As more teams and more signals get added over time, nobody wants to be the one to say "this dashboard has become too big," so it keeps growing.

**The correct fix:** Separate cluster-level and application-level dashboards, and give each team ownership of their own application-level dashboard scoped to their own services (Chapter 13):

```text
Platform team dashboard: node health, control plane, schedulable capacity
Team Checkout dashboard: checkout-api's four golden signals + SLO burn rate
Team Search dashboard: search-api's four golden signals + SLO burn rate
```

**Impact:** During an actual incident, nobody can parse a 60-panel dashboard quickly enough for it to be useful — the on-call engineer for one team has to visually hunt through panels belonging to a dozen other teams to find the two or three that matter for the system that's actually paging them. The dashboard becomes a screen everyone has open and nobody can actually read under pressure, which is precisely the moment dashboard clarity matters most.

**Prevention:** Set the norm explicitly from the start of an observability rollout: platform-level dashboards for platform concerns, one dashboard per team/service for application concerns, and resist the organizational pressure to consolidate them "for convenience" as more teams onboard. kube-prometheus-stack's built-in dashboards (Chapter 5, Chapter 13) already give the platform team a strong cluster-level starting point, which removes any practical excuse for folding application-level panels into the same view.

---

## Mistake 11: Dashboards and Alerting Rules as UI Clickops Instead of Code

```text
# WRONG — a dashboard exists only because someone clicked through
# the Grafana UI to build it; a PrometheusRule was created via
# `kubectl edit` directly against the live cluster and never
# committed anywhere
```

**Why it happens:** Building a dashboard in Grafana's UI, or tweaking an alerting threshold with `kubectl edit`, is faster in the moment than going through a Git commit, a PR, and a GitOps sync — especially under the time pressure of actively investigating an incident. The mistake isn't making the change quickly; it's never following up to commit it anywhere afterward.

**The correct fix:** Export the dashboard JSON and commit it to Git, and manage `PrometheusRule`/Alertmanager config exclusively through GitOps-deployed manifests (Chapter 14):

```bash
# Export a UI-built dashboard immediately after building it
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/dashboards/uid/$DASHBOARD_UID" \
  | jq '.dashboard' > dashboards/team-checkout/checkout-golden-signals.json
git add dashboards/team-checkout/checkout-golden-signals.json
git commit -m "Add checkout golden-signals dashboard"
```

**Impact:** A dashboard or alerting rule that exists only as live state in Grafana or the cluster's etcd has no audit trail, no review process, no diff history, and — critically — no recovery path if it's accidentally deleted, if Grafana's own database is lost, or if the cluster is rebuilt from scratch. Teams that have operated this way for months discover the gap only at the worst possible time: after an accidental deletion or a disaster-recovery restore, when the only path forward is rebuilding a complex dashboard entirely from memory, often imperfectly and slower than it originally took to build.

**Prevention:** Treat any dashboard or alerting rule not backed by a file in Git as provisional and at-risk, not as finished work. Where possible, disable direct UI editing of production dashboards entirely (Grafana supports provisioning dashboards as read-only when sourced from files/Git) so the GitOps path is the *only* path, removing the temptation to skip it.

---

## Mistake 12: Copy-Pasting the Same Generic Threshold Alert Onto Every Service

```yaml
# WRONG — the exact same CPU/memory threshold alert, copy-pasted
# onto every service in the fleet regardless of that service's
# actual workload pattern
- alert: HighCPU
  expr: instance_cpu_usage_percent{job="checkout-api"} > 80
  for: 5m
- alert: HighCPU
  expr: instance_cpu_usage_percent{job="batch-report-generator"} > 80
  for: 5m
  # batch-report-generator is SUPPOSED to run near 100% CPU during
  # its nightly batch window — this alert fires every single night
```

**Why it happens:** A generic CPU/memory threshold template is easy to copy across every service's alerting rules in one sweep, and it feels like reasonable due diligence — "every service should have basic resource alerts." It doesn't account for the fact that different workloads have fundamentally different normal operating patterns.

**The correct fix:** Tune each alert to the specific service's actual workload pattern, or replace generic resource alerts entirely with symptom-based, SLO-driven alerts where they make more sense (Chapter 7, Chapter 14):

```yaml
# A batch job's "high CPU" is expected during its run window —
# either exclude that window, or don't alert on CPU for this
# workload type at all; alert on job completion/failure instead
- alert: BatchReportJobFailed
  expr: kube_job_status_failed{job_name=~"batch-report-generator-.*"} > 0
  for: 0m
  labels:
    severity: ticket
```

**Impact:** A workload with a legitimately different normal operating pattern (a nightly batch job that's supposed to run near full CPU, an autoscaled service whose replica-level CPU is expected to vary widely) generates a stream of alerts that are technically accurate but never actually actionable — the CPU really is above 80%, and that's completely fine for this specific service. Over time, this trains the on-call team to recognize "oh, that's just batch-report-generator again" and dismiss it without looking closely, which is exactly the habit that lets a *genuinely* meaningful alert on the same service slip past unnoticed later.

**Prevention:** Treat alerting rules as something to design per-service based on that service's actual behavior, not a template to blanket-apply. When a generic starting template is used, explicitly review and tune (or intentionally remove) each alert against the specific service's real operating characteristics before considering it finished, and revisit thresholds whenever a service's workload pattern changes materially.

---

## Mistake 13: Not Monitoring the Monitoring System Itself

```text
# WRONG — Prometheus, Alertmanager, and the log pipeline have no
# alerts of their own; the assumption is "the monitoring system
# will tell us if something's wrong" — but nothing is watching
# the monitoring system to catch it if IT breaks
```

**Why it happens:** It's an easy blind spot precisely because the monitoring system's whole job is to notice problems — it's natural to assume it will also notice problems with itself. In reality, a stopped scrape target, a full disk on the Prometheus node, a crashed Logstash pipeline, or an Alertmanager instance that's lost its routing config are all failures *of* the observability system, and by definition nothing inside that same broken system is positioned to notice and report them.

**The correct fix:** Monitor the observability stack with an independent, minimal, separate monitoring path — at minimum, an external heartbeat/dead-man's-switch check that something outside the stack itself verifies is still alive:

```yaml
# A "dead man's switch" alert: this rule is designed to ALWAYS fire,
# routed through Alertmanager to an external service (e.g. a
# third-party heartbeat monitor) that pages if it ever STOPS
# receiving this alert — catching the case where Prometheus or
# Alertmanager itself has gone down and can no longer alert on anything
- alert: Watchdog
  expr: vector(1)
  labels:
    severity: none
  annotations:
    description: "This is a permanent alert used as a dead man's switch."
```

```bash
# Independently verify the logging pipeline is actually ingesting,
# from outside the pipeline itself
curl -s "$ELASTICSEARCH_URL/_cluster/health" | jq '.status'
# a script or external check, not a rule living inside the same
# Elasticsearch cluster it's checking
```

**Impact:** A stopped Prometheus scrape target, a disk-full Prometheus node, or a crashed log-shipping pipeline can persist for hours or days with nobody noticing, precisely because the system that would normally have caught it is the thing that's broken. This is discovered, in the worst version of this mistake, only when an unrelated incident happens and the team reaches for dashboards or logs that have actually been silently empty or stale the entire time — meaning they're debugging a real incident with no visibility at exactly the moment they need it most.

**Prevention:** Set up a dead-man's-switch alert (the `Watchdog` pattern shown above is the standard kube-prometheus-stack approach) routed through an independent, external notification path, and periodically and deliberately verify the logging and tracing pipelines are ingesting fresh data via a check that lives outside the pipeline itself.

---

## Mistake 14: Choosing a Tool Based on Hype Instead of Actual Fit

```text
# WRONG — a 3-service startup adopts a full ELK stack because it's
# the most-talked-about logging solution, despite having simple,
# label-based query needs that Loki would serve at a fraction of
# the operational and infrastructure cost
```

**Why it happens:** Popular tools accumulate conference talks, blog posts, and hiring-market visibility that make them feel like the "default, safe" choice, independent of whether their actual design tradeoffs match your situation. It's easier to justify a decision by pointing at what's popular than to do the comparative analysis of your own actual query patterns and existing stack.

**The correct fix:** Choose based on actual query patterns and existing infrastructure fit, as covered explicitly in Chapters 9, 10, and 14 — already running Prometheus and Grafana with simple label-based query needs points toward Loki's lower operational overhead; a genuine need for rich full-text search or compliance-driven SIEM-style queries points toward Elasticsearch, regardless of which one is more visible in the industry at any given moment.

**Impact:** A small team running a full Elasticsearch/Logstash/Kibana stack for query needs Loki would have served comfortably ends up operating a meaningfully heavier, more resource- and operations-intensive system than their actual requirements justify — more infrastructure to patch, scale, and troubleshoot, for capability they never use. The mirror-image mistake (forcing genuine full-text/security-search needs through Loki's label-first model because "we're already using Loki elsewhere") produces the opposite problem: a tool that's operationally lighter but structurally unable to answer the queries the team actually needs, discovered only when a real investigation runs into the tool's design limits.

**Prevention:** Before adopting any observability tool, write down the actual query patterns you expect to need and the infrastructure you already operate, and evaluate candidate tools against that written list — not against which one appeared most recently at a conference. Revisit the choice if your actual usage diverges significantly from what was originally expected.

---

## Summary

| # | Mistake | Key Fix |
|---|---------|---------|
| 1 | Graphing a raw counter | Always wrap counters in `rate()` |
| 2 | Unbounded label cardinality (`pod`, `user_id`) | Stable, bounded labels only; push identifiers to logs/traces |
| 3 | Threshold alert with no `for:` | Require the condition to hold for a meaningful duration |
| 4 | No Alertmanager grouping/inhibition | Configure `group_by` and `inhibit_rules` for root-cause suppression |
| 5 | SLO set at 100% | Set a realistic target with a genuine, usable error budget |
| 6 | Unstructured free-text logging | Structured JSON logs with consistent field names |
| 7 | Sensitive data logged into a shared store | Redact sensitive fields before logging, always |
| 8 | Elasticsearch with no ILM | Configure retention/lifecycle policies from day one |
| 9 | Trace context lost across async boundaries | Explicitly propagate trace context into queues/jobs |
| 10 | One giant kitchen-sink dashboard | Separate cluster-level and per-team application dashboards |
| 11 | Dashboards/alerts as UI clickops | Version-controlled, GitOps-deployed dashboards and rules |
| 12 | Generic threshold alert copy-pasted everywhere | Tune alerts to each service's actual workload pattern |
| 13 | The monitoring system itself isn't monitored | A dead-man's-switch alert and independent pipeline health checks |
| 14 | Tool chosen by hype, not fit | Choose based on actual query patterns and existing stack |

---

## Knowledge Check

1. Why does a raw counter graph always trend upward and then drop suddenly, and what single function fixes it?
2. Explain, mechanically, why a `pod` label on a custom metric is worse in Kubernetes than in a typical long-lived VM-based application.
3. An alerting rule has a correct PromQL expression and a correct severity label but no `for:` field. What specific failure mode does this cause?
4. Why does setting an SLO to 100% actually undermine the purpose of having an SLO at all, rather than just being an ambitious but harmless target?
5. A trace looks complete and successful up until a message is published to a queue, then a brand-new, unrelated trace starts on the consumer side. What is the most likely cause, and what is the fix?
6. Why is "the monitoring system will tell us if something is wrong" an unsafe assumption to rely on without further action?

---

## Hands-On Exercise

**Find and Fix the Broken Observability Setup**

The configuration set below contains at least 5 of the mistakes covered in this chapter. Find every one of them, explain why each is harmful, and rewrite the configuration correctly using the patterns from this course.

```yaml
# 1) Grafana dashboard panel query (this dashboard was built entirely
#    by hand in the Grafana UI and has never been exported to Git)
# Panel: "Checkout Request Volume"
# Query: http_requests_total{job="checkout-api"}
```

```python
# 2) Custom metric instrumentation in checkout-api
orders_processed.labels(
    pod=os.environ["HOSTNAME"],
    status=order.status
).inc()
```

```yaml
# 3) Alerting rule
- alert: CheckoutHighLatency
  expr: histogram_quantile(0.99, checkout_request_duration_seconds_bucket) > 1
  labels:
    severity: page
  # no `for:` field
```

```python
# 4) Logging statement in the payment-processing path
logger.info(f"Charging card: {payment_request.card_number}, cvv: {payment_request.cvv}")
```

```yaml
# 5) SLO definition
slo:
  name: checkout-api-availability
  target: 100%
  window: 30d
```

Steps:

1. List every mistake you can find (aim for at least 5). Hint: check the dashboard's provenance, the metric's labels, the alerting rule's fields, the logging statement's content, and the SLO's target value.
2. Rewrite the dashboard query using `rate()`, and describe the steps you'd take to get this dashboard into Git going forward.
3. Rewrite the metric instrumentation to remove the unbounded label, and explain where the Pod-level detail should live instead if it's ever needed for debugging.
4. Rewrite the alerting rule with an appropriate `for:` duration, and justify the duration you chose.
5. Rewrite the logging statement to redact the sensitive fields.
6. Rewrite the SLO with a realistic target, and calculate the resulting error budget in minutes over the 30-day window.

---

## Further Reading

- [Prometheus Documentation — Instrumentation Best Practices](https://prometheus.io/docs/practices/instrumentation/)
- [Prometheus Documentation — Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Alertmanager Documentation — Inhibition](https://prometheus.io/docs/alerting/latest/configuration/#inhibit_rule)
- [Google SRE Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [OWASP — Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Elastic Documentation — Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [OpenTelemetry Documentation — Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [kube-prometheus-stack — Watchdog Alert Pattern](https://github.com/prometheus-operator/kube-prometheus)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-projects.md">Next: Hands-On Projects →</a>
</div>
