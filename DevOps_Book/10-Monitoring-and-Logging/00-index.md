# Monitoring & Logging — Complete Course Index

> **DevOps Learning Path — Topic 10 of 11**

---

## Course Overview

Every previous course in this roadmap ended with some version of the same admission: "full observability depth is out of scope here — that's Topic 10." This is that course.

You can deploy applications (Docker, Kubernetes Basics), scale and expose them (Kubernetes Basics), secure and operate the cluster they run on (Advanced Kubernetes) — but none of that tells you whether the system is actually healthy *right now*, why a request was slow, or how much downtime you can tolerate before it becomes a business problem. That is the job of observability: metrics, logs, and traces, tied together with alerting and the SRE discipline of SLIs/SLOs/error budgets.

This course teaches the industry-standard open-source observability stack — Prometheus and Grafana for metrics, the ELK stack and Loki for logs, OpenTelemetry and Jaeger/Tempo for distributed tracing — and how to wire all of it into the Kubernetes clusters you built in Topics 8 and 9. It ends with the SRE frameworks (SLIs, SLOs, error budgets) that turn raw telemetry into decisions about what to fix first and when to page a human.

---

## What You'll Be Able to Do

- Explain the three pillars of observability (metrics, logs, traces) and when each one is the right tool
- Understand Prometheus's architecture, data model, and pull-based scraping model
- Write PromQL queries, including rate calculations, aggregations, and recording rules
- Deploy Prometheus and Grafana on Kubernetes using the Prometheus Operator (building directly on Advanced Kubernetes, Chapter 5)
- Build meaningful Grafana dashboards and manage them as code
- Design alerting rules and route them through Alertmanager without causing alert fatigue
- Set up centralized, structured logging with the ELK stack and with Loki, and explain the tradeoffs between them
- Implement distributed tracing with OpenTelemetry and correlate traces with logs and metrics
- Define SLIs and SLOs, calculate error budgets, and design burn-rate alerts
- Apply Kubernetes-specific observability patterns (kube-state-metrics, cAdvisor, cardinality management)
- Apply production-grade observability best practices and avoid the most common monitoring/logging mistakes
- Answer observability and SRE interview questions, including incident-response scenarios

---

## Prerequisites

- **Kubernetes Basics (Topic 8)** — required. Prometheus and Grafana are deployed on Kubernetes throughout this course.
- **Advanced Kubernetes (Topic 9)** — required, specifically Chapter 5 (Custom Resources and Operators) and Chapter 13 (Auditing and Troubleshooting at Scale), which this course builds on directly.
- **Docker (Topic 4)** — assumed throughout.

---

## Estimated Learning Timeline

**4–5 weeks** at 1–2 hours/day.

---

## Learning Milestones

| Milestone | Chapters | Skills Unlocked |
|-----------|----------|-----------------|
| Metrics Foundations | 01–04 | Observability theory, Prometheus's data model, PromQL |
| Metrics in Production | 05–07 | Prometheus Operator on Kubernetes, Grafana, alerting |
| Logs and Traces | 08–11 | Centralized logging (ELK, Loki), distributed tracing |
| SRE and Kubernetes Depth | 12–13 | SLIs/SLOs/error budgets, Kubernetes-specific observability |
| Professional | 14–18 | Best practices, avoiding common mistakes, capstone projects, interview-ready |

---

## Full Chapter Index

| # | Chapter | Topics |
|---|---------|--------|
| 01 | [Introduction to Observability](./01-introduction.md) | Monitoring vs. observability, the three pillars, history (Nagios → Prometheus), why this course exists |
| 02 | [Metrics Fundamentals](./02-metrics-fundamentals.md) | What a metric is, counters/gauges/histograms/summaries, time-series data, cardinality |
| 03 | [Prometheus Architecture](./03-prometheus-architecture.md) | Pull-based scraping, the TSDB, service discovery, exporters |
| 04 | [PromQL and Querying](./04-promql-and-querying.md) | PromQL syntax, `rate`/`irate`, aggregation operators, recording rules |
| 05 | [Prometheus on Kubernetes](./05-prometheus-on-kubernetes.md) | The Prometheus Operator, `ServiceMonitor`/`PodMonitor`, kube-prometheus-stack, kube-state-metrics, node-exporter |
| 06 | [Grafana and Visualization](./06-grafana-and-visualization.md) | Grafana architecture, dashboards, data sources, variables, dashboards-as-code |
| 07 | [Alerting and Alertmanager](./07-alerting-and-alertmanager.md) | Alerting rules, Alertmanager routing/grouping/silencing, notification channels, alert fatigue |
| 08 | [Logging Fundamentals](./08-logging-fundamentals.md) | Why centralized logging, structured logging, log levels, the collect-ship-store-query pipeline |
| 09 | [The ELK Stack](./09-elk-stack.md) | Elasticsearch, Logstash, Kibana, ingestion pipelines |
| 10 | [Loki and Modern Log Aggregation](./10-loki-and-log-aggregation.md) | Loki's label-based model, Promtail/Fluent Bit, LogQL, ELK vs. Loki |
| 11 | [Distributed Tracing](./11-distributed-tracing.md) | Spans and traces, OpenTelemetry, Jaeger/Tempo, correlating traces with logs and metrics |
| 12 | [SLIs, SLOs, and Error Budgets](./12-slis-slos-and-error-budgets.md) | SRE fundamentals, error budgets, burn-rate alerting |
| 13 | [Observability for Kubernetes](./13-observability-for-kubernetes.md) | kube-state-metrics vs. cAdvisor, cluster vs. app-level monitoring, cardinality explosion in Kubernetes |
| 14 | [Best Practices](./14-best-practices.md) | Production-grade observability patterns |
| 15 | [Common Mistakes and Pitfalls](./15-common-mistakes.md) | The most frequent monitoring/logging/alerting mistakes, why they happen, how to fix them |
| 16 | [Hands-On Projects](./16-projects.md) | Beginner → intermediate → advanced → production-grade capstone projects |
| 17 | [Interview Preparation](./17-interview-preparation.md) | Observability and SRE interview questions, incident-response scenarios |
| 18 | [Course Summary](./18-course-summary.md) | Recap, checklist, quick reference, what's next |

---

## DevOps Roadmap Series

| # | Topic | Status |
|---|-------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | ✅ Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | ✅ Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | ✅ Complete |
| 4 | [Docker](../04-Docker/00-index.md) | ✅ Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | ✅ Complete |
| 6 | [Cloud Fundamentals AWS](../06-Cloud-Fundamentals-AWS/00-index.md) | ✅ Complete |
| 7 | [Infrastructure as Code (Terraform)](../07-Infrastructure-as-Code-Terraform/00-index.md) | ✅ Complete |
| 8 | [Kubernetes Basics](../08-Kubernetes-Basics/00-index.md) | ✅ Complete |
| 9 | [Advanced Kubernetes](../09-Advanced-Kubernetes/00-index.md) | ✅ Complete |
| 10 | **Monitoring & Logging** | 📍 You are here |
| 11 | Security (DevSecOps) | Coming soon |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction.md">Next: Introduction to Observability →</a>
</div>
