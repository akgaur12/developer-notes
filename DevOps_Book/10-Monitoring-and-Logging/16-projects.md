# Chapter 16 — Hands-On Projects

## Learning Objectives

By the end of this chapter you will have:

- Instrumented a real application with Prometheus client metrics, scraped it automatically via a `ServiceMonitor`, and built a Grafana dashboard from those metrics
- Defined a real SLO, calculated its error budget, and wired a multi-window burn-rate alert through to Slack
- Correlated metrics, logs, and traces for a single request in one Grafana view — the three pillars working together instead of as three separate tools
- Designed a shared, multi-team observability platform with cardinality controls, per-team routing, and a meta-monitoring runbook, suitable as a portfolio piece for a platform/SRE role

---

## Project 1 — Metrics and Dashboards for an App (Beginner)

**Goal:** Take a small application running on the `kind` cluster from Advanced Kubernetes Chapter 5, instrument it with a Prometheus client library, scrape it automatically via a `ServiceMonitor`, and build a Grafana dashboard that visibly reacts to load.

**Architecture:**

```
kind cluster
  └── namespace: monitoring
        └── kube-prometheus-stack (Prometheus + Alertmanager + Grafana + Operator)
  └── namespace: demo-app
        ├── Deployment: hello-metrics (exposes /metrics on :8080)
        ├── Service: hello-metrics (port 8080, labeled for discovery)
        └── ServiceMonitor: hello-metrics (tells Prometheus to scrape it)
```

**Step 1 — Install kube-prometheus-stack** (Chapter 5 — this single Helm chart installs the Prometheus Operator, Prometheus, Alertmanager, Grafana, and the CRDs like `ServiceMonitor` that everything else in this course depends on):

```bash
kind create cluster --name observability
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

kubectl get pods -n monitoring   # wait for everything to reach Running
```

**Step 2 — Instrument a minimal app.** A small Python/Flask app using `prometheus_client` (any language's client library follows the same three-metric pattern from Chapter 2 — counter, histogram, gauge):

```python
# app.py
import random, time
from flask import Flask
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

REQUEST_COUNT = Counter(
    "hello_requests_total", "Total requests received", ["method", "path", "status"]
)
REQUEST_LATENCY = Histogram(
    "hello_request_duration_seconds", "Request latency in seconds",
    buckets=[0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5]
)
INFLIGHT_REQUESTS = Gauge(
    "hello_inflight_requests", "Number of requests currently being processed"
)

@app.route("/")
def hello():
    INFLIGHT_REQUESTS.inc()
    start = time.time()
    time.sleep(random.uniform(0.05, 0.4))   # simulate variable work
    REQUEST_LATENCY.observe(time.time() - start)
    REQUEST_COUNT.labels(method="GET", path="/", status="200").inc()
    INFLIGHT_REQUESTS.dec()
    return "hello\n"

@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

This deliberately covers all three metric types from Chapter 2 in one app: a `Counter` that only goes up (requests by status), a `Histogram` that buckets latency (needed for `histogram_quantile` later), and a `Gauge` that goes up and down (in-flight requests).

**Step 3 — Build and deploy** (`manifests/deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-metrics
  namespace: demo-app
  labels: { app: hello-metrics }
spec:
  replicas: 2
  selector:
    matchLabels: { app: hello-metrics }
  template:
    metadata:
      labels: { app: hello-metrics }
    spec:
      containers:
        - name: hello-metrics
          image: yourdockerhub/hello-metrics:1.0
          ports:
            - containerPort: 8080
              name: http
---
apiVersion: v1
kind: Service
metadata:
  name: hello-metrics
  namespace: demo-app
  labels: { app: hello-metrics }
spec:
  selector: { app: hello-metrics }
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

**Step 4 — The `ServiceMonitor`** (Chapter 5 — this is the Custom Resource that tells the Prometheus Operator to start scraping this Service automatically, with zero manual edits to `prometheus.yml`):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hello-metrics
  namespace: demo-app
  labels:
    release: kube-prom-stack   # must match the Prometheus Operator's serviceMonitorSelector
spec:
  selector:
    matchLabels: { app: hello-metrics }
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

The `release: kube-prom-stack` label is the single most common gotcha here — the Prometheus Operator only picks up `ServiceMonitor`s matching the label selector configured on the `Prometheus` object itself (kube-prometheus-stack defaults to matching its own Helm release label). A `ServiceMonitor` without it is silently ignored, with no error anywhere.

**Step 5 — Load-test it** and confirm the graphs move:

```bash
kubectl create namespace demo-app
kubectl apply -f manifests/
kubectl port-forward -n demo-app svc/hello-metrics 8080:8080 &
hey -z 60s -c 10 http://localhost:8080/   # or `ab`, or a simple bash while-loop with curl
```

**Step 6 — Confirm scraping in Prometheus's own UI first**, before touching Grafana — this isolates "is the metric being collected at all" from "is my dashboard query wrong":

```bash
kubectl port-forward -n monitoring svc/kube-prom-stack-kube-prome-prometheus 9090:9090 &
```

Open `http://localhost:9090/targets` and confirm `demo-app/hello-metrics` shows `UP`. Then open `/graph` and run `hello_requests_total` — you should see a rising counter per Pod.

**Step 7 — Build the Grafana dashboard.** Log in (`kubectl port-forward -n monitoring svc/kube-prom-stack-grafana 3000:80`, user `admin` / `admin123`) and create three panels:

```promql
# Panel 1 — Request rate (requests/sec, per status code)
sum(rate(hello_requests_total[1m])) by (status)

# Panel 2 — p95 latency (requires aggregating by le BEFORE histogram_quantile — Chapter 4)
histogram_quantile(0.95, sum(rate(hello_request_duration_seconds_bucket[5m])) by (le))

# Panel 3 — In-flight requests gauge (no rate() — gauges are read directly)
hello_inflight_requests
```

**Success criteria:** All three panels render without "No data"; `/targets` in Prometheus shows the target `UP`; running the load test visibly moves the request-rate and latency panels in real time, and the in-flight gauge oscillates rather than sitting flat at zero.

---

## Project 2 — Alerting and SLOs (Intermediate)

**Goal:** Turn Project 1's app into a service with a real reliability target: define an SLO, calculate its error budget, write a multi-window burn-rate alert (Chapter 12), and route it to Slack through Alertmanager with sensible grouping and inhibition (Chapter 7).

**Step 1 — Define the SLO.** For a homework-scale service: **99% of requests complete in under 300ms, measured over a rolling 7 days.** This becomes the SLI expression — the fraction of "good" (fast enough) requests:

```promql
# SLI: fraction of requests under 300ms over the last 7 days
sum(rate(hello_request_duration_seconds_bucket{le="0.3"}[7d]))
/
sum(rate(hello_request_duration_seconds_count[7d]))
```

**Step 2 — Calculate the error budget** (Chapter 12). A 99% SLO over 7 days (604,800 seconds) allows:

```
Error budget = (1 - 0.99) × 604,800s = 6,048 seconds of "bad" time per week
             ≈ 100.8 minutes/week the service is allowed to breach the 300ms target
```

That number is the whole point of an SLO: instead of "latency looks a bit high, should we page someone," the team now has a concrete, pre-agreed budget to spend.

**Step 3 — Write a multi-window, multi-burn-rate alert** (Chapter 12's pattern — a single long-window alert is either too slow to catch fast burns or too noisy on slow ones, so production SLO alerting always pairs a fast, short window with a slow, long window that must **both** agree before paging):

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: hello-metrics-slo-burn-rate
  namespace: demo-app
  labels:
    release: kube-prom-stack
spec:
  groups:
    - name: hello-metrics-slo
      rules:
        - record: hello:sli_error_ratio_5m
          expr: |
            1 - (
              sum(rate(hello_request_duration_seconds_bucket{le="0.3"}[5m]))
              /
              sum(rate(hello_request_duration_seconds_count[5m]))
            )
        - record: hello:sli_error_ratio_1h
          expr: |
            1 - (
              sum(rate(hello_request_duration_seconds_bucket{le="0.3"}[1h]))
              /
              sum(rate(hello_request_duration_seconds_count[1h]))
            )
        - alert: HelloMetricsSLOFastBurn
          expr: |
            hello:sli_error_ratio_5m > (14.4 * 0.01)
            and
            hello:sli_error_ratio_1h > (14.4 * 0.01)
          for: 2m
          labels:
            severity: critical
            team: demo-app
          annotations:
            summary: "hello-metrics burning error budget 14.4x too fast"
            description: "At this rate the full 7-day error budget is exhausted in under a day. Check for a deploy, a downstream dependency, or a resource limit."
        - alert: HelloMetricsSLOSlowBurn
          expr: |
            hello:sli_error_ratio_1h > (3 * 0.01)
            and
            hello:sli_error_ratio_1h offset 6h > (3 * 0.01)
          for: 15m
          labels:
            severity: warning
            team: demo-app
          annotations:
            summary: "hello-metrics burning error budget 3x too fast (sustained)"
            description: "Not urgent enough to page immediately, but left unaddressed this exhausts the weekly error budget in about 2 days."
```

**Step 4 — Route the alert to Slack through Alertmanager**, grouped so a burst of related firings arrives as one message, not one-per-Pod (Chapter 7):

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prom-stack-kube-prome-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
    route:
      receiver: slack-demo-app
      group_by: ["alertname", "team"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 3h
      routes:
        - matchers: [severity="critical"]
          receiver: slack-demo-app-critical
          repeat_interval: 30m
      inhibit_rules:
        # A firing fast-burn already tells you everything the slow-burn would —
        # suppress the warning so on-call isn't paged twice for the same underlying issue
        - source_matchers: [alertname="HelloMetricsSLOFastBurn"]
          target_matchers: [alertname="HelloMetricsSLOSlowBurn"]
          equal: ["team"]
    receivers:
      - name: slack-demo-app
        slack_configs:
          - api_url: "https://hooks.slack.com/services/T000/B000/XXXXXXXX"
            channel: "#demo-app-alerts"
            title: "{{ .CommonAnnotations.summary }}"
            text: "{{ .CommonAnnotations.description }}"
      - name: slack-demo-app-critical
        slack_configs:
          - api_url: "https://hooks.slack.com/services/T000/B000/XXXXXXXX"
            channel: "#demo-app-alerts"
            title: ":rotating_light: {{ .CommonAnnotations.summary }}"
            text: "{{ .CommonAnnotations.description }}"
type: Opaque
```

**Step 5 — Degrade the app on purpose and prove it fires.** Add a `SLOW_MODE` environment variable to `app.py` that sleeps for 1–2 seconds instead of 0.05–0.4s, redeploy, and re-run the load test:

```bash
kubectl set env deployment/hello-metrics -n demo-app SLOW_MODE=true
hey -z 5m -c 10 http://localhost:8080/
```

Watch `hello:sli_error_ratio_5m` climb past the 14.4% threshold in Prometheus's `/graph`, then watch `HelloMetricsSLOFastBurn` transition `pending` → `firing` in the Prometheus UI's Alerts tab, and confirm the Slack message arrives within `group_wait` + a few seconds.

**Success criteria:** Injecting artificial latency causes `HelloMetricsSLOFastBurn` to fire within roughly 2–3 minutes; the Slack notification appears in the configured channel with a readable summary; if both the fast-burn and slow-burn conditions are true simultaneously, only the critical fast-burn message arrives thanks to the inhibition rule, rather than two separate noisy pages for the same root cause.

---

## Project 3 — Full Three-Pillars Observability (Advanced)

**Goal:** Extend the same app with centralized logging (Loki + Promtail, Chapter 10) and distributed tracing (OpenTelemetry + Tempo, Chapter 11), make sure logs carry the trace ID, and build one Grafana view that lets you pivot from a metrics anomaly to the exact trace to the exact log lines — Chapter 11's correlated-pillars workflow made real instead of theoretical.

**Step 1 — Install Loki and Promtail:**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

Promtail auto-discovers every Pod via the Kubernetes API and attaches `namespace`, `pod`, `container`, and label metadata to each log line automatically (Chapter 10) — no manual configuration needed for basic collection.

**Step 2 — Add structured logging with the trace ID embedded**, and instrument the app with OpenTelemetry so every request gets a trace ID that's threaded straight into the log line:

```python
# app.py additions
import logging, json
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="tempo.monitoring.svc.cluster.local:4317", insecure=True))
)

logger = logging.getLogger("hello-metrics")

def log_json(level, msg, **fields):
    span = trace.get_current_span()
    ctx = span.get_span_context()
    record = {
        "level": level,
        "msg": msg,
        "trace_id": format(ctx.trace_id, "032x"),
        "span_id": format(ctx.span_id, "016x"),
        **fields,
    }
    print(json.dumps(record))   # stdout — Promtail/kubelet picks this up like any container log

@app.route("/")
def hello():
    with tracer.start_as_current_span("handle-hello"):
        INFLIGHT_REQUESTS.inc()
        start = time.time()
        time.sleep(random.uniform(0.05, 0.4))
        duration = time.time() - start
        REQUEST_LATENCY.observe(duration)
        REQUEST_COUNT.labels(method="GET", path="/", status="200").inc()
        INFLIGHT_REQUESTS.dec()
        log_json("info", "request handled", duration_ms=round(duration * 1000, 1))
        return "hello\n"
```

Emitting `trace_id` as a structured JSON field (rather than free text) is what makes the log-to-trace pivot in Step 4 possible — Grafana's Loki-to-Tempo linking is just a regex/label extraction on that field.

**Step 3 — Install Tempo** for trace storage:

```bash
helm install tempo grafana/tempo --namespace monitoring
```

Confirm the app's Deployment points its OTLP exporter at `tempo.monitoring.svc.cluster.local:4317` (already set above), and confirm the `tempo` and `loki` data sources are added in Grafana (`Connections → Data sources`), each pointing at their respective in-cluster Services.

**Step 4 — Wire the Loki→Tempo derived field**, so a `trace_id` string inside a log line becomes a clickable link straight into the matching trace (Grafana → Loki data source settings → Derived fields):

```yaml
# Loki data source derived field config (via Grafana provisioning ConfigMap)
derivedFields:
  - datasourceUid: tempo
    matcherRegex: '"trace_id":"(\w+)"'
    name: TraceID
    url: "$${__value.raw}"
```

**Step 5 — Build the correlated dashboard.** Extend Project 1's Grafana dashboard with two new panels:

- A **Logs panel** (Loki data source) querying `{namespace="demo-app", container="hello-metrics"}` with a LogQL filter, e.g. `{namespace="demo-app"} | json | duration_ms > 300` to surface only slow requests.
- A **Traces panel** or trace-search link (Tempo data source) that a user reaches either directly from a slow log line's `TraceID` derived-field link, or by searching Tempo for spans matching `handle-hello` with a duration above 300ms.

**Step 6 — Prove the correlated workflow end-to-end.** Re-enable `SLOW_MODE`, run the load test again, and walk through:

1. Notice the p95 latency panel spike in Grafana (metrics).
2. Click into the same time window in the logs panel and find a slow request's log line.
3. Click the `TraceID` derived-field link on that log line — Grafana jumps straight to that request's trace in Tempo.
4. From the trace, confirm the span duration matches the log's `duration_ms`, closing the loop between all three pillars for one specific request.

**Success criteria:** A single Grafana dashboard displays a metrics panel, a Loki logs panel, and a Tempo trace view together; clicking a slow log line's trace ID link opens the exact matching trace; the trace's span duration and the log's `duration_ms` field agree, proving genuine correlation rather than three panels that merely happen to share a time range.

---

## Project 4 — Multi-Team Observability Platform (Production-Grade Capstone)

**Goal:** Design a shared observability platform for multiple teams on one cluster — tying back to Advanced Kubernetes Chapter 6's multi-tenancy model — where each team gets its own `ServiceMonitor`s and dashboards managed as code in their own GitOps repo, cardinality is enforced cluster-wide via relabeling (Chapter 13), each team's SLO burn-rate alerts route to their own Slack channel instead of one shared noisy one, and there's a documented runbook for the "Prometheus itself is unhealthy" meta-monitoring case (Chapter 15). This is explicitly framed as a resume/GitHub-portfolio piece demonstrating platform/SRE readiness — every manifest below is real and reusable against an actual cluster.

**Architecture:**

```
                     ┌───────────────────────────────┐
                     │  namespace: monitoring          │
                     │  kube-prometheus-stack           │
                     │  + Loki + Promtail                │
                     │  + Tempo                            │
                     │  cardinality-limiting relabel rules  │
                     │  meta-monitoring (watchdog + 2nd     │
                     │  Prometheus scraping the 1st)         │
                     └───────────────┬───────────────────┘
                                     │ discovers via label selectors
        ┌────────────────────────────┼────────────────────────────┐
        ▼                             ▼                             ▼
 team-payments namespace      team-checkout namespace      team-search namespace
 ├── ServiceMonitor (GitOps)  ├── ServiceMonitor (GitOps)  ├── ServiceMonitor (GitOps)
 ├── Grafana dashboard        ├── Grafana dashboard        ├── Grafana dashboard
 │   (ConfigMap, GitOps)      │   (ConfigMap, GitOps)      │   (ConfigMap, GitOps)
 ├── PrometheusRule (SLOs)    ├── PrometheusRule (SLOs)    ├── PrometheusRule (SLOs)
 └── Alertmanager route       └── Alertmanager route       └── Alertmanager route
     → #payments-alerts            → #checkout-alerts           → #search-alerts
```

**Step 1 — The `monitoring` namespace as a dedicated, platform-owned tenant** (Advanced Kubernetes Chapter 6 pattern — teams cannot write into this namespace; they only produce metrics/logs/traces that its stack consumes):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: baseline
```

**Step 2 — Per-team dashboards and `ServiceMonitor`s provisioned as code**, living in each team's *own* GitOps repo (Advanced Kubernetes Chapter 8), not the platform team's repo — this is the key design decision: the platform owns the stack, teams own their own observability config, exactly mirroring how they already own their own application manifests.

```yaml
# team-payments-gitops-repo/observability/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payments-api
  namespace: team-payments
  labels:
    release: kube-prom-stack   # required for the shared Operator to discover it
spec:
  selector:
    matchLabels: { app: payments-api }
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
      metricRelabelings:
        # Cardinality enforcement (Chapter 13) — drop any label carrying a raw
        # customer/order ID before it ever reaches Prometheus's TSDB. This is
        # enforced per-ServiceMonitor so it can't be "forgotten" at write time.
        - action: labeldrop
          regex: "(customer_id|order_id|raw_user_agent)"
---
# team-payments-gitops-repo/observability/dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"   # Grafana's sidecar auto-loads any ConfigMap with this label
data:
  payments-dashboard.json: |
    { "title": "Payments API", "panels": [ "...dashboard JSON committed by the team..." ] }
```

Because the dashboard ConfigMap is labeled `grafana_dashboard: "1"`, Grafana's sidecar container (enabled via `grafana.sidecar.dashboards.enabled=true` in the kube-prometheus-stack Helm values) picks it up automatically from any namespace — teams ship a dashboard by merging a PR to their own repo, with no platform-team ticket required.

**Step 3 — Cluster-wide cardinality limits**, enforced centrally in the shared Prometheus itself as a backstop behind the per-team `metricRelabelings` above (Chapter 13 — defense in depth, since a team can forget a relabel rule but shouldn't be able to take down shared Prometheus):

```yaml
# values.yaml (kube-prometheus-stack Helm values, platform-owned)
prometheus:
  prometheusSpec:
    enforcedLabelLimit: 20
    enforcedLabelNameLengthLimit: 64
    enforcedLabelValueLengthLimit: 256
    enforcedSampleLimit: 200000       # per-target sample ceiling; a runaway target gets dropped, not the whole TSDB
    retention: 15d
    retentionSize: 40GB
```

**Step 4 — A documented SLO catalog**, one entry per team's service, committed to the platform repo so SLOs are discoverable rather than scattered across tribal knowledge:

```yaml
# platform-repo/slo-catalog.yaml
services:
  - name: payments-api
    team: payments
    slo: "99.9% of requests under 200ms, rolling 30d"
    alert_channel: "#payments-alerts"
  - name: checkout-api
    team: checkout
    slo: "99.5% of requests succeed (non-5xx), rolling 7d"
    alert_channel: "#checkout-alerts"
  - name: search-api
    team: search
    slo: "99% of requests under 500ms, rolling 7d"
    alert_channel: "#search-alerts"
```

**Step 5 — Per-team Alertmanager routing**, so a burn-rate alert for payments never lands in checkout's channel (Chapter 7 — this is the single biggest lever against org-wide alert fatigue: nobody mutes a channel that's *only* their own team's real problems):

```yaml
route:
  receiver: fallback-platform-channel
  group_by: ["alertname", "team"]
  routes:
    - matchers: [team="payments"]
      receiver: slack-payments
      continue: false
    - matchers: [team="checkout"]
      receiver: slack-checkout
      continue: false
    - matchers: [team="search"]
      receiver: slack-search
      continue: false
receivers:
  - name: slack-payments
    slack_configs: [{ channel: "#payments-alerts", api_url: "https://hooks.slack.com/services/..." }]
  - name: slack-checkout
    slack_configs: [{ channel: "#checkout-alerts", api_url: "https://hooks.slack.com/services/..." }]
  - name: slack-search
    slack_configs: [{ channel: "#search-alerts", api_url: "https://hooks.slack.com/services/..." }]
  - name: fallback-platform-channel
    slack_configs: [{ channel: "#platform-observability-fallback", api_url: "https://hooks.slack.com/services/..." }]
```

**Step 6 — Meta-monitoring: what watches Prometheus itself** (Chapter 15's "who monitors the monitor" scenario). A second, minimal Prometheus (or Grafana Cloud's free tier, or a simple external blackbox check) scrapes the primary Prometheus's own `/metrics` and `up` targets, plus the built-in `Watchdog` alert pattern that should always be firing:

```yaml
# meta-monitoring/watchdog-rule.yaml — always-firing canary; if this ever stops
# firing, the alerting pipeline itself is broken (not the services it watches)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: watchdog
  namespace: monitoring
  labels: { release: kube-prom-stack }
spec:
  groups:
    - name: watchdog
      rules:
        - alert: Watchdog
          expr: vector(1)
          labels: { severity: none }
          annotations:
            summary: "Alerting pipeline heartbeat — this should always be firing."
```

Route `Watchdog` to a dead-man's-switch service (e.g., a scheduled check that expects a heartbeat and pages if one is missed) rather than Slack — a Slack message is useless if Slack routing itself is what broke.

**Step 7 — The runbook for "Prometheus itself is unhealthy"** (Chapter 15), committed to the platform repo:

```
## Runbook: Prometheus is down, OOMKilled, or its TSDB won't start

1. Confirm scope first — is this the shared platform Prometheus, or a
   per-team recording-rule/query problem being misreported as "Prometheus is down"?
   kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus

2. Check for OOMKill, the most common root cause (usually a cardinality
   spike from an unrelabeled team metric, per Chapter 13):
   kubectl describe pod <prometheus-pod> -n monitoring | grep -A3 "Last State"
   kubectl top pod <prometheus-pod> -n monitoring

3. If OOMKilled, identify which ServiceMonitor's target is responsible
   BEFORE just bumping memory limits (bumping limits without finding the
   cause just delays the next crash):
   # once Prometheus is back up even briefly:
   topk(10, count by (job)({__name__=~".+"}))

4. Confirm the Watchdog alert's absence in Alertmanager as independent
   confirmation the pipeline (not just the target service) is impacted:
   curl -s http://alertmanager.monitoring.svc:9093/api/v2/alerts | \
     jq '.[] | select(.labels.alertname=="Watchdog")'

5. If TSDB corruption is suspected (crash-loop on startup with WAL errors),
   do NOT delete the PVC as a first move — first attempt:
   kubectl exec -it <prometheus-pod> -n monitoring -- \
     promtool tsdb repair-corrupted /prometheus
   Only wipe and restart clean as a last resort, since 15 days of history
   (per the retention setting) is unrecoverable once gone.

6. Post-incident: add the specific offending label combination to the
   metricRelabelings labeldrop list for that team's ServiceMonitor, and
   lower enforcedSampleLimit if the same team's targets keep tripping it.
```

**Implementation steps:**

1. Install kube-prometheus-stack, Loki/Promtail, and Tempo into the shared `monitoring` namespace with the cardinality-limiting Helm values from Step 3.
2. Create three team namespaces (`team-payments`, `team-checkout`, `team-search`), each with a placeholder app exposing basic metrics.
3. Set up three separate small Git repos (or three directories simulating separate repos) representing each team's own GitOps-managed observability config, each containing a `ServiceMonitor`, a dashboard ConfigMap, and a `PrometheusRule` for their own SLO.
4. Apply the SLO catalog and the per-team Alertmanager routing config to the shared Alertmanager.
5. Deploy the `Watchdog` rule and confirm it's continuously firing in Alertmanager.
6. Deliberately push a metric with high-cardinality labels from one team's app without a `labeldrop` rule, and confirm the `enforcedLabelLimit`/`enforcedSampleLimit` backstop prevents it from taking down shared Prometheus for the other two teams.
7. Trigger one team's SLO alert and confirm it lands only in that team's Slack channel, not the others'.
8. Write the runbook, commit the entire platform repo (Helm values, SLO catalog, routing config, runbook) to a public GitHub repository with a README describing the architecture — this is the artifact for a resume or portfolio.

**Success criteria:** Each team can ship a new dashboard or alert rule via a PR to their own repo with no platform-team ticket; a cardinality spike from one team's misbehaving metric is contained by the enforced limits and does not take down monitoring for other teams; each team's burn-rate alerts route only to that team's own Slack channel; the `Watchdog` heartbeat alert is continuously visible in Alertmanager as proof the pipeline itself is healthy; the repo is public, documented, and includes the meta-monitoring runbook — explicitly a "I built a real observability platform" project demonstrating platform/SRE readiness.

---

## Summary

These four projects build on each other deliberately: Project 1 proves you can get an app's metrics into Prometheus and onto a dashboard at all, Project 2 turns those metrics into an actual reliability commitment with an alert that pages a human only when it matters, Project 3 completes the three pillars by adding logs and traces and proving they correlate for a single request, and Project 4 stitches every earlier chapter's skill into a shared platform multiple teams can safely use at once — worth putting on a resume.

| Project | Level | Approx Time | Key Skills |
|---------|-------|-------------|------------|
| 1 — Metrics and Dashboards | Beginner | 3–4 hours | Prometheus client library, `ServiceMonitor`, `histogram_quantile`, Grafana panels |
| 2 — Alerting and SLOs | Intermediate | 5–7 hours | SLI/SLO definition, error budget math, multi-window burn-rate alerts, Alertmanager grouping/inhibition |
| 3 — Full Three-Pillars Observability | Advanced | 8–10 hours | Loki/Promtail, OpenTelemetry tracing, Tempo, trace-ID log correlation, unified Grafana view |
| 4 — Multi-Team Observability Platform | Advanced/Capstone | 12–16 hours | Shared platform design, GitOps-managed per-team dashboards/alerts, cardinality enforcement, per-team routing, meta-monitoring runbook |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-common-mistakes.md">← Previous: Common Mistakes and Pitfalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./17-interview-preparation.md">Next: Interview Preparation →</a>
</div>
