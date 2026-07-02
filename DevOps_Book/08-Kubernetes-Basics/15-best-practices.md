# Chapter 15 — Best Practices

## Learning Objectives

By the end of this chapter, you will be able to:

- Apply a checklist of production-grade defaults to any Kubernetes workload
- Explain the consequences of skipping resource requests/limits, probes, or image pinning
- Configure a baseline `securityContext` that runs containers safely without root privileges
- Use namespaces, labels, and ResourceQuotas to keep multi-team clusters organized and governed
- Protect critical workloads from voluntary disruption with a PodDisruptionBudget
- Explain what GitOps is and why declarative, Git-stored manifests beat imperative `kubectl` commands
- Configure graceful shutdown so rolling updates cause zero dropped requests
- Describe where logging and monitoring fit into the overall production picture, and what's coming next in this course

---

## Prerequisites for This Chapter

- Resource requests, limits, and QoS classes — Chapter 9
- Liveness, readiness, and startup probes — Chapter 10
- Rolling update mechanics for Deployments — Chapter 5
- Imperative vs. declarative management of Kubernetes objects — Chapter 3
- DaemonSets, used by log collection agents — Chapter 12
- Helm chart labeling conventions — Chapter 13
- Docker image best practices (multi-stage builds, tag pinning) — Topic 4, Chapters 8 and 13

---

## Always Set Resource Requests and Limits

Chapter 9 introduced requests and limits as the mechanism the scheduler and kubelet use to place and constrain Pods. In production, skipping this isn't a minor omission — it actively puts a workload at risk.

A container with no `resources` block at all gets the **BestEffort** QoS class — the lowest priority Kubernetes assigns. When a node runs low on memory, the kubelet evicts BestEffort Pods *first*, before anything with a request set. Your Pod doesn't need to be the one misbehaving to get killed — it just needs to be the least-protected tenant on a node where *something else* is consuming memory.

```yaml
# BAD — no resources block at all: BestEffort QoS, first to be evicted
containers:
  - name: app
    image: myapp:1.4.0

# GOOD — Burstable or Guaranteed QoS, protected under node pressure
containers:
  - name: app
    image: myapp:1.4.0
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

Beyond eviction order, missing requests silently breaks CPU-based Horizontal Pod Autoscaling (Chapter 14) — utilization percentage has no baseline to compute against. And without limits, a single runaway container can consume an entire node's CPU or memory, degrading every other Pod scheduled there — the classic "noisy neighbor" problem that resource governance exists specifically to prevent.

---

## Always Configure Liveness and Readiness Probes

Chapter 10 covered the mechanics; here's why skipping probes is a production risk, not just an incomplete config.

Without a **readiness probe**, a Service starts sending traffic to a Pod the instant its container process starts — even if the application is still loading configuration, warming a cache, or waiting for a database connection pool. Requests during that window fail, and during a rolling update (Chapter 5), the new Pod is considered "ready" for traffic before it actually is, causing a burst of errors on every deploy.

Without a **liveness probe**, a Pod that has deadlocked or hung — process still running, but no longer doing useful work — is never restarted. Kubernetes has no way to distinguish "healthy but idle" from "silently broken" unless you tell it how to check.

```yaml
readinessProbe:
  httpGet:
    path: /healthz/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /healthz/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3
```

Treat "no probes configured" the same as "no resource limits configured" — both are missing safety nets that Kubernetes assumes you'll provide, not defaults it applies for you.

---

## Never Use the `latest` Tag; Pin Image Versions

The same principle from the Docker course (Topic 4) applies with even more force inside a Deployment, because Kubernetes adds a second layer of ambiguity on top of the mutable-tag problem: `imagePullPolicy`.

```yaml
# BAD
image: myapp:latest

# GOOD
image: myapp:1.4.0
# Better still, for full reproducibility:
image: myapp@sha256:a5127df1c6b5e5b3f8e9c2d1a4f6b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5
```

`imagePullPolicy` controls when the kubelet re-fetches an image from the registry:

| Policy | Behavior |
|---|---|
| `Always` | Pulls the image (and re-checks the tag against the registry) on every Pod start, even if a local copy exists |
| `IfNotPresent` | Uses the local image if any copy with that tag exists on the node — does not re-check the registry |
| `Never` | Only uses a local image; fails if it's not already present on the node |

Kubernetes defaults `imagePullPolicy` to `Always` whenever the tag is `latest`, and to `IfNotPresent` for any other tag. This is precisely why `latest` is dangerous in a cluster: a Pod that gets rescheduled to a different node — during a routine rolling update, a node drain, or a crash recovery — pulls whatever `latest` currently points to on the registry at that moment, which may be a completely different build than the Pod running right next to it on another node. Two replicas of the "same" Deployment can silently end up running two different versions of your application. Pinning to an immutable tag (or better, a digest) guarantees every replica, on every node, at every point in time, runs bit-for-bit the same image.

---

## Run as Non-Root and Use a Restrictive Security Context

Every container should run with the least privilege it needs to function, as a baseline default — not an exception applied only to "sensitive" workloads.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: app
      image: myapp:1.4.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

- `runAsNonRoot: true` — the kubelet refuses to start the container if its image would run as UID 0. This forces the container image itself to declare a non-root user (typically via a `USER` instruction in the Dockerfile).
- `readOnlyRootFilesystem: true` — the container's filesystem is mounted read-only; if the application needs to write anything (temp files, caches), mount an explicit `emptyDir` volume at that specific path rather than leaving the whole filesystem writable.
- `allowPrivilegeEscalation: false` — prevents a process from gaining more privileges than its parent (blocks setuid-based escalation).
- `capabilities.drop: [ALL]` — strips every Linux capability by default; add back only the specific ones a workload genuinely needs (e.g., `NET_BIND_SERVICE` to bind to a port below 1024), rather than starting from "everything allowed" and trying to remember what to remove.

This is the baseline every workload should carry, full stop. Cluster-wide enforcement of these settings — Pod Security Standards, admission controllers that reject non-compliant Pods outright — is covered in Topic 9, Advanced Kubernetes. For now, treat this `securityContext` block as something you add to every Deployment you write, the same way you'd never skip `resources` or probes.

---

## Use Namespaces to Separate Environments/Teams and Apply ResourceQuotas

Chapter 9 introduced namespaces as a logical partitioning mechanism and ResourceQuotas as a way to cap total resource consumption within one. In production, this pairing is what prevents one team or one environment from starving another on a shared cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-checkout-prod
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-checkout-quota
  namespace: team-checkout-prod
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
```

A common convention is one namespace per team per environment (`team-checkout-dev`, `team-checkout-staging`, `team-checkout-prod`), each with its own quota sized to that team's budget. Without this, a single misconfigured Deployment (say, one that accidentally scales to 500 replicas) in a shared namespace can consume the entire cluster's capacity and take down every other team's workloads with it — a ResourceQuota turns that into a contained, namespace-scoped failure instead of a cluster-wide incident.

---

## Label and Annotate Everything Consistently

Labels are how every other Kubernetes mechanism you've learned — Services (Chapter 6), Deployments' Pod selectors (Chapter 5), NetworkPolicies, and Helm itself (Chapter 13) — finds the right objects. Inconsistent labeling doesn't just look messy; it silently breaks selectors and makes cluster-wide tooling (dashboards, cost allocation, log aggregation) far less useful.

Kubernetes and the broader ecosystem (including Helm) have converged on a standard label set:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: checkout-service
    app.kubernetes.io/instance: checkout-service-prod
    app.kubernetes.io/version: "1.4.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: helm
```

| Label | Purpose |
|---|---|
| `app.kubernetes.io/name` | The application's name, independent of any specific deployment of it |
| `app.kubernetes.io/instance` | A unique identifier for this particular installation (useful when one chart is installed multiple times, as in Chapter 13) |
| `app.kubernetes.io/version` | The application version currently running — useful for auditing what's deployed where |
| `app.kubernetes.io/managed-by` | The tool managing this object's lifecycle (`helm`, `kubectl`, `kustomize`) — helps other engineers know whether it's safe to hand-edit |

Adopting this standard set (rather than inventing your own ad hoc labels per team) means dashboards, cost-reporting tools, and `kubectl` label selectors all work consistently across every application in the cluster, and it's exactly what Helm applies automatically to resources it manages.

---

## Set PodDisruptionBudgets for Critical Workloads

Not every Pod termination is caused by a crash. **Voluntary disruptions** — a cluster administrator draining a node for a Kubernetes version upgrade, a Cluster Autoscaler (Chapter 14) removing an underutilized node, routine maintenance — can terminate multiple Pods belonging to the same Deployment simultaneously, purely because they happened to land on the node being drained.

A **PodDisruptionBudget (PDB)** tells Kubernetes the minimum availability a workload requires, so voluntary disruption operations (which respect PDBs) won't evict enough Pods at once to violate it.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-pdb
spec:
  minAvailable: 2          # or use maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: checkout-service
```

With this PDB in place, if an administrator runs `kubectl drain` on a node hosting two of your five `checkout-service` replicas, and evicting both would drop availability below `minAvailable: 2`, the drain operation blocks or retries on the second Pod until it's safe to proceed — instead of silently taking your service down to below the availability you require. PDBs don't protect against unplanned failures (a node crashing outright) — only against operations that check for permission first, which is exactly what routine cluster maintenance does.

---

## Prefer Declarative Manifests Stored in Git Over Imperative kubectl Commands

Chapter 3 introduced the distinction between imperative commands (`kubectl create deployment ...`, `kubectl scale ...`) and declarative manifests (`kubectl apply -f deployment.yaml`). In production, the declarative approach isn't just "cleaner" — it's the only approach that gives you an audit trail, a rollback path, and a source of truth that doesn't live only inside a cluster's current state.

```bash
# Imperative — works, but leaves no record of *why* or *what changed*
kubectl scale deployment/checkout --replicas=8
kubectl set image deployment/checkout checkout=myapp:1.5.0

# Declarative — the manifest in Git is the source of truth
# 1. Edit deployment.yaml in a Git repo: replicas: 8, image: myapp:1.5.0
# 2. Open a pull request, get it reviewed
# 3. Merge, then apply
kubectl apply -f deployment.yaml
```

Taking this one step further is the industry-standard practice known as **GitOps**: the cluster's entire desired state lives in a Git repository, and a reconciliation tool running inside (or against) the cluster — most commonly **Argo CD** or **Flux** — continuously compares the live cluster state against what's declared in Git and automatically applies any difference. Nobody runs `kubectl apply` by hand at all; the only way to change the cluster is to open a pull request. This gives you the same review, approval, and rollback guarantees for infrastructure that you already expect for application code. A full GitOps workflow — installing Argo CD/Flux, structuring a Git repo for multi-cluster/multi-environment reconciliation, handling secrets in a GitOps-safe way — is covered in depth in Topic 9, Advanced Kubernetes. For now, the actionable takeaway is: **store every manifest you apply in Git, and prefer `kubectl apply -f` (or `helm upgrade --install` from a values file in Git) over one-off imperative commands**, even before you adopt a full GitOps tool.

---

## Keep Containers Small and Use Multi-Stage Builds

This is fundamentally a Docker-layer practice (covered in depth in Topic 4), but it has direct, measurable consequences at the Kubernetes layer, which is why it belongs on this checklist too.

A smaller image means:

- **Faster rollouts.** Every rolling update (Chapter 5) requires the new image to be pulled onto each node before the new Pod can start. A 1.5 GB image pulling across dozens of nodes during a deploy can add minutes of latency to a release; a 150 MB multi-stage build pulls in seconds.
- **Faster autoscaling.** When Cluster Autoscaler (Chapter 14) adds a brand-new node with no cached layers at all, every image that needs to run on it must be pulled from scratch — image size directly affects how quickly a freshly-added node can start serving traffic under load.
- **Smaller attack surface.** A multi-stage build's final image contains only the runtime and compiled artifacts, not compilers, build tools, or package manager caches — fewer binaries an attacker can exploit if they gain a foothold in a container.

```dockerfile
# Multi-stage build — the pattern from the Docker course, still paying off in Kubernetes
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

---

## Graceful Shutdown

Chapter 5 covered rolling updates as Kubernetes' mechanism for zero-downtime deploys — but a rolling update is only actually zero-downtime if each terminating Pod shuts down cleanly instead of dropping in-flight requests.

When Kubernetes terminates a Pod, it sends the container process a `SIGTERM`, then waits up to `terminationGracePeriodSeconds` (default 30 seconds) before force-killing it with `SIGKILL`. If your application doesn't handle `SIGTERM` — finishing in-flight requests, closing database connections cleanly, stopping the acceptance of new work — it either gets killed mid-request (dropped connections) or simply hangs until the grace period expires and gets forcibly terminated anyway.

```yaml
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: app
      image: myapp:1.4.0
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

The `preStop` hook above addresses a specific and easy-to-miss race condition: when a Pod is marked for termination, it takes a brief moment for that Pod to be removed from a Service's list of endpoints across the whole cluster (kube-proxy rules, Chapter 6, need to propagate). Without a short delay, the Pod's application process can receive `SIGTERM` and start shutting down *before* some nodes have finished removing it from load-balancing rotation — resulting in a handful of requests routed to a Pod that's already stopped accepting connections. A brief `sleep` in `preStop` (a few seconds is typical) gives that propagation time to complete before the application itself begins its shutdown sequence.

The application-side responsibility (handling `SIGTERM` to drain in-flight work) can't be configured away in YAML — it has to be written into the application itself. This is exactly why "graceful shutdown" is a joint responsibility: Kubernetes gives you the grace period and the hook, but your code has to use it correctly.

---

## Structured Logging to stdout/stderr, Never to Files Inside the Container

Kubernetes' entire logging model assumes a container's logs are whatever it writes to `stdout`/`stderr` — the container runtime captures that stream, and `kubectl logs` reads it back. Writing application logs to a file inside the container instead breaks this model in two ways: the log is invisible to `kubectl logs`, and it's lost entirely the moment the container is destroyed (which happens routinely — every rolling update, every crash restart, every node drain).

The DaemonSet-based log collection pattern from Chapter 12 (a log-shipping agent like Fluent Bit running once per node, reading every container's stdout/stderr and forwarding it to a central system) only works if applications cooperate by logging to the standard streams in the first place:

```
# GOOD — application logs to stdout, kubectl logs and the node's
# log-collector DaemonSet both see it
{"timestamp":"2026-07-01T10:15:00Z","level":"info","msg":"order placed","order_id":"8842"}

# BAD — application writes to /var/log/app/app.log inside the container
# invisible to kubectl logs, lost when the container is destroyed
```

Structuring logs as JSON (rather than free-text) additionally makes them queryable once they reach a central aggregation system — you can filter on `level:error` or `order_id:8842` instead of grepping raw text.

---

## Health of the Whole System: Monitoring and Observability Basics

Everything in this chapter helps a single workload behave well in isolation, but running Kubernetes in production also requires visibility across the *whole* cluster: are nodes under memory pressure, is a particular Deployment's error rate climbing, did that last rollout's latency get worse? The de facto standard stack for this is **Prometheus** (a time-series metrics database that scrapes metrics from applications, `kube-state-metrics`, and node exporters) paired with **Grafana** (dashboards and alerting on top of that data).

You've already met a narrow slice of this world in this course — `metrics-server` (Chapter 14) exists specifically to serve the *narrow* Metrics API that HPA needs, and is not a substitute for a full observability stack; Prometheus captures far richer, longer-retained, more queryable metrics, and is usually what backs custom-metric HPA scaling in real production setups. Full depth on setting up Prometheus, writing alerting rules, building Grafana dashboards, and correlating metrics/logs/traces is the entire subject of Topic 10, Monitoring & Logging, later in this roadmap. For now, know that this is the next major capability layer on top of everything covered in this course: you can deploy, expose, configure, store data for, autoscale, and package your workloads — the next step is seeing, at a glance, whether all of it is actually healthy.

---

## Summary

Production-readiness in Kubernetes is less about any single advanced feature and more about consistently applying a checklist of defaults to every workload: resource requests/limits (avoiding BestEffort QoS and enabling correct HPA behavior), liveness/readiness probes (avoiding traffic to unready Pods and undetected hangs), pinned image tags (avoiding silent version drift across replicas), a restrictive `securityContext` (non-root, read-only filesystem, dropped capabilities), namespace-scoped ResourceQuotas (containing the blast radius of misconfiguration), consistent `app.kubernetes.io/*` labeling (keeping selectors and tooling reliable), PodDisruptionBudgets (protecting availability during routine cluster maintenance), declarative Git-stored manifests as the source of truth (with GitOps as the natural next step), small multi-stage-built images (faster rollouts and autoscaling, smaller attack surface), graceful shutdown handling (`terminationGracePeriodSeconds`, `preStop`, and application-level `SIGTERM` handling), and stdout/stderr-only structured logging (so DaemonSet-based collectors can do their job). Layered on top of all of this, Prometheus and Grafana provide the cluster- and application-wide visibility that ties everything together — the subject of Topic 10.

---

## Knowledge Check

1. What QoS class does a container get if it has no `resources` block at all, and why does that matter during node memory pressure?
2. Why does `imagePullPolicy: Always` combined with a `latest` tag risk two replicas of the same Deployment running different actual image versions?
3. What three `securityContext` settings form a reasonable non-root baseline, and what does each one prevent?
4. What specifically does a PodDisruptionBudget protect against, and what kind of Pod termination is it powerless to prevent?
5. Explain the difference between "declarative manifests in Git" and full "GitOps" — what additional piece does GitOps add?
6. Why does writing application logs to a file inside the container break Kubernetes' logging model, even if the file itself contains perfectly good structured logs?

---

## Hands-On Exercise

1. Take any Deployment manifest you've written earlier in this course (or create a simple one) and audit it against this chapter's checklist — note every missing item.
2. Add a `resources` block with sensible requests and limits, and both a `readinessProbe` and `livenessProbe`.
3. Replace any `latest` or untagged image reference with a pinned version tag.
4. Add a baseline `securityContext` at both the Pod and container level (`runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`) and confirm with `kubectl apply` that the Pod still starts successfully — adjust your container image/Dockerfile if it fails due to needing root or write access somewhere.
5. Add standard `app.kubernetes.io/*` labels to the Deployment and its Pod template.
6. Create a `PodDisruptionBudget` for the Deployment with `minAvailable` set to a sensible value for its replica count, and confirm it exists with `kubectl get pdb`.
7. Add `terminationGracePeriodSeconds` and a `preStop` hook with a short `sleep`, then trigger a rolling update (`kubectl rollout restart deployment/<name>`) and watch `kubectl get pods -w` to confirm old Pods terminate cleanly alongside new ones starting.

---

## Further Reading

- [Kubernetes Documentation — Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Kubernetes Documentation — Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes Documentation — Specifying a Disruption Budget for your Application](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes Documentation — Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Kubernetes Documentation — Pod Lifecycle (Termination)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-scaling-and-autoscaling.md">← Previous: Scaling and Autoscaling</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-common-mistakes.md">Next: Common Mistakes and Pitfalls →</a>
</div>
