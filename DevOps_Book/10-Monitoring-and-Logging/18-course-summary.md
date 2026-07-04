# Chapter 18 — Course Summary & Next Steps

## 18.1 What You've Learned

Congratulations on completing **Monitoring & Logging** — Topic 10 of the DevOps Learning Path. You started this course able to deploy, scale, secure, and operate applications on Kubernetes with no real way to answer "is it actually healthy right now, and why isn't it." You finish it able to instrument, monitor, alert on, and debug a distributed system end to end — the skill set every previous course quietly assumed would eventually exist.

| Chapter | What You Can Do Now |
|---------|----------------------|
| 01 Introduction to Observability | Explain the difference between monitoring and observability, and place the three pillars (metrics, logs, traces) in context |
| 02 Metrics Fundamentals | Define counters, gauges, histograms, and summaries correctly, and reason about cardinality before it becomes a production incident |
| 03 Prometheus Architecture | Explain the pull-based scraping model, the TSDB, service discovery, and exporters |
| 04 PromQL and Querying | Write `rate`/`irate` queries, aggregation expressions, and recording rules with confidence |
| 05 Prometheus on Kubernetes | Deploy the Prometheus Operator and kube-prometheus-stack, and wire up `ServiceMonitor`/`PodMonitor` scraping |
| 06 Grafana and Visualization | Build meaningful dashboards, manage data sources and variables, and treat dashboards as code |
| 07 Alerting and Alertmanager | Design alerting rules and route them through Alertmanager's grouping/routing/inhibition pipeline without causing alert fatigue |
| 08 Logging Fundamentals | Explain the collect-ship-store-query pipeline and the case for structured, centralized logging |
| 09 The ELK Stack | Deploy Elasticsearch, Logstash, and Kibana, and build log ingestion pipelines |
| 10 Loki and Modern Log Aggregation | Deploy Loki and Promtail, write LogQL queries, and choose correctly between Loki and ELK |
| 11 Distributed Tracing | Instrument applications with OpenTelemetry, visualize traces in Jaeger/Tempo, and correlate traces with logs and metrics |
| 12 SLIs, SLOs, and Error Budgets | Define SLIs and SLOs, calculate error budgets, and design multi-window burn-rate alerts |
| 13 Observability for Kubernetes | Apply kube-state-metrics and cAdvisor correctly, and manage cardinality explosion at Kubernetes scale |
| 14 Best Practices | Apply production-grade observability patterns across metrics, logs, traces, and alerting |
| 15 Common Mistakes | Recognize and avoid the most frequent monitoring/logging/alerting mistakes before they become incidents |
| 16 Projects | Instrumented an app with real metrics, defined and alerted on an SLO, correlated all three pillars in one Grafana view, and designed a multi-team observability platform |
| 17 Interview Preparation | Answer SRE/observability-engineer-level foundational, architectural, scenario, and system-design questions with confidence |

---

## 18.1.1 The Mental Model, in One Paragraph

Every prior course in this roadmap taught you to build and operate systems; this course taught you to *see* them. The unifying idea across all eighteen chapters is that observability is not one tool but a discipline of turning raw signal into a decision: a metric alone is just a number, a `rate()` and a threshold turn it into a trend, an SLO turns the trend into a reliability commitment, and a burn-rate alert turns the commitment into "should a human be paged right now, or can this wait." Logs and traces exist for the moment that decision requires a *why*, not just a *that* — a metric tells you latency spiked, a trace tells you which downstream call caused it, and a log tells you the exact error inside that call. Once you see metrics, logs, and traces as three views of the same underlying reality rather than three separate tools with three separate learning curves, the entire stack — Prometheus, Grafana, Loki, Tempo, Alertmanager — stops being a list of software to install and becomes one coherent answer to a single question: what is this system actually doing, right now, and is that okay?

---

## 18.2 Completion Checklist

```
Metrics Fundamentals & PromQL:
  [ ] Can explain the difference between a counter, gauge, histogram, and summary
  [ ] Can identify a cardinality risk in a metric's label set before deploying it
  [ ] Can write rate()/irate() queries and explain when to use each
  [ ] Can write a histogram_quantile() query correctly, including the by (le) aggregation
  [ ] Can write and explain the purpose of a Prometheus recording rule

Kubernetes-Native Monitoring:
  [ ] Can deploy kube-prometheus-stack via Helm and explain what each component does
  [ ] Can write a ServiceMonitor/PodMonitor and diagnose why one isn't being scraped
  [ ] Can explain the difference between kube-state-metrics and cAdvisor
  [ ] Can apply cardinality-limiting relabel rules cluster-wide

Visualization & Alerting:
  [ ] Can build a Grafana dashboard with panels backed by real PromQL queries
  [ ] Can manage dashboards and alerting rules as code (ConfigMaps/PrometheusRule)
  [ ] Can design an Alertmanager routing tree with grouping and inhibition rules
  [ ] Can distinguish a well-designed alert (actionable, owned, has a runbook) from a noisy one

Logging:
  [ ] Can explain the collect-ship-store-query pipeline end to end
  [ ] Can choose correctly between Loki and ELK for a given use case and justify it
  [ ] Can write a LogQL query combining a label selector, a line filter, and a metric extraction
  [ ] Can design a retention/sampling strategy that cuts cost without losing debuggability

Tracing:
  [ ] Can instrument an application with OpenTelemetry and export to Tempo/Jaeger
  [ ] Can explain context propagation and identify where it breaks (async/queue boundaries)
  [ ] Can correlate a trace to its exact log lines via a shared trace ID

SRE Practices:
  [ ] Can define an SLI/SLO pair and calculate its error budget by hand
  [ ] Can write a multi-window, multi-burn-rate alerting rule and explain why it beats a static threshold
  [ ] Can design a per-team alert routing strategy that avoids a single noisy shared channel

Projects Completed:
  [ ] Project 1: App instrumented with counter/histogram/gauge metrics, scraped and dashboarded
  [ ] Project 2: SLO defined, error budget calculated, burn-rate alert firing to Slack
  [ ] Project 3: Metrics, Loki logs, and Tempo traces correlated in one Grafana view via trace ID
  [ ] Project 4: Multi-team observability platform with cardinality limits, per-team routing, and a meta-monitoring runbook
```

If any row above still feels shaky, that's a signal, not a failure — go back to the relevant chapter and redo the hands-on exercise before moving to Topic 11. Observability skills, like platform engineering skills, are trusted in an interview (and in production) only once they're muscle memory, not just something you've read about once.

---

## 18.3 PromQL / LogQL Quick Reference

```promql
# ── PromQL ───────────────────────────────────────────────────────────────

# Rate of a counter (requests/sec over the last 5 minutes)
rate(http_requests_total[5m])

# Rate broken down by a label, summed across all instances
sum(rate(http_requests_total[5m])) by (status)

# Fast-moving, noisy instantaneous rate — dashboards only, never alerting
irate(http_requests_total[5m])

# p95 latency from a histogram — the by (le) grouping is mandatory
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# p95 latency per route — aggregate by le AND the dimension you care about
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))

# Aggregation: "by" keeps only the named labels, "without" drops only the named labels
sum(rate(http_requests_total[5m])) by (job)
sum(rate(http_requests_total[5m])) without (instance)

# Error ratio (fraction of 5xx out of total)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Recording rule syntax — precomputes an expensive/frequent query
groups:
  - name: example
    rules:
      - record: job:http_errors:ratio_5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

# ── LogQL ────────────────────────────────────────────────────────────────

# Label selector + line filter (the most common LogQL query shape)
{namespace="demo-app", container="hello-metrics"} |= "error"

# Label selector + JSON parsing + field filter
{namespace="demo-app"} | json | duration_ms > 300

# Negated line filter
{namespace="demo-app"} != "healthcheck"

# Metric query derived from logs — count of matching lines per second
sum(rate({namespace="demo-app"} |= "error" [5m]))

# Metric query extracting a numeric field from structured logs
quantile_over_time(0.95, {namespace="demo-app"} | json | unwrap duration_ms [5m])
```

---

## 18.4 Observability Stack Quick Reference

| Component | Purpose | This Course's Chapter |
|-----------|---------|------------------------|
| Prometheus | Pull-based metrics collection and time-series storage; the query engine behind PromQL | 03, 04, 05 |
| Alertmanager | Deduplicates, groups, routes, and inhibits alerts fired by Prometheus rules | 07 |
| Grafana | Dashboards and visualization across Prometheus, Loki, and Tempo data sources | 06 |
| Loki | Label-indexed, cost-efficient log aggregation queried with LogQL | 10 |
| Promtail | Kubernetes-aware log collector shipping container logs to Loki, auto-labeled from Pod metadata | 10 |
| Elasticsearch / Logstash / Kibana (ELK) | Full-text-indexed log storage and search, for workloads that need rich ad hoc querying | 09 |
| Tempo / Jaeger | Distributed trace storage and visualization, queried by trace ID or span attributes | 11 |
| OpenTelemetry | Vendor-neutral instrumentation standard for producing traces, metrics, and logs, and propagating context across services | 11 |
| kube-state-metrics | Exposes Kubernetes object state (Deployments, Pods, PVCs) as Prometheus metrics | 13 |
| cAdvisor | Exposes per-container resource usage (CPU, memory, network) as Prometheus metrics, built into the kubelet | 13 |

---

## 18.5 What's Next: Topic 11 — Security (DevSecOps)

You now know how to see what your systems are doing — metrics, logs, and traces, tied together with SLOs and alerting that pages a human only when it matters. What this course deliberately did not cover in depth is turning that same instinct toward a different category of signal: security. Topic 11 covers secrets management, SAST/DAST (static and dynamic application security testing), supply chain security (SBOMs, image signing, dependency scanning), and compliance — the discipline of building security into the pipeline rather than bolting it on afterward.

The jump is smaller than it looks, because you already have the core instinct. This course's Chapter 7 taught you to alert on anomalies in latency and error rate without drowning on-call in noise; Topic 11 extends that exact same alerting discipline to security-specific signals — a spike in failed authentication attempts, an unexpected outbound connection, a container running with unexpected privileges — using the same grouping, routing, and inhibition patterns from Alertmanager so security alerts don't become their own separate source of fatigue. Advanced Kubernetes Chapter 13's audit logging, which you used to reconstruct "what happened" during a cluster incident, becomes one of Topic 11's core data sources for detecting "what happened" during a security incident — the same log, a different lens.

| What you learned in this course | How it maps to Security (DevSecOps) |
|---|---|
| Alerting design without alert fatigue (Chapter 7) | Security alerting (failed logins, policy violations, anomalous access) uses the same grouping/routing/inhibition discipline to stay actionable |
| SLIs/SLOs/error budgets (Chapter 12) | Security posture gets its own measurable targets (e.g., "% of images scanned before deploy," "mean time to patch a critical CVE") using the same SLI/SLO framing |
| Centralized logging (Chapters 08–10) | Security investigations depend on the same centralized, structured, correlated logs you built for debugging — now queried for suspicious patterns instead of errors |
| Distributed tracing and correlation (Chapter 11) | Tracing a request across services is the same skill needed to trace an attacker's lateral movement across services during an incident |
| Kubernetes observability (Chapter 13) | Runtime security monitoring (detecting an unexpected process inside a container) builds on the same metrics/exporter pattern used for cAdvisor and kube-state-metrics |
| Best practices and common mistakes (Chapters 14–15) | The same "why do teams get this wrong" analytical lens applies directly to security anti-patterns in Topic 11 |

---

## 18.6 The Full Picture

```
Topics 1–4:   Core foundations
  Linux, Networking, Git, Docker

Topics 5–7:   Cloud and automation
  CI/CD Pipelines, AWS, Terraform

Topics 8–9:   Container orchestration
  Kubernetes Basics, Advanced Kubernetes

Topic 10:     Seeing the system                (YOU ARE HERE — complete)
  Monitoring & Logging

Topic 11:     Securing the system               (Coming soon)
  Security (DevSecOps)
```

You are now 10 of 11 courses complete — only **one topic remains** in the entire roadmap. Everything from here builds directly on this course: the dashboards and alerts you built will be the same ones that flag a security anomaly once Topic 11 adds security-specific signals to them; the audit logs and centralized logging pipeline you now know how to query will double as the forensic record during a security incident; and the SLO discipline from Chapter 12 extends naturally into security posture targets. If Chapters 1–17 felt solid, Topic 11 will feel like pointing a stack you already trust at one more category of problem — not learning a new discipline from scratch.

---

## 18.7 Recommended Resources

- **Prometheus official documentation** (prometheus.io/docs) — the authoritative reference for PromQL, the data model, and exporter ecosystem this entire course is built on
- **Grafana documentation** (grafana.com/docs) — dashboard design, data source configuration, and alerting features beyond what Chapter 6 had room to cover
- **"Site Reliability Engineering"** by Google, edited by Betsy Beyer, Chris Jones, Jennifer Petoff, and Niall Richard Murphy (O'Reilly, free online at sre.google/sre-book) — the canonical text that defines SLIs, SLOs, and error budgets; required reading to go deeper than Chapter 12
- **"The Site Reliability Workbook"** by Google, edited by Betsy Beyer et al. (O'Reilly, free online at sre.google/workbook) — the practical, exercise-driven companion to the first book, with worked SLO and alerting examples
- **"Observability Engineering"** by Charity Majors, Liz Fong-Jones, and George Miranda (O'Reilly) — the best available argument for observability as a distinct discipline from monitoring, and a deeper treatment of high-cardinality, high-dimensionality telemetry than this course's introduction could cover
- **OpenTelemetry documentation** (opentelemetry.io/docs) — the canonical reference for instrumentation, context propagation, and exporter configuration across every language, extending Chapter 11 well past its introductory depth
- **CNCF Cloud Native Landscape — Observability and Analysis category** (landscape.cncf.io) — now that you've used real projects from it (Prometheus, Grafana, Loki, OpenTelemetry), it's worth revisiting to see the surrounding ecosystem and what else exists for specialized needs

None of these replace hands-on repetition. Keep the observability stack from this course's projects running, point it at something real, and deliberately break things to watch how the metrics, logs, and traces react — the fastest way to retain everything in this course is to keep operating a real (even if small) observability pipeline the way an SRE team would.

---

## 18.8 Progress Tracker

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | Complete |
| 4 | [Docker](../04-Docker/00-index.md) | Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | Complete |
| 6 | [Cloud Fundamentals (AWS)](../06-Cloud-Fundamentals-AWS/00-index.md) | Complete |
| 7 | [Infrastructure as Code (Terraform)](../07-Infrastructure-as-Code-Terraform/00-index.md) | Complete |
| 8 | [Kubernetes Basics](../08-Kubernetes-Basics/00-index.md) | Complete |
| 9 | [Advanced Kubernetes](../09-Advanced-Kubernetes/00-index.md) | Complete |
| 10 | Monitoring & Logging | Complete — you are here |
| 11 | Security (DevSecOps) | Coming soon |

Continue to: **[Security (DevSecOps)](../11-Security-DevSecOps/00-index.md)**

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./17-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../11-Security-DevSecOps/00-index.md">Next: Security (DevSecOps) →</a>
</div>
