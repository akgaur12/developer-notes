# Chapter 17 — Interview Preparation

**Learning Objectives**

By the end of this chapter you will be able to confidently answer SRE/observability-engineer-level questions on metrics, Prometheus internals, logging architecture, distributed tracing, and SLO-based alerting — spanning foundational concepts, architecture/internals, incident scenarios, and system design — and articulate your own observability experience in a structured way.

---

## 17.1 Foundational Questions

**Q: What's the difference between monitoring and observability?**
> Monitoring is watching a predefined set of signals for known failure modes — dashboards and alerts built around questions you already knew to ask, like "is CPU above 90%" or "is the health check returning 200." Observability is the broader property of a system that lets you answer questions you *didn't* know to ask in advance, by exploring high-cardinality, high-dimensionality telemetry (metrics, logs, traces together) after the fact. A well-monitored system tells you *that* something is wrong; an observable system lets you figure out *why*, for a failure mode nobody wrote a dashboard for ahead of time. In practice the two aren't opposed — Chapter 1 frames monitoring as a subset of what an observable system gives you, not a replacement for it.

**Q: Explain the difference between a counter, a gauge, and a histogram.**
> A counter is a cumulative value that only ever increases (or resets to zero on restart) — request counts, error counts, bytes sent. Because it only goes up, you almost never graph it raw; you graph `rate()` of it to see requests/sec. A gauge is a value that goes up and down freely — current memory usage, in-flight requests, queue depth — and you graph it directly, since there's no "rate of a gauge" that makes general sense. A histogram samples observations (typically latencies or sizes) into configurable buckets and exposes both a running count-per-bucket and a `_sum`/`_count`, which lets you calculate quantiles (p50, p95, p99) after the fact via `histogram_quantile()` — critically, without the client ever pre-deciding which single percentile you'll want to ask about later.

**Q: Why does Prometheus use a pull model instead of push?**
> With a pull model, Prometheus itself controls scrape timing and can immediately tell the difference between "the target is down" (scrape fails) and "the target has nothing to report" — that `up` metric is a free, reliable liveness signal you get for nothing. It also means targets don't need to know anything about where Prometheus lives or manage their own delivery reliability/retry logic, and Prometheus can be pointed at a completely new target just by adding service discovery config, with no code change or redeploy on the target's side. The tradeoff is that pull assumes Prometheus can reach the target over the network, which is awkward for short-lived batch jobs that finish before a scrape would ever happen — which is exactly why the Pushgateway exists as a deliberate, narrow exception rather than the default model.

**Q: What is cardinality and why is it dangerous?**
> Cardinality is the number of unique time series a metric produces, which is the product of every label's number of distinct values. A metric with a `status` label (a handful of values) and a `user_id` label (potentially millions of values) doesn't multiply those two numbers by coincidence — Prometheus's TSDB allocates real memory and disk for every unique combination of label values as its own independent time series, so a single careless high-cardinality label can turn one metric into millions of series. This is dangerous because Prometheus's memory usage scales with the number of active series, not the number of metric *names* — a cardinality explosion looks like "Prometheus randomly OOMKilled itself" with no warning, which is why Chapter 13's cardinality-management practices (never label with user IDs, request IDs, IPs, or raw error messages) exist as a hard rule, not a style preference.

**Q: What's the difference between an SLI, an SLO, and an SLA?**
> An SLI (Service Level Indicator) is the actual measured metric — e.g., "the percentage of requests that complete in under 300ms." An SLO (Service Level Objective) is the internal target for that SLI — e.g., "99% of requests under 300ms, measured over a rolling 7 days" — the number a team designs its alerting and prioritization around. An SLA (Service Level Agreement) is an SLO with a contractual, usually financial, consequence attached if it's missed, typically made externally to a customer. The practical relationship is that SLAs are usually set looser than the internal SLO specifically so the team has room to notice and fix a problem via its error budget before ever breaching the customer-facing contractual promise.

**Q: What is an error budget and what is it used for?**
> An error budget is simply `1 - SLO`, converted into an amount of allowed "bad" time or bad requests over the SLO's measurement window — e.g., a 99.9% SLO over 30 days allows about 43 minutes of budget. It's used as a shared, numeric decision-making tool between engineering and product: while budget remains, the team can ship features and take reasonable risks; once the budget is exhausted, the agreed-upon response is to freeze risky changes and prioritize reliability work instead, removing the recurring, subjective argument over whether "reliability" or "velocity" should win this week. It also directly drives alerting design — burn-rate alerts (Chapter 12) page based on how fast the budget is being consumed, not on a static threshold.

**Q: Why would you choose Loki over Elasticsearch, or vice versa?**
> Loki indexes only a small set of labels (much like Prometheus indexes metric labels) and stores the actual log lines compressed and unindexed, which makes it dramatically cheaper to run and simpler to operate at scale, at the cost of full-text search being slower since it has to grep across compressed chunks rather than hit an inverted index. Elasticsearch (the "E" in ELK) full-text-indexes every field by default, which makes ad hoc searching across arbitrary content in log bodies fast, at the cost of significantly higher storage, memory, and operational overhead — you're paying to index data you may never search. The practical heuristic from Chapter 10: choose Loki when your query pattern is "logs from this service, in this time range, filtered to lines matching X" (the vast majority of real debugging), and choose Elasticsearch when you genuinely need rich full-text search or complex cross-field analytics across your logs as a first-class requirement, not an occasional one.

**Q: What is a trace and how does it differ from a log?**
> A trace represents the full lifecycle of a single request as it moves through a distributed system, made up of spans — one span per unit of work (an HTTP call, a DB query, a downstream service call) — each carrying a start time, duration, and parent-child relationship to other spans, all sharing a common trace ID. A log is a discrete, timestamped event emitted by one process at one point in time, with no inherent structural relationship to other logs unless you add one yourself (like a trace ID field). The core difference is that a trace is inherently relational and shows causality and timing *across service boundaries* for one request, while a log is a point-in-time record from inside one service — which is exactly why correlating them (embedding the trace ID in every log line, per Chapter 11) is so valuable: the trace tells you *where* the time went across services, the log tells you *what actually happened* inside the slow one.

**Q: What's the difference between `rate()` and `irate()` in PromQL?**
> Both calculate the per-second rate of increase of a counter over a time range, but `rate()` uses all data points in the range and smooths across them (extrapolating at the range's edges), making it appropriate for alerting rules and dashboards where you want a stable trend. `irate()` uses only the last two data points in the range, making it much more sensitive to very recent, sudden spikes — useful for fast-moving dashboards showing "what's happening right now" but far too noisy and prone to false positives to use in an alerting rule. The practical rule of thumb from Chapter 4: `rate()` for anything that pages a human or drives a decision, `irate()` only for high-resolution, human-eyeballed dashboards.

**Q: Why is structured logging preferred over plain-text logs?**
> A plain-text log line like `ERROR user login failed for bob at 10:32` requires a regex or string parse to extract `user=bob` reliably, and that parser breaks the moment the message wording changes even slightly. A structured log line — JSON, e.g. `{"level":"error","event":"login_failed","user":"bob","ts":"..."}` — exposes every field as queryable data from the start, with no brittle parsing step, which is what lets LogQL/Elasticsearch queries filter and aggregate on fields directly (`| json | user="bob"`) rather than grepping for substrings. It also makes correlation trivial — adding a `trace_id` field is one line of code, versus trying to reliably regex a trace ID out of free text later.

---

## 17.2 Architecture and Internals Questions

**Q: Explain how `histogram_quantile` actually works, and why you must aggregate `by (le)` first.**
> A Prometheus histogram metric exposes multiple time series, one per bucket boundary (`le`, "less than or equal"), each a cumulative count of observations at or below that boundary, plus `_sum` and `_count`. `histogram_quantile(0.95, ...)` takes those cumulative bucket counts and linearly interpolates within the bucket that crosses the 95th-percentile rank to estimate that quantile's value — it is an approximation, not an exact percentile, and its accuracy is bounded by how many buckets you defined and how tightly they're spaced around your actual latency distribution. The `by (le)` aggregation is mandatory when you have multiple label dimensions (e.g., per-Pod histograms) because `histogram_quantile` needs one clean, ordered set of cumulative buckets to interpolate across — if you sum across Pods without also preserving `le` as a grouping key, you either get a nonsensical result or a query error, since the function has no way to know it's looking at buckets from different, incompatible series.

**Q: Explain Alertmanager's grouping, routing, and inhibition pipeline, in order.**
> An alert firing in Prometheus is first sent to Alertmanager, which evaluates the `route` tree top-down, matching the alert's labels against each route's matchers to decide which receiver(s) and grouping/timing config apply — child routes can override the parent's `group_by`, `group_wait`, `group_interval`, and `repeat_interval`. Alerts matching the same `group_by` label set within the same route are bundled into a single notification rather than firing individually — `group_wait` delays the first notification briefly to catch near-simultaneous related alerts, `group_interval` controls how soon a new alert can join an already-notified group, and `repeat_interval` controls how often an unresolved alert re-notifies. Only after grouping and routing decide *what would be sent* does the inhibition stage run, suppressing a target alert entirely if a matching source alert (per `inhibit_rules`, matched via `equal` labels) is already firing — this is the mechanism that stops a root-cause alert and all its downstream symptom alerts from paging the same person three separate times for one incident.

**Q: Explain how Promtail attaches Kubernetes labels automatically.**
> Promtail runs as a DaemonSet with access to the Kubernetes API and uses the same `kubernetes_sd_config` service-discovery mechanism Prometheus itself uses — it watches the API server for Pod objects on its own node, and for every container's log file it discovers (via the container runtime's log path convention on-disk), it attaches labels like `namespace`, `pod`, `container`, and any Kubernetes labels already present on the Pod, entirely from Kubernetes metadata rather than by parsing the log content itself. This is why Loki queries can filter by `{namespace="demo-app", container="hello-metrics"}` with zero manual configuration per-application — the labeling happens at the collection layer based on where the log physically came from, and it's also why Chapter 10 warns against relabeling high-cardinality Kubernetes labels (like a Pod name that changes every deploy) into Loki's indexed label set, since Loki's cost model penalizes high-cardinality *labels* specifically, unlike its unindexed log body.

**Q: Explain context propagation in distributed tracing, and what breaks it.**
> Context propagation is how a trace ID (and its current span ID, acting as the new span's parent) survives a request as it crosses a process or network boundary — typically carried in HTTP headers (`traceparent` in the W3C Trace Context standard) that the calling service's instrumentation library injects and the receiving service's instrumentation library extracts to start its own child span under the same trace. It breaks most commonly at boundaries the instrumentation library doesn't automatically understand: a message queue (Kafka, SQS) where the trace context has to be manually placed into and read back out of message attributes/headers, a fire-and-forget background job kicked off without the header ever being passed along, or a manually-constructed HTTP client call that bypasses the auto-instrumented client library entirely. The visible symptom is a trace with a suspicious gap — a parent span ends and a seemingly-unrelated new trace starts elsewhere with no connecting span — which is precisely Chapter 11's and Chapter 15's "trace missing spans across an async boundary" failure mode.

**Q: Explain why Prometheus's local storage isn't meant for long-term retention, and what solves that.**
> Prometheus's local TSDB is designed around fast local disk access and a retention window measured in days-to-a-few-weeks; it has no built-in replication, clustering, or long-term compaction strategy suited for storing months or years of data cost-effectively, and a single Prometheus instance is a single point of failure for its own history — losing the disk loses everything before the last backup. The standard solution is a remote-write-compatible long-term storage backend (Thanos, Cortex, Mimir, or a managed equivalent) that Prometheus streams samples into continuously via `remote_write`, giving you cheap object-storage-backed long-term retention, global query federation across many Prometheus instances, and deduplication — while local Prometheus keeps only a short operational window on fast local disk for the queries and alerting rules that actually need low latency.

**Q: What is the actual difference between a `summary` and a `histogram` metric type in Prometheus, given both can produce quantiles?**
> A `summary` calculates its configured quantiles (e.g., p95) client-side, at observation time, and exposes those pre-calculated quantile values directly — meaning they cannot be aggregated across instances after the fact (you cannot average or re-derive a p95 across ten Pods' individually-calculated p95s and get a meaningful number). A `histogram` exposes raw bucket counts and defers the quantile calculation to query time via `histogram_quantile()`, which means bucket counts *can* be summed across every Pod first and then one accurate cluster-wide quantile calculated from the combined buckets. This is why histograms are almost always the right default for anything that will be aggregated across replicas (which is nearly everything in Kubernetes), and summaries are reserved for the narrow case of needing an exact client-side quantile for a single process that will never need cross-instance aggregation.

**Q: How does the Prometheus Operator's `ServiceMonitor` actually result in Prometheus scraping a target?**
> The Prometheus Operator watches for `ServiceMonitor` (and `PodMonitor`) custom resources matching the label selector configured on the `Prometheus` CR itself, and for every match, resolves the `ServiceMonitor`'s `selector` against Services in the cluster to find the underlying Pods and their endpoints. It then generates the actual low-level `scrape_configs` section of Prometheus's configuration file, renders it into a Secret, and triggers Prometheus (via a sidecar config-reloader container) to reload that config without a restart. The entire value of this indirection is that application teams never write raw Prometheus scrape config or touch the central `prometheus.yml` at all — they just declare a `ServiceMonitor` next to their own app's manifests, and the Operator handles wiring it into the shared instance's actual configuration safely.

---

## 17.3 Scenario-Based Questions

**Scenario 1: "Prometheus's memory usage keeps growing until it crashes"**
```
1. Confirm it's actually a cardinality problem and not simply undersized
   memory for legitimate load, by checking total active series first:
   prometheus_tsdb_head_series
   # a healthy small/medium cluster is usually in the tens of thousands;
   # millions strongly suggests a cardinality explosion, not normal growth

2. Find which metric names contribute the most series:
   topk(10, count by (__name__)({__name__=~".+"}))

3. For the worst offender, find which LABEL is driving the explosion —
   this is almost always the real root cause, not the metric name itself:
   topk(10, count by (label_name)({__name__="the_offending_metric"}))
   # try this against each suspicious label (user_id, request_id, path, etc.)

4. Confirm the source — usually a ServiceMonitor/PodMonitor scraping an
   app that labels a metric with something unbounded (Chapter 13):
   kubectl get servicemonitor -A -o yaml | grep -B5 metricRelabelings

5. Apply an immediate metricRelabelings labeldrop rule on the offending
   ServiceMonitor to stop the bleeding without redeploying the app:
   metricRelabelings:
     - action: labeldrop
       regex: "user_id|request_id"

6. Set enforcedLabelLimit and enforcedSampleLimit on the shared Prometheus
   as a permanent backstop so no single team's mistake can repeat this,
   and fix the root cause in the app's instrumentation code itself —
   the relabel rule is a stopgap, not the real fix
```

**Scenario 2: "An alert fired but nobody knows what to do about it"**
```
1. This is an actionability gap, not a Prometheus problem — first confirm
   the alert even HAS a runbook link in its annotations:
   kubectl get prometheusrule <name> -n <ns> -o yaml | grep -A3 annotations

2. If there's no runbook_url annotation, that's the root cause found —
   every paging alert should carry one, added as a required field via a
   linter/policy check on PrometheusRule PRs going forward

3. If a runbook exists but wasn't followed, check whether it's actually
   findable in the alert — a runbook buried in a wiki nobody links from
   the Slack message is functionally the same as no runbook

4. Distinguish "no runbook" from "the alert itself is meaningless" — ask
   whether this alert has ever previously required a different action
   twice; if the response is always identical and mechanical, this may be
   a candidate for auto-remediation instead of a human page at all

5. Fix going forward: require every new alert's annotations to include
   `summary` (what's wrong), `description` (why it matters/impact), and
   `runbook_url` (exact steps) before it's allowed to page, per Chapter 14's
   "no alert without an owner and a runbook" practice
```

**Scenario 3: "A dashboard shows normal metrics but users are actually complaining"**
```
1. Suspect the SLI itself doesn't reflect real user experience before
   assuming users are wrong — the most common cause is measuring server-
   side latency/success while the user experiences client-side or
   edge/CDN-layer problems the server never sees:
   # check: does the metric measure from the load balancer, the app, or
   # is there a client-side RUM (real user monitoring) signal at all?

2. Check for averaging hiding a tail-latency or a segment-specific problem
   — an aggregate p50 or even p95 across ALL traffic can look perfectly
   healthy while one specific endpoint, region, or customer segment is badly
   broken:
   histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path))
   # break down BY the dimension likely to be hiding the problem

3. Check whether the "successful" status code the SLI counts as good is
   actually good from the user's perspective — a 200 response containing
   an error payload, or a slow-but-technically-200 response above the
   user's patience threshold, both pass a naive success-rate SLI

4. This is exactly the case for adding tracing and log correlation
   (Chapter 11, Project 3 of this course) — pull an actual affected user's
   session/request via a support ticket's approximate timestamp, and trace
   through it directly rather than trusting an aggregate metric to represent
   their individual experience

5. Fix going forward: redefine the SLI closer to genuine user experience
   (add client-side RUM, segment the SLI by endpoint/region/customer tier,
   or tighten the latency threshold considered "good") rather than just
   trusting that "the dashboard is green" settles the question
```

**Scenario 4: "A trace is missing spans for part of the request"**
```
1. Identify exactly where the trace gap occurs — which span is the last
   one before the gap, and what does that service do immediately after
   that span ends (this narrows the boundary being crossed):
   # look at the trace waterfall in Tempo/Jaeger UI for the last span
   # before the visible time gap, and its service name

2. Check whether that boundary is a message queue, background job, or
   fire-and-forget call — these are the classic places automatic
   instrumentation does NOT propagate context without explicit help
   (Chapter 11, Chapter 15):
   # Kafka/SQS: was the trace context explicitly written into message
   # headers/attributes on publish, and explicitly read back on consume?

3. Check whether the receiving side is even instrumented at all, versus
   context propagation actually failing — a service with zero
   instrumentation produces the identical symptom (a visible gap) to one
   with a genuine propagation bug:
   kubectl get pods -n <ns> -l app=<downstream-service> -o yaml | \
     grep -i otel   # confirm OTel SDK/auto-instrumentation is even present

4. Check for a manually-constructed HTTP/gRPC client bypassing the
   auto-instrumented client library — a raw socket call or an
   uninstrumented HTTP client won't inject the traceparent header even
   though "the same service" is instrumented everywhere else

5. Fix: manually propagate context across the identified boundary using
   the OTel SDK's inject/extract API at the queue producer/consumer edge,
   and add an instrumentation coverage check to code review for any new
   async or queue-based integration going forward
```

**Scenario 5: "You're getting paged constantly for the same noisy alert"**
```
1. Pull the alert's actual firing history first — confirm it's genuinely
   frequent and not a perception issue from one bad week:
   curl -s http://alertmanager:9093/api/v2/alerts?filter=alertname="X" | \
     jq '. | length'

2. Check whether the threshold is simply wrong for this service's normal
   variance — a static threshold set once during a low-traffic period is
   a classic Chapter 7 alert-fatigue root cause as traffic grows:
   # plot the underlying metric over 30 days; does it regularly cross the
   # threshold during normal, non-incident operation?

3. Check whether this should be a burn-rate/SLO-based alert instead of a
   static threshold (Chapter 12) — a static "latency > 300ms" alert fires
   on every brief blip, while a burn-rate alert only fires when the blip
   is sustained enough to actually threaten the error budget

4. Check `for:` duration — an alert with no `for:` clause (or too short
   one) fires on single-scrape noise instead of a sustained condition:
   kubectl get prometheusrule <name> -o yaml | grep -A2 "alert: X"

5. Check grouping/repeat_interval in Alertmanager — a correctly-firing
   alert can still feel like constant paging if repeat_interval is too
   short relative to how long the on-call engineer needs to actually work
   the issue before being re-notified

6. Fix: retune the threshold from real historical data, convert to a
   burn-rate alert if this is SLO-relevant, add/lengthen the `for:` clause,
   and if the alert has fired repeatedly with the same non-actionable
   resolution every time, seriously consider deleting it rather than
   retuning it yet again — a repeatedly-ignored alert trains people to
   ignore the next real one too
```

**Scenario 6: "You need to reduce logging costs by 60% without losing debugging capability"**
```
1. Start by measuring where the volume/cost actually comes from — it is
   very rarely evenly distributed:
   sum by (namespace) (rate({job=~".+"}[1h]))   # LogQL, or the ELK equivalent
   # a small number of extremely chatty services usually dominate total volume

2. Check for accidental DEBUG-level logging left on in production, or
   duplicate logging (an app logging AND its sidecar/proxy logging the
   same event) — this is often the single biggest free win:
   kubectl get deployment <chatty-app> -o yaml | grep -i log_level

3. Tune retention/index lifecycle rather than cutting collection first —
   most debugging happens within hours of an incident, not months later;
   move older data to cheaper storage tiers or delete it sooner:
   # Loki: reduce retention_period in the compactor config for less-critical
   # namespaces; ELK: tighten ILM (Index Lifecycle Management) hot→warm→delete phases

4. Apply sampling to genuinely high-volume, low-value logs (e.g., successful
   health-check pings) while explicitly NOT sampling error-level logs —
   errors are exactly what you need fully intact during an incident:
   # sample INFO/DEBUG at 10%, keep WARN/ERROR at 100% — this is the standard
   # asymmetric sampling pattern for cost control without losing signal

5. Convert unstructured, verbose text logs to structured JSON with fewer,
   more meaningful fields — structured logs compress better AND let you
   drop redundant free-text framing ("Request received: ...") since the
   fields themselves already carry the same information

6. Validate the change actually preserved debugging capability before
   declaring success — replay a past real incident's investigation using
   only the post-change logs and confirm you could still have diagnosed it
```

---

## 17.4 System Design Questions

**"Design an observability stack for a 50-microservice Kubernetes platform."**
```
1. Metrics: one shared kube-prometheus-stack instance (or several federated
   by domain/team at that scale) with per-service ServiceMonitors, remote-
   written to a long-term store (Thanos/Mimir) for retention beyond the
   local TSDB's practical window, since 50 services generate meaningfully
   more series than a single-app deployment

2. Logging: Loki + Promtail as the default for cost reasons at this scale
   (Chapter 10) — Elasticsearch only if specific services genuinely need
   rich full-text search as a first-class product requirement, not as a
   blanket default for every service

3. Tracing: OpenTelemetry Collector deployed as a DaemonSet or sidecar
   pattern across all 50 services, exporting to Tempo, with a *consistent*
   context-propagation story enforced via a shared internal HTTP/gRPC
   client library so individual teams can't accidentally skip instrumentation

4. Correlation: enforce a shared structured-logging convention across all
   50 services (a common internal logging library) that always includes
   trace_id, so cross-pillar correlation (Chapter 11, Project 3) works
   uniformly rather than only for the services that happened to add it

5. Cardinality governance from day one, not retrofitted later: enforced
   label/sample limits on the shared Prometheus, mandatory metricRelabelings
   review for new ServiceMonitors, and a documented list of label names that
   are never allowed (user IDs, request IDs, raw paths with embedded IDs)

6. Ownership model: each of the 50 services' teams owns their own
   dashboards, alerts, and SLOs as code in their own repo (mirroring
   Project 4 of this course), while the platform team owns the shared
   infrastructure (Prometheus, Loki, Tempo, Grafana, Alertmanager) itself

7. Meta-monitoring: a Watchdog-style always-firing canary alert plus an
   independent, minimal secondary check on the observability stack's own
   health, so "the monitoring system is down" is itself detected
```

**"Design an SLO-based alerting strategy for a multi-team platform."**
```
1. Every customer-facing service gets exactly one or two SLOs defined in a
   central, discoverable catalog (a latency SLO and/or an availability
   SLO) — resist the temptation to define an SLO for everything measurable;
   too many SLOs dilutes what "on the error budget" even means

2. Every SLO is backed by a multi-window, multi-burn-rate alerting rule
   (Chapter 12, Project 2 of this course) rather than a single static
   threshold — fast+short window for urgent pages, slow+long window for
   ticket-level, non-paging warnings

3. Routing is per-team by default (Project 4's pattern) — a shared,
   catch-all "#alerts" channel across 50 services guarantees alert fatigue
   and diffusion of responsibility; each team owns their own channel and
   their own pager rotation

4. Error budget policy is written down and agreed to in advance, not
   improvised during an incident: what specifically happens when a
   service's budget is exhausted (feature freeze, mandatory reliability
   sprint, escalation to a director) — the SLO is only useful as a
   decision-making tool if the consequence of breaching it is pre-agreed

5. A shared platform-level view (a "fleet SLO dashboard") lets leadership
   see budget status across all 50 services at a glance, without needing
   50 individual dashboards, while each team's own dashboard shows their
   service's SLO in full depth

6. Regular SLO review cadence (monthly/quarterly) to retire SLOs that
   turned out not to matter, tighten ones that were set too loosely, and
   adjust ones where real user expectations shifted — an SLO set once and
   never revisited drifts out of sync with what users actually expect
```

**"Design a cost-effective logging pipeline for a high-volume, log-heavy workload."**
```
1. Default to Loki's label-based indexing model rather than full-text
   indexing everything (Chapter 10) — index only what you actually filter
   by (namespace, service, level), leave the log body itself unindexed
   but compressed and fully searchable via grep-style LogQL filters

2. Tier retention deliberately: recent data (say, 7–14 days) on fast,
   more expensive storage for active incident investigation; older data
   moved to cheap object storage (S3/GCS) with slower query performance,
   accepted because it's rarely queried; delete entirely past a compliance-
   driven maximum

3. Apply asymmetric sampling at the collection layer — sample high-volume,
   low-value log lines (health checks, successful routine polling) while
   never sampling WARN/ERROR/FATAL, so debugging capability for actual
   problems is fully preserved while routine noise volume drops sharply

4. Push structure to the edge: require services to emit structured JSON
   logs rather than free text, since structured logs compress better and
   let the pipeline extract only the fields actually needed downstream
   rather than shipping large repetitive text blobs

5. Batch and compress at the collector (Promtail/Fluent Bit/OTel Collector)
   before shipping over the network — shipping every line individually in
   real time is both slower and more expensive than reasonable batching
   with a small, acceptable delay

6. Measure cost per team/namespace and make it visible — teams without
   visibility into their own logging volume's cost have no incentive to
   fix a chatty DEBUG log left on by accident; a per-namespace cost
   dashboard turns this into self-service cleanup instead of a platform-
   team chase
```

---

## 17.5 Quick-Fire Questions

| Question | Answer |
|----------|--------|
| Metric type that only increases? | Counter |
| Metric type you graph directly, without `rate()`? | Gauge |
| Function used to calculate a percentile from a histogram? | `histogram_quantile()` |
| Why does Prometheus scrape rather than receive pushes? | Pull model — built-in liveness (`up`), simpler targets, no delivery-reliability burden on targets |
| CRD that tells the Prometheus Operator what to scrape? | `ServiceMonitor` (or `PodMonitor`) |
| What turns raw metrics into a reliability commitment? | An SLO (backed by an SLI and an error budget) |
| Alertmanager feature that suppresses a symptom alert when a root-cause alert fires? | Inhibition |
| Alertmanager feature that bundles related alerts into one notification? | Grouping (`group_by`) |
| Log aggregation tool that indexes labels, not full text? | Loki |
| Component that ships Kubernetes Pod logs to Loki? | Promtail (or Fluent Bit) |
| Standard for propagating trace context across services? | W3C Trace Context (`traceparent` header) |
| Backend commonly used for long-term Prometheus retention beyond local TSDB? | Thanos, Cortex, or Mimir (via `remote_write`) |
| Vendor-neutral instrumentation standard for traces/metrics/logs? | OpenTelemetry |
| Trace storage/visualization backends mentioned throughout this course? | Tempo and Jaeger |
| What single label mistake most commonly causes a cardinality explosion? | Labeling with an unbounded value like `user_id` or `request_id` |

---

## 17.6 "Walk Me Through Your Observability/SRE Experience"

STAR format example:

```
Situation: Our platform ran about a dozen services on Kubernetes with only
kubectl logs and ad hoc CPU/memory dashboards to go on. When users reported
slowness, the team's only real diagnostic step was guessing which service
to check first and tailing logs on individual Pods one at a time. A major
incident took over three hours to root-cause because nobody could see
across service boundaries, and the postmortem's single biggest finding was
"we had no way to see this coming or find it quickly."

Task: Build real observability into the platform — metrics, centralized
logs, and tracing — and define actual reliability targets so the team
could tell the difference between "annoying but within budget" and
"needs to be fixed right now," instead of treating every complaint as
equally urgent.

Action:
1. Deployed kube-prometheus-stack via the Prometheus Operator and
   instrumented each service with request-count, latency-histogram, and
   in-flight-gauge metrics, wired to auto-discovery via ServiceMonitors
   so no team had to hand-edit a central scrape config
2. Defined one SLO per customer-facing service (latency and/or
   availability), calculated each service's error budget, and replaced
   every static-threshold alert with a multi-window burn-rate alert,
   cutting the on-call team's total page volume by roughly 70% while
   catching real incidents faster than the old thresholds ever had
3. Rolled out Loki and Promtail for centralized, structured logging
   across every service, requiring a shared internal logging library so
   every log line included a trace ID automatically
4. Instrumented the platform's critical request paths with OpenTelemetry,
   exporting to Tempo, and built Grafana dashboards linking metrics
   panels directly to logs and traces for the same time window — turning
   a three-hour guess-and-check incident process into a five-minute
   metric-to-trace-to-log pivot
5. Set up per-team Alertmanager routing so each team's SLO alerts landed
   in their own Slack channel instead of one shared, ignored firehose
6. Documented a runbook for the observability stack's own failure modes
   (Prometheus OOMKill from cardinality, Alertmanager routing gaps) after
   a cardinality spike from one team's unbounded label briefly took down
   shared Prometheus for everyone

Result: Mean time to diagnosis for production incidents dropped from
hours to single-digit minutes for the incidents that occurred afterward,
directly attributable to the metrics-to-traces-to-logs correlation
workflow. Alert volume dropped sharply while genuine incident detection
got faster, because burn-rate alerts replaced noisy static thresholds.
The cardinality incident led to enforced per-team relabeling rules and
cluster-wide sample limits that prevented any repeat, and the SLO catalog
gave product and engineering a shared, numeric basis for prioritizing
reliability work instead of arguing about it every sprint.
```

**Self-Check Before Your Interview**

- Can you explain, without notes, why `histogram_quantile` requires aggregating `by (le)` and what goes wrong if you forget it?
- Can you calculate an error budget from a stated SLO and window, out loud, without a calculator?
- Can you walk through Alertmanager's grouping → routing → inhibition pipeline in the correct order and explain what each stage is for?
- Can you describe a real (or project-based) incident using the diagnostic flow from section 17.3, narrating your reasoning rather than jumping straight to the answer?
- Can you defend a specific logging-cost-reduction plan under a follow-up question like "how do you know you didn't just make debugging harder"?

No separate hands-on exercise for this chapter — working through the scenarios and system design questions above out loud, from memory, and defending your reasoning under follow-up questions, is the exercise.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./16-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./18-course-summary.md">Next: Course Summary →</a>
</div>
