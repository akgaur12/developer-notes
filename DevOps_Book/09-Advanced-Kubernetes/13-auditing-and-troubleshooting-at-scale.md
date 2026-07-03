# Chapter 13 — Auditing and Troubleshooting at Scale

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what Kubernetes audit logging records, configure an audit policy, and use audit logs for security forensics and compliance
- Use ephemeral containers (`kubectl debug`) to troubleshoot Pods running minimal or distroless images with no shell
- Use `kubectl debug node/<node>` to inspect a node without SSH access
- Use `crictl` to inspect containers directly through the container runtime when the kubelet or API server is unreliable
- Use `etcdctl` to check the health of an etcd cluster during a control-plane incident
- Work through structured diagnostic playbooks for the most common cluster-wide failure scenarios
- Apply a priority-ordered cluster health checklist during a live incident

---

## Prerequisites

- RBAC and authentication — Chapter 2 (audit logs record identity, which requires understanding how Kubernetes authenticates requests)
- Cluster administration, kubeadm, and etcd backup basics — Chapter 9
- Autoscaling and CPU throttling behavior — Chapter 12 (referenced in the DNS and eviction scenarios below)
- Namespaces, health probes, and `kubectl exec`/`kubectl logs` — Kubernetes Basics (Topic 8)

---

## 13.1 Kubernetes Audit Logging

Every other troubleshooting tool in this chapter tells you the *current state* of the cluster. Audit logs tell you *how it got there* — a durable, tamper-evident record of every request made to the API server: who made it, from where, what they asked for, and what happened.

This matters for two distinct reasons:

- **Security forensics.** "Who deleted the `payments` Deployment in production at 2 AM?" is unanswerable from `kubectl get events` (events expire after about an hour by default) or from application logs (the Deployment is gone, so are its Pods' logs). It is trivially answerable from an audit log, which records the identity, source IP, and exact API call.
- **Compliance.** Standards like SOC 2, PCI-DSS, and HIPAA require organizations to demonstrate who has access to production systems and to produce evidence of what they did with it. Audit logs are frequently the artifact an auditor asks for directly.

### What gets recorded

Every request to the kube-apiserver — `kubectl` commands, controller reconciliation loops, webhook calls, even other components like the scheduler and kubelet — passes through the audit pipeline. A single audit event captures:

| Field | Example |
|---|---|
| `user` | `system:serviceaccount:ci:deployer` or a human's OIDC identity |
| `verb` | `delete`, `create`, `update`, `patch`, `get`, `list` |
| `objectRef` | `apps/v1, Deployments, namespace=payments, name=api` |
| `sourceIPs` | The IP the request came from |
| `requestReceivedTimestamp` | When the request hit the API server |
| `responseStatus` | HTTP status code returned |
| `stage` | `RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic` |

### Audit policy: choosing what to log

Logging every field of every request/response body for every API call would produce an unusable volume of data — a busy production cluster can receive thousands of API requests per minute just from controllers reconciling state. The **audit policy** lets you choose, per rule, how much detail to capture for which requests, using four levels:

| Level | Records | Use for |
|---|---|---|
| `None` | Nothing | High-volume, low-value requests (e.g., health checks from monitoring ServiceAccounts) |
| `Metadata` | User, timestamp, verb, resource, response code — no request/response bodies | The default for most requests; enough for "who did what" without the payload |
| `Request` | Metadata plus the request body | Mutating operations on sensitive resources, where you need to know *what* was changed |
| `RequestResponse` | Metadata plus both request and response bodies | Reserved for the highest-sensitivity resources (Secrets, RBAC bindings) — expensive, use sparingly |

A minimal audit policy that captures the signal that actually matters without drowning in noise:

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Don't log read-only requests from the monitoring/health-check identities at all
  - level: None
    users: ["system:serviceaccount:monitoring:kube-state-metrics"]
    verbs: ["get", "list", "watch"]

  # Full request+response body for Secrets and RBAC changes — highest sensitivity
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
      - group: "rbac.authorization.k8s.io"
        resources: ["*"]

  # Request body (what changed) for anything that mutates a workload
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "apps"
        resources: ["deployments", "statefulsets", "daemonsets"]

  # Metadata only for everything else — cheap, still answers "who/when/what verb"
  - level: Metadata
    omitStages:
      - RequestReceived
```

Wire this into the API server via `--audit-policy-file` and `--audit-log-path` (kubeadm-managed clusters set these as static Pod flags on `kube-apiserver`; managed services like EKS/GKE/AKS expose audit logging as a control-plane toggle instead, shipping logs to CloudWatch/Cloud Logging/Log Analytics rather than a local file). Ship the resulting log stream to your log aggregation system, not just to disk on the control-plane node — a node that's part of the incident is not where you want the only copy of the evidence.

```bash
# Find who deleted a Deployment, once audit logs are flowing into your log system
# (example query shape, adapt to your log backend)
grep '"verb":"delete"' audit.log \
  | grep '"resource":"deployments"' \
  | grep '"namespace":"payments"'
```

---

## 13.2 Advanced `kubectl debug`

Topic 8 covered `kubectl exec -it <pod> -- sh` as the way to get a shell inside a running container. That works fine when the image has a shell. It does not work at all against a **distroless or minimal image** (Chapter recap from the Docker course, Topic 4) — precisely the kind of image you should be shipping to production for a smaller attack surface. There is no `sh`, no `busybox`, no debugging tools of any kind inside the container, by design.

### Ephemeral containers: debugging a running Pod with no shell

An **ephemeral container** is injected into an already-running Pod's namespaces (network, process, sometimes filesystem) without restarting it. It brings its own image — a debugging toolbox — while sharing the target container's network and process view.

```bash
# Attach a busybox debug container into a running Pod,
# sharing the network namespace of the "app" container
kubectl debug -it my-app-7d9f8-x --image=busybox --target=app

# Once inside, you can curl localhost (the app's network namespace),
# inspect /proc, check DNS resolution, etc. — none of which the
# distroless "app" container itself can do
```

`--target` matters: without it, the ephemeral container shares the Pod's network namespace but not its process namespace, so you can't see the target container's actual processes. With `--target=app`, running `ps aux` inside the ephemeral container shows the real application process, and you can inspect it directly (e.g., `/proc/1/environ`, open file descriptors under `/proc/1/fd`).

A richer toolbox image than plain `busybox` is often worth using for real incidents:

```bash
kubectl debug -it my-app-7d9f8-x \
  --image=nicolaka/netshoot \
  --target=app
# netshoot bundles curl, dig, tcpdump, iproute2, netcat — a full
# network-debugging toolkit, useful for the DNS scenario in 13.4
```

Ephemeral containers are added to the Pod's `spec.ephemeralContainers` and are visible via `kubectl get pod <pod> -o yaml` — they are not silent. They also cannot be removed once added (the Pod must be recreated to clear them), so use them for diagnosis, not as a permanent sidecar pattern.

### Node-level debugging: `kubectl debug node/<node>`

Sometimes the problem isn't inside a container at all — it's the node's kubelet, container runtime, or filesystem. `kubectl debug node/<node>` schedules a privileged debugging Pod onto that node with the **host's root filesystem** mounted at `/host`, without requiring SSH access to the node.

```bash
kubectl debug node/worker-03 -it --image=busybox

# Inside the debug Pod:
chroot /host
# Now you're effectively "on" the node itself:
systemctl status kubelet
journalctl -u kubelet --no-pager | tail -100
df -h                       # disk pressure?
cat /etc/kubernetes/kubelet.conf
```

This is the direct replacement for "SSH to the node," and it's the right default even when SSH access *is* available, because it goes through the same RBAC and audit pipeline as every other `kubectl` command — the access is logged and permission-controlled like everything else, instead of being a side channel around Kubernetes entirely.

### `kubectl exec` vs. `kubectl debug` — when to use which

| | `kubectl exec` | `kubectl debug` (ephemeral container) | `kubectl debug node/` |
|---|---|---|---|
| Target | Existing container, must already have a shell | Any Pod, even with no shell/tools in its image | Any node, no SSH required |
| Modifies the Pod | No | Yes — adds an ephemeral container (visible, permanent record) | N/A — separate debug Pod |
| Works on distroless images | No | Yes | N/A |
| Typical use | Quick check on a normal (non-minimal) image | Debugging a production distroless/minimal container | kubelet/runtime/filesystem issues on a node |

---

## 13.3 Node-Level Tooling: `crictl` and `etcdctl`

`kubectl` talks to the API server. When the API server itself is degraded, or when you need to know what the container runtime on a specific node actually sees (which may disagree with what the API server *thinks* is running there), you need tools that talk to the node and the runtime directly.

### `crictl` — talking to the container runtime directly

`crictl` is a CLI for the Container Runtime Interface (CRI) — it talks to containerd or CRI-O directly, bypassing the kubelet and API server entirely. It's the tool of choice when you suspect the disagreement is *between* Kubernetes and the runtime — e.g., the API server reports a Pod as `Running` but something is wrong at the container level, or the kubelet itself is unresponsive and you need ground truth.

```bash
# Run from a shell on the node itself (SSH, or via `kubectl debug node/` + chroot /host)

# List all containers the runtime knows about, including ones kubelet
# doesn't currently report correctly
crictl ps -a

# List images cached locally on this node
crictl images

# Inspect a specific container's runtime-level state (not the K8s object)
crictl inspect <container-id>

# Pull a container's logs directly from the runtime,
# useful when `kubectl logs` itself is failing because the API
# server or kubelet is the thing that's broken
crictl logs <container-id>

# Check whether the runtime itself is healthy
crictl info
```

`crictl ps -a` showing a container in `Exited` state with a non-zero exit code, while `kubectl get pods` on that node hasn't updated yet, is a common sign the kubelet itself is the bottleneck (busy, crashed, or unable to reach the API server to report status) — see the `NotReady` node playbook below.

### `etcdctl` — troubleshooting an unhealthy etcd cluster

Chapter 9 introduced `etcdctl` for backup and restore. In an incident, it's also your primary tool for confirming whether etcd — the single source of truth for all cluster state — is actually healthy, since a struggling etcd cluster produces symptoms that look like "the whole cluster is broken" everywhere else.

```bash
# Run against each etcd member (usually control-plane nodes) with TLS certs configured
export ETCDCTL_API=3
alias etcdctl='etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

# Is this member healthy, and what is its request latency?
etcdctl endpoint health --write-out=table

# Compare all members — looking for one that disagrees (raft lag) or is unreachable
etcdctl endpoint status --cluster --write-out=table

# Who is the current leader? Frequent leader elections are a red flag —
# they indicate network partitions or disk I/O too slow to keep up with
# etcd's heartbeat/election timeouts
etcdctl endpoint status --cluster --write-out=table | grep -i leader

# How large is the database file? etcd degrades badly as it approaches
# its default 2GB quota — check this before assuming the problem is elsewhere
etcdctl endpoint status --write-out=table
# (DB SIZE column)

# Alarms — NOSPACE is the most common, triggered by exceeding the quota
etcdctl alarm list
```

The single most important thing to internalize about etcd: it is **latency-sensitive to disk fsync time**, not just disk throughput. Slow disks (network-attached storage with high write latency, noisy-neighbor contention on shared storage) cause etcd heartbeats to miss their deadlines, which triggers leader elections, which cascades into API server timeouts — because every write through the API server round-trips through etcd's Raft consensus. This is the mechanism behind the first failure scenario below.

---

## 13.4 Cluster-Wide Failure Scenario Playbooks

Each playbook follows the same shape: symptom, ordered diagnostic steps, and the fix. Work top-to-bottom — the ordering exists because later steps are often *caused by* earlier ones, and fixing symptom 3 while root cause 1 is still active wastes the only time you have during an incident.

### Playbook 1: API server is slow or unresponsive cluster-wide

**Symptom:** `kubectl` commands hang or time out. Every controller's reconciliation is delayed. New Pods take much longer than usual to schedule.

1. **Check etcd health and latency first** — the API server does almost nothing without a round-trip to etcd for every read (unless served from cache) and every write.
   ```bash
   etcdctl endpoint status --cluster --write-out=table
   etcdctl endpoint health --write-out=table
   ```
   Look specifically for high `RAFT TERM` values changing frequently (repeated leader elections) and DB size approaching the quota.

2. **Check API server resource usage** on the control-plane nodes — CPU throttling or memory pressure on kube-apiserver itself directly causes request queuing.
   ```bash
   kubectl top pod -n kube-system -l component=kube-apiserver
   kubectl describe pod -n kube-system -l component=kube-apiserver | grep -A5 Limits
   ```

3. **Check for a runaway client hammering the API server.** A misbehaving controller that isn't respecting client-side rate limiting (missing or misconfigured `QPS`/`Burst` settings in its client config, or a reconcile loop stuck retrying without backoff) can flood the API server with requests. The API server's own metrics identify the culprit directly:
   ```bash
   # apiserver_request_total broken down by client identity — look for
   # one identity dominating the request volume
   kubectl get --raw /metrics | grep apiserver_request_total | grep -v '"get"' | sort -t'=' -k2 -rn | head -20
   ```
   In practice, teams query this through Prometheus rather than raw `--raw /metrics` scraping, but the metric name and the diagnostic question ("which `client` or `user_agent` label has an outsized share of requests") is the same either way.

**Fix:** If etcd is the bottleneck, address the underlying disk I/O problem (move etcd to faster/dedicated disks, reduce write volume). If a specific controller is the culprit, apply client-side rate limiting or a `Priority and Fairness` API server flow-control rule to cap its impact without taking the whole API server down while you fix the controller properly.

### Playbook 2: DNS resolution failing cluster-wide

**Symptom:** Pods across many namespaces suddenly can't resolve Service names or external hostnames; connection errors mention DNS timeouts, not connection refused.

1. **Check CoreDNS Pod health and logs.**
   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
   ```
   Look for `SERVFAIL` responses, restart counts, or `OOMKilled` terminations.

2. **Check for CoreDNS being CPU-throttled under load** — recall from Chapter 12 that a CPU `limit` set too tight causes throttling even when the node has spare capacity, and CoreDNS is exactly the kind of latency-sensitive workload where throttling shows up immediately as timeouts rather than gradual slowness.
   ```bash
   kubectl top pod -n kube-system -l k8s-app=kube-dns
   # Check the throttling metric directly if metrics-server/Prometheus exposes it
   kubectl get --raw /metrics | grep container_cpu_cfs_throttled_seconds_total
   ```
   A cluster that recently grew in Pod count without CoreDNS's `replicas` or resource limits scaling proportionally is a common root cause — query volume grows with the number of Pods making lookups, not just with traffic.

3. **Check `nodelocaldns` if it's deployed.** Many clusters run a `NodeLocalDNS` DaemonSet as a caching layer in front of CoreDNS specifically to absorb query volume and avoid this exact failure mode. If it's in use, check whether its cache is being bypassed or whether the DaemonSet itself is unhealthy on affected nodes.
   ```bash
   kubectl get pods -n kube-system -l k8s-app=node-local-dns -o wide
   ```

**Fix:** Scale CoreDNS replicas, raise its CPU limit (or remove the CPU limit and keep only a request, since CoreDNS is a case where throttling is worse than occasional extra CPU usage), and add `nodelocaldns` if not already present to reduce load on the central CoreDNS Deployment.

### Playbook 3: A node goes `NotReady`

**Symptom:** `kubectl get nodes` shows a node stuck `NotReady`; Pods scheduled there eventually get evicted and rescheduled elsewhere.

1. **Check kubelet logs on the node** (via `kubectl debug node/<node>` and `chroot /host`, or SSH):
   ```bash
   journalctl -u kubelet --no-pager | tail -200
   ```
   Common findings: kubelet crashed and isn't restarting, kubelet certificate expired, or kubelet can't reach the container runtime socket.

2. **Check node resource pressure** — the node's own conditions report this directly:
   ```bash
   kubectl describe node worker-03 | grep -A10 Conditions
   # Look for MemoryPressure, DiskPressure, PIDPressure all reporting True
   ```

3. **Check network connectivity from the node to the API server.** A `NotReady` status is fundamentally the API server not having heard a heartbeat from that node's kubelet within the expected window — this can be the kubelet's fault, or it can be the network between them.
   ```bash
   # From the node (via kubectl debug node/ + chroot /host)
   curl -k https://<api-server-endpoint>:6443/healthz
   ```

**Fix:** Restart or reinstall the kubelet if it crashed; free up disk/memory if under pressure (often container image garbage collection catching up, or a runaway process on the node itself, not a Kubernetes-managed one); fix the network path (security group, firewall rule, or a flaky NIC) if connectivity is the root cause.

### Playbook 4: Sudden mass Pod evictions across a node

**Symptom:** Many unrelated Pods on one or more nodes get evicted at roughly the same time, with `Evicted` status and a reason referencing resource pressure.

1. **Check node disk/memory pressure directly:**
   ```bash
   kubectl describe node worker-05 | grep -A10 Conditions
   kubectl describe node worker-05 | grep -A5 "Allocated resources"
   ```

2. **Correlate with recent Pod scheduling that caused over-commitment.** This ties directly back to Chapter 12: if requests were set unrealistically low (or missing) relative to actual usage, the scheduler can pack far more Pods onto a node than it can actually sustain once they're all under real load simultaneously — the node runs fine at low traffic and falls over during a shared traffic spike.
   ```bash
   kubectl get events --field-selector reason=Evicted -A --sort-by=.lastTimestamp
   # Cross-reference the eviction timestamps against any recent HPA
   # scale-out or new Deployment rollout landing on the same node
   kubectl get events --field-selector reason=SuccessfulCreate -A --sort-by=.lastTimestamp
   ```

**Fix:** Immediately, cordon the affected node(s) and let the scheduler spread load elsewhere. Structurally, correct the resource requests that allowed over-commitment (Chapter 12's VPA-in-recommendation-mode guidance is the right long-term source of truth for what requests *should* be), and consider node-level eviction thresholds (`kubelet`'s `--eviction-hard` settings) if they're looser than they should be for your workload mix.

---

## 13.5 The Cluster Health Checklist

During an incident, check things in this order — top to bottom, not by intuition or by whichever team is loudest about being affected. Each layer, if unhealthy, produces symptoms that look exactly like the layer above it failing, which is why checking bottom-up wastes time chasing symptoms of a problem you haven't found yet.

| Priority | Layer | What to check | Command |
|---|---|---|---|
| 1 | Control plane health | Are kube-apiserver, kube-controller-manager, kube-scheduler all up and not restarting? | `kubectl get pods -n kube-system -l tier=control-plane` (or provider console for managed control planes) |
| 2 | etcd health | Member health, leader stability, DB size, request latency | `etcdctl endpoint health`, `etcdctl endpoint status --cluster` |
| 3 | Node health | Are all nodes `Ready`? Any resource pressure conditions? | `kubectl get nodes`, `kubectl describe node <name>` |
| 4 | Networking / DNS health | Can Pods resolve DNS? Is CNI healthy? Are NetworkPolicies unexpectedly blocking traffic? | `kubectl logs -n kube-system -l k8s-app=kube-dns`, `kubectl get pods -n kube-system -o wide` (CNI Pods) |
| 5 | Workload-level health | Are specific Deployments/Pods healthy once the platform layers above are confirmed fine? | `kubectl get pods -A \| grep -v Running`, `kubectl top pod` |

The discipline this encodes: don't debug an individual application's `CrashLoopBackOff` (layer 5) while etcd (layer 2) is silently timing out and causing every controller in the cluster to behave erratically — you'll "fix" symptoms that reappear the moment you move on, because the actual root cause was never addressed.

---

## 13.6 Best Practices

- **Enable audit logging before you need it.** The worst time to discover audit logging isn't enabled is during the security incident where you need it most.
- **Ship audit logs off-cluster immediately.** A log that only exists on the same control-plane disk that might be part of the incident is not a reliable forensic record.
- **Prefer `kubectl debug` over baking debugging tools into production images.** Keep production images minimal; bring your toolbox on demand instead.
- **Practice the failure playbooks before an incident, not during one.** A platform team that has run a `kind`/staging-cluster fire drill for "etcd is slow" reacts in minutes during a real incident; a team seeing it for the first time in production loses precious time to trial and error.
- **Always check etcd before assuming an application is at fault** for cluster-wide slowness — it's the layer most likely to produce misleading downstream symptoms.
- **Automate the health checklist** as a single script or runbook page that any on-call engineer, not just the person who wrote it, can execute under pressure at 3 AM.

## Common Mistakes (Preview)

Chapter 15 covers these in depth, but two are worth flagging here directly: teams that never enable or review audit logs until after an incident forces them to (no forensic trail exists when it's needed most), and teams that debug from the workload layer downward instead of the control-plane layer up, wasting time chasing symptoms of a root cause they haven't reached yet.

---

## Summary

Audit logs are the durable record of every API server request — who did what, when, and from where — configured via an audit policy that balances forensic value against log volume using `None`/`Metadata`/`Request`/`RequestResponse` levels. `kubectl debug` extends troubleshooting to Pods with no shell (ephemeral containers) and to nodes without SSH access (`kubectl debug node/`). When `kubectl` itself is unreliable, `crictl` talks to the container runtime directly and `etcdctl` reports on the health of the cluster's single source of truth. Cluster-wide failures — a slow API server, cluster-wide DNS failures, a `NotReady` node, mass Pod evictions — each follow a structured diagnostic path, and a priority-ordered health checklist (control plane → etcd → nodes → networking/DNS → workloads) keeps an incident response focused on root cause instead of downstream symptoms.

---

## Knowledge Check

1. What is the difference between the `Metadata`, `Request`, and `RequestResponse` audit levels, and why would you use different levels for Secrets versus routine `get` requests?
2. Why can't `kubectl exec` be used to debug a container running a distroless image, and what command solves this without restarting the Pod?
3. When would you reach for `crictl` instead of `kubectl` or `kubectl logs`?
4. A cluster is reporting cluster-wide slowness. Why should you check etcd health before looking at any individual Deployment?
5. What CoreDNS-specific failure mode connects directly to the CPU throttling behavior covered in Chapter 12, and how would you detect it?
6. Put these in the correct diagnostic priority order during an incident: node health, workload health, etcd health, control plane health, networking/DNS health.

---

## Hands-On Exercise

**Diagnose a Simulated Cluster Incident**

Using a local `kind` or `minikube` cluster:

1. Deploy a distroless-based application (any `gcr.io/distroless/*` image running a simple HTTP server) and confirm `kubectl exec -it <pod> -- sh` fails as expected. Use `kubectl debug -it <pod> --image=busybox --target=<container>` to get a working shell in the Pod's network namespace instead, and `curl localhost:<port>` from inside it to confirm the app is actually reachable.
2. Run `kubectl debug node/<node>` against one of your cluster's nodes, `chroot /host`, and inspect `journalctl -u kubelet` output — note what a healthy kubelet's log stream looks like so you can recognize an unhealthy one later.
3. If your cluster setup gives you etcd access (kubeadm-based clusters do; most managed clusters don't expose this), run `etcdctl endpoint status --cluster --write-out=table` and identify the current leader and DB size.
4. Write a one-page runbook (a plain Markdown file is fine) implementing the priority-ordered health checklist from section 13.5 as a series of copy-pasteable commands, so that a teammate unfamiliar with the cluster could execute it during an incident.

---

## Real-World Scenario: "Everything Is Slow"

An on-call platform engineer is paged at 2:47 AM: `"Elevated latency across all services — multiple teams reporting timeouts."` No single team's dashboards show an obvious root cause; it looks like a platform-wide problem, not an application bug.

Following the priority-ordered checklist from section 13.5:

1. **Control plane health** — `kubectl get pods -n kube-system -l tier=control-plane` shows all three kube-apiserver replicas `Running`, no recent restarts. Not obviously the control plane itself, but `kubectl` commands are noticeably slower to return than usual — a hint, not a conclusion.

2. **etcd health** — `etcdctl endpoint status --cluster --write-out=table` shows something clearly wrong: two of three members report normal latency, but the third shows a much higher `RAFT TERM` count than the others, meaning it has fallen behind and triggered several leader elections in the last hour. `etcdctl endpoint health` confirms that member is intermittently failing its health check.

3. **Correlating the timing** — cross-referencing the timestamps of the leader elections against the audit log (already flowing to the central log system per section 13.1's setup) reveals a burst of `create` events against the `Event` resource, hundreds per minute, all from a single ServiceAccount belonging to another team's newly-deployed custom controller. That controller is watching a CRD and, due to a bug in its reconcile loop, writing a Kubernetes `Event` object on every single reconciliation pass — including reconciliations that didn't actually change anything. Each Event write is a full round-trip through etcd's Raft consensus, and the write volume was enough to push disk fsync latency past etcd's election timeout on the node with the slowest disk.

4. **Immediate fix** — the engineer contacts the owning team and has them scale the controller's replica count to zero, immediately relieving the write pressure. Within two minutes, `etcdctl endpoint status` shows all three members back to normal latency and no further leader elections; API server response times return to baseline.

5. **Longer-term fix** — the controller's bug (writing an Event on every reconcile instead of only on state change) gets a tracked ticket and a proper code fix with rate-limited event recording. But the platform engineer doesn't stop there: they add a Prometheus alerting rule on `etcd_server_leader_changes_seen_total` (page if more than N leader elections occur in a 10-minute window) so this class of problem pages someone *before* it degrades the whole cluster next time, and they add an audit-log-based query — the same shape used to find the culprit here — as a saved dashboard panel showing top API request volume by ServiceAccount, so an unusually chatty client is visible at a glance instead of requiring a live forensic investigation to discover.

The incident is closed with three outcomes: the immediate fix (scale down the bad controller), the root-cause fix (a ticket for the actual code bug), and a detection improvement (an alert plus a standing dashboard) — so the same failure mode, if it recurs from a different team's controller next month, is caught in minutes instead of requiring another 2 AM page and a from-scratch investigation.

---

## Further Reading

- [Kubernetes Documentation — Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
- [Kubernetes Documentation — Debugging Running Pods (Ephemeral Containers)](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes Documentation — Debugging Kubernetes Nodes with `kubectl debug`](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/)
- [`crictl` Documentation](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [etcd Documentation — `etcdctl`](https://etcd.io/docs/latest/dev-guide/interacting_v3/)
- [CoreDNS Documentation — Scaling and Performance](https://coredns.io/manual/scaling/)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./12-autoscaling-and-performance-tuning.md">← Previous: Autoscaling and Performance Tuning</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./14-best-practices.md">Next: Best Practices →</a>
</div>
