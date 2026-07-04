# Chapter 14 — Best Practices

## Learning Objectives

By the end of this chapter, you will be able to:

- Apply an observability-wide best-practices checklist synthesizing Chapters 1–13 of this course
- Instrument new services for the four golden signals before reaching for exotic custom metrics
- Choose symptom-based, SLO-driven alerting as your primary paging mechanism instead of ad-hoc thresholds
- Correlate logs, metrics, and traces deliberately, rather than treating the three pillars as separate silos
- Manage dashboards, alerting rules, and observability cost as first-class, version-controlled, budgeted infrastructure

---

## Prerequisites for This Chapter

- Metrics fundamentals and cardinality — Chapter 2
- Prometheus architecture — Chapter 3
- PromQL and recording rules — Chapter 4
- Prometheus on Kubernetes — Chapter 5
- Grafana and dashboards-as-code — Chapter 6
- Alerting and Alertmanager — Chapter 7
- Logging fundamentals — Chapter 8
- The ELK stack — Chapter 9
- Loki and modern log aggregation — Chapter 10
- Distributed tracing — Chapter 11
- SLIs, SLOs, and error budgets — Chapter 12
- Observability for Kubernetes — Chapter 13
- Advanced Kubernetes, Chapter 8 (GitOps)

This chapter assumes you've read all of the above — it doesn't reteach any of them, it tells you what to actually *do* with them on a real production observability stack.

---

## Instrument for the Four Golden Signals First

Chapters 2 and 6 introduced latency, traffic, errors, and saturation as the four golden signals — the minimum set of measurements that answer "is this service healthy" for almost any request-driven workload. Before adding a single custom business metric, make sure every service exposes these four:

- **Latency** — request duration, as a histogram (not just an average — Chapter 2 covered why averages hide tail latency), ideally with `le` buckets that let you compute p50/p95/p99 in PromQL.
- **Traffic** — request rate, broken out by the dimensions you'll actually query on (endpoint, method), not by anything unbounded (Section on cardinality below).
- **Errors** — error rate as a ratio of failed to total requests, not just a raw error count (a raw count is meaningless without knowing the traffic volume it happened against).
- **Saturation** — how close the service is to its capacity limit: queue depth, connection pool utilization, or (via Chapter 13) container CPU/memory relative to its configured limits.

A practical starting checklist for any new service, before it ships:

```text
[ ] http_request_duration_seconds (histogram) — latency
[ ] http_requests_total (counter, labeled by method/status/route) — traffic and errors
[ ] A saturation signal specific to this service's actual bottleneck
    (queue depth, DB connection pool usage, CPU/memory vs. limits)
[ ] A ServiceMonitor or scrape config wired up before the first deploy,
    not added retroactively after the first incident
```

Exotic custom metrics (business-specific counters, feature-flag-scoped gauges) are valuable, but they're valuable *in addition to* the four golden signals, not instead of them. A service with twenty custom business metrics and no latency histogram can tell you a lot about what happened and nothing about whether it's healthy right now.

---

## Always Use `rate()` on Counters — Never Graph a Raw Counter

This is a one-line rule from Chapter 4 worth repeating as an explicit, non-negotiable best practice, precisely because it is so easy to get wrong once and never notice: a Prometheus counter only ever goes up, and resets to zero on every process restart. Graphing it directly produces a permanently climbing sawtooth that tells you nothing about the current rate of anything.

```promql
# WRONG — a raw counter graph
http_requests_total

# RIGHT — the rate of change over a window, which is the number
# you actually want on a dashboard or in an alert
rate(http_requests_total[5m])
```

Every counter, in every dashboard panel and every alerting rule, should be wrapped in `rate()` (or `irate()` for fast-moving debugging queries, per Chapter 4's distinction) before it's used for anything except a "total lifetime count" stat panel. If a dashboard panel is a counter metric with no `rate()`, `increase()`, or similar function applied, that is a bug in the dashboard, not a stylistic choice.

---

## Control Cardinality Deliberately, at Instrumentation Time

Chapters 2 and 13 covered why cardinality matters and how Kubernetes' ephemeral Pod identities make it worse than in a typical application. The best-practice discipline is to make this a decision made *once, at instrumentation time*, rather than a problem discovered later and patched with relabeling rules after the fact.

- Use templated route paths, not raw URLs, as a label value: `/users/:id`, never `/users/48291` — otherwise every unique user ID becomes its own time series forever.
- Never put a raw user ID, session ID, request ID, or Kubernetes Pod name/IP on a metric label. These belong in logs and trace spans (high-cardinality-friendly by design), not in metrics.
- Before adding any new label to any metric, ask: "how many distinct values can this take, across the service's entire lifetime?" If the answer is "unbounded" or "grows without limit," it does not belong on a metric label.

```promql
# WRONG — unbounded cardinality baked in at the instrumentation layer
http_requests_total{path="/users/48291", user_id="48291"}

# RIGHT — a bounded, templated label; the specific user ID belongs in a
# log line or trace span if you need to find that specific request later
http_requests_total{path="/users/:id", method="GET", status="200"}
```

Fixing cardinality at the relabeling stage (Chapter 13) is a legitimate safety net for existing problems — it is not a substitute for getting instrumentation right the first time, because every relabeling rule is one more piece of platform configuration that has to be remembered and maintained for the life of that metric.

---

## Alert on Symptoms, Not Just Causes

Chapter 7 introduced alert fatigue and the distinction between symptom-based and cause-based alerting. The best-practice default: **page a human only when user-facing experience is actually degraded** — elevated error rate, elevated latency, a failing SLO. Reserve cause-based alerts (one specific internal dependency slightly elevated, one node's disk climbing toward a warning threshold, a single Pod restarting) for lower-urgency tickets, not pages.

```yaml
# Symptom-based — pages, because it means users are actually affected
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
      /
    sum(rate(http_requests_total[5m])) > 0.05
  for: 5m
  labels:
    severity: page

# Cause-based — files a ticket, because on its own it doesn't mean
# users are affected yet; it's useful context, not an emergency
- alert: ElevatedDatabaseConnectionPoolUsage
  expr: db_connection_pool_used_ratio > 0.8
  for: 15m
  labels:
    severity: ticket
```

The reasoning: a single internal signal being "off" doesn't reliably mean users are impacted — the system may be well within its designed tolerance for that signal to fluctuate. Symptom-based alerts, tied to what a user actually experiences, are a much stronger signal that something genuinely needs a human's immediate attention, and they degrade gracefully as root causes change over time (a symptom-based alert on error rate still fires correctly even if next month's root cause is a completely different dependency than this month's).

---

## Adopt SLOs and Burn-Rate Alerting as Your Primary Paging Mechanism

This is, without qualification, the single most impactful practice in this entire course for reducing alert fatigue. Chapter 12 covered SLIs, SLOs, error budgets, and burn-rate alerting in depth; the best-practice conclusion is to make burn-rate alerts against a defined SLO the *default* paging mechanism for user-facing services, rather than a collection of independently-tuned static thresholds that were each set by a different engineer at a different time for a different reason.

A static threshold ("page if p99 latency exceeds 800ms") is disconnected from any stated tolerance for how bad "bad" actually is, and it doesn't distinguish a brief blip from a sustained, budget-threatening trend. A burn-rate alert directly answers "at the current rate of error-budget consumption, will we breach our SLO soon enough that a human needs to act now" — which is precisely the question a page should be answering.

```yaml
# Fast-burn: exhausting a 30-day error budget within roughly 2 days —
# clearly page-worthy, something is acutely broken right now
- alert: ErrorBudgetFastBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h]))
        /
      sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)
  for: 2m
  labels:
    severity: page

# Slow-burn: exhausting the same budget over roughly 10 days —
# real, but not an emergency; ticket-worthy, investigate this week
- alert: ErrorBudgetSlowBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[6h]))
        /
      sum(rate(http_requests_total[6h]))
    ) > (3 * 0.001)
  for: 15m
  labels:
    severity: ticket
```

Migrating a service from a pile of static thresholds to a small number of SLO-driven burn-rate alerts is usually the single change that most visibly reduces the volume of pages a team receives, without reducing the team's actual ability to detect real user impact quickly — because burn-rate math is specifically designed to catch fast, severe degradations quickly while tolerating brief, minor blips that would have paged a naive static threshold.

---

## Structured Logging Everywhere, Correlated With Trace IDs

Chapters 8 and 11 covered structured (JSON) logging and distributed tracing separately. The best practice that ties them together: every log line emitted during the handling of a request should carry that request's trace ID as a structured field, so an engineer can pivot from a trace span directly to the exact log lines emitted during that span, and from a log line directly to the full distributed trace it was part of.

```json
{
  "timestamp": "2026-07-01T09:14:02.881Z",
  "level": "error",
  "service": "checkout-api",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "message": "payment provider timeout",
  "order_id": "ord_9f2c1a",
  "provider": "stripe"
}
```

With `trace_id` present as a structured, indexed field in both the logging backend (Chapters 9–10) and the tracing backend (Chapter 11), Grafana can wire up a direct link from a Tempo/Jaeger trace span to the exact matching log lines in Loki or Elasticsearch, and vice versa — turning "which logs correspond to this slow trace" from a manual timestamp-and-service-name guessing exercise into a single click.

---

## Choose ELK vs. Loki Based on Actual Query Patterns, Not Hype

Chapters 9 and 10 covered the ELK stack and Loki as two legitimate, different approaches to log aggregation, with a real architectural difference: Elasticsearch indexes full log content for rich full-text search, Loki indexes only a small set of labels and treats log content as an unindexed stream. The best practice is choosing between them based on your actual needs, not on which one has more conference talks this year.

- **Already on Prometheus and Grafana?** Loki's label model deliberately mirrors Prometheus's, and LogQL's syntax deliberately mirrors PromQL's — the operational and mental-model overlap is substantial, and Grafana's native Loki integration is first-class. This is usually the lower-total-cost choice for a team already running the Prometheus/Grafana stack.
- **Need rich full-text search, complex boolean queries across unindexed content, or a security/compliance use case that demands searching raw log bodies (SIEM-adjacent use cases)?** Elasticsearch's full-content indexing is built exactly for this, at the cost of significantly higher storage and operational overhead per unit of log volume.

Neither is a universally "better" choice — a three-service startup adopting a full ELK stack because it's the most famous logging solution, when their actual query patterns are simple label-based filters Loki would serve at a fraction of the operational cost, is over-engineering. The reverse — a security team with heavy compliance-driven full-text search needs trying to force those queries through Loki's label-first model — is under-engineering in the other direction. Match the tool to the query pattern.

---

## Sample Traces Deliberately

Chapter 11 covered trace sampling strategies. The best practice sits between two failure modes: tracing 100% of requests on a high-volume service (expensive in both storage and the performance overhead of instrumentation on every single request) and not tracing at all (leaving you completely blind to cross-service latency and failure propagation when you need it most, typically during an incident).

```yaml
# Deliberate sampling: always capture errors and slow requests in full,
# sample a small percentage of normal, healthy traffic for baseline visibility
sampling:
  default_strategy:
    type: probabilistic
    param: 0.05        # 5% of ordinary traffic
  per_operation_strategies:
    - operation: "checkout.process_payment"
      type: probabilistic
      param: 0.25       # sample more heavily on your most critical path
  # tail-based sampling (sampled after the fact, at the collector) is
  # preferable when available — it lets you always keep error traces
  # and slow traces at 100%, while still sampling healthy traffic down
```

Tail-based sampling, where the decision to keep or discard a trace is made *after* the full trace completes (so you can guarantee 100% retention of anything that errored or ran slow, while sampling healthy traffic down aggressively), gives the best of both worlds where your tracing backend supports it — full visibility into exactly the traces you'd want during an incident, at a fraction of the storage cost of tracing everything uniformly.

---

## Separate Cluster-Level and Application-Level Dashboards, With Per-Team Ownership

Chapter 13 covered why cluster-level and application-level monitoring are different concerns for different audiences. The best practice extends this into an ownership model, not just a dashboard-layout preference: every team owns and maintains its own application-level dashboards, and the platform team owns cluster-level dashboards (largely inherited from kube-prometheus-stack's defaults) — nobody maintains a single sprawling dashboard on behalf of everyone else.

This scales far better than a central "observability team" owning every dashboard for every service, because the team that actually understands a service's specific failure modes and traffic patterns is the team best positioned to know which panels actually matter for it. A centrally-maintained dashboard for a service the maintaining team doesn't operate day to day tends to drift out of relevance quickly — panels for metrics nobody looks at anymore, missing panels for a new failure mode nobody told the central team about.

---

## Dashboards and Alerting Rules as Code, in Git

Chapter 6 covered dashboards-as-code and Advanced Kubernetes Chapter 8 covered GitOps generally. The best practice is applying that discipline completely to the observability stack: Grafana dashboard JSON, Prometheus alerting and recording rules, and Alertmanager routing configuration all live in Git and deploy through the same GitOps pipeline as application manifests — never hand-edited through the Grafana UI or `kubectl edit`'d directly against a live `PrometheusRule`.

```text
observability-gitops-repo/
├── dashboards/
│   ├── platform/
│   │   └── cluster-health.json
│   └── team-checkout/
│       └── checkout-golden-signals.json
├── alerting-rules/
│   └── team-checkout/
│       └── checkout-slo-burn-rate.yaml
└── alertmanager/
    └── routing.yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: checkout-slo-burn-rate
  namespace: production
  labels:
    release: kube-prometheus-stack     # picked up automatically by the Operator
spec:
  groups:
    - name: checkout.slo
      rules:
        - alert: ErrorBudgetFastBurn
          expr: ... # as defined earlier in this chapter
          for: 2m
          labels: { severity: page }
```

A dashboard built by hand in the Grafana UI and never exported to Git is, functionally, a piece of undocumented, unreviewed, unbacked-up production configuration — indistinguishable in risk profile from a manually `kubectl edit`'d Deployment. A cluster rebuild, an accidental deletion, or simple staff turnover means starting over from memory rather than a `git apply`. Treat a dashboard or alerting rule as "real" only once it has a corresponding file in version control.

---

## Budget for Observability Cost Explicitly

Chapter 13 named cardinality and retention as the two dominant cost drivers for an observability stack at scale. The best practice is treating that cost as a first-class infrastructure line item from the start of a project, not a number discovered later via a surprise bill or a full disk:

- Set an explicit retention window per data type (raw high-resolution metrics, downsampled rollups, raw logs, sampled traces) as a deliberate decision, documented alongside the reasoning, not left at whatever the tool's default happens to be.
- Set cardinality limits or alerts (Chapter 13's `prometheus_tsdb_head_series` watch) so a cardinality regression is caught within hours, not discovered six weeks later as an OOM.
- Review the actual cost breakdown (storage, ingestion, and if applicable per-series billing on a managed vendor) on the same recurring cadence you'd review any other significant infrastructure cost — quarterly, at minimum.

---

## Regularly Review and Prune Dashboards and Alerting Rules

Dashboards and alerting rules accumulate the same way stale code does, and for the same underlying reason: it's much cheaper, in the moment, to add a new panel or a new alert than to go back and remove one that's no longer useful. Left unchecked, this accumulates in two damaging ways:

- **Dead metrics from decommissioned services** linger in dashboards and recording rules long after the service itself is gone, occasionally still consuming scrape and storage resources if the `ServiceMonitor` was never cleaned up alongside the service.
- **Stale alerts nobody remembers the purpose of** keep firing (or keep silently *not* firing, having been quietly muted by someone during an old incident and never un-muted) and erode trust in the alerting system as a whole — once engineers learn that some fraction of alerts are safe to ignore, they start second-guessing all of them, which is the seed of exactly the alert fatigue Chapter 7 warned about.

Put a recurring review on the calendar — quarterly is a reasonable minimum — where every active alerting rule and every dashboard is checked against two questions: does the thing it monitors still exist, and has it fired/been viewed usefully in the review period. Anything that fails both is a strong candidate for deletion, not indefinite retention "just in case."

---

## Summary

- Instrument every new service for the four golden signals (latency, traffic, errors, saturation) before adding custom business metrics (Ch. 2, 6).
- Never graph or alert on a raw counter — always wrap it in `rate()` (Ch. 4).
- Control cardinality at instrumentation time with templated paths and bounded label values, not as a retroactive fix (Ch. 2, 13).
- Prefer symptom-based alerting (is the user actually affected) as your paging trigger; reserve cause-based alerts for tickets (Ch. 7).
- Adopt SLOs and burn-rate alerting as your primary paging mechanism — the single highest-impact practice in this course for cutting alert fatigue (Ch. 12).
- Correlate structured logs with trace IDs so any log line or trace span can pivot directly to the other (Ch. 8, 11).
- Choose ELK vs. Loki based on your actual query patterns and existing stack, not hype (Ch. 9, 10).
- Sample traces deliberately — never 100% of high-volume traffic, never nothing (Ch. 11).
- Keep cluster-level and application-level dashboards separate, each owned by the team that actually understands that layer (Ch. 13).
- Treat dashboards and alerting rules as code in Git, deployed via GitOps, never hand-edited in a UI (Ch. 6; Advanced Kubernetes, Ch. 8).
- Budget observability cost (retention, cardinality, downsampling) explicitly and review it on a recurring cadence (Ch. 13).
- Regularly prune dead metrics and stale alerts before they silently erode trust in the whole system.

---

## Knowledge Check

1. Why should the four golden signals be instrumented before any custom business metric, rather than after?
2. What specifically is wrong with graphing `http_requests_total` directly on a dashboard panel, and what is the one-line fix?
3. Give an example of a symptom-based alert and a cause-based alert for the same underlying database problem, and explain why only one of them should typically page a human.
4. Why is SLO-driven burn-rate alerting described as the single most impactful practice in this course for reducing alert fatigue, rather than just "one good practice among many"?
5. A three-person startup with one Prometheus/Grafana stack is deciding between ELK and Loki for their first centralized logging setup. Which factors from this chapter should drive that decision?
6. Why is a Grafana dashboard that only exists because someone built it by hand in the UI considered a risk, even if it currently works fine?

---

## Hands-On Exercise

1. Take an existing service (or a small sample app) and audit its current metrics against the four-golden-signals checklist in this chapter. Add whichever of the four is missing.
2. Find any dashboard panel or alerting rule in your own environment (or a sample `kube-prometheus-stack` deployment) that graphs a raw counter without `rate()`, and fix it.
3. Write one symptom-based alert (based on an SLO burn rate, using the pattern in this chapter) and one cause-based alert (a specific internal signal) for the same service, and route them to different severities in Alertmanager (`page` vs. `ticket`).
4. Add a `trace_id` field to a sample service's structured log output, and verify in your logging backend that you can filter logs down to exactly the ones belonging to one specific trace.
5. Export one Grafana dashboard to JSON, commit it to a Git repository, and describe how you would wire it into a GitOps pipeline (Argo CD or Flux) so it's redeployed automatically on every commit.

---

## Further Reading

- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Prometheus Documentation — Instrumentation Best Practices](https://prometheus.io/docs/practices/instrumentation/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Grafana Documentation — Provisioning Dashboards and Alerting as Code](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [OpenTelemetry Documentation — Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [Grafana Loki Documentation — When to Use Loki](https://grafana.com/docs/loki/latest/get-started/overview/)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./13-observability-for-kubernetes.md">← Previous: Observability for Kubernetes</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./15-common-mistakes.md">Next: Common Mistakes and Pitfalls →</a>
</div>
