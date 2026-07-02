# Chapter 16 — Common Mistakes and Pitfalls

## Learning Objectives

By the end of this chapter, you will be able to:

- Identify the most common Kubernetes mistakes by recognizing their symptoms
- Understand the misunderstanding that leads to each mistake and its real-world impact
- Apply the correct manifest pattern immediately, without having to look it up
- Use `kubectl` diagnostic commands to recover a broken workload in a live cluster

---

## How to Read This Chapter

Each mistake is presented with four parts:

1. **The wrong pattern** — a manifest or command you will encounter in the wild
2. **Why it happens** — the misunderstanding that leads to it
3. **The correct fix** — a drop-in replacement
4. **Impact / Prevention** — what breaks in production, and how to stop it happening again

Read this chapter after Chapter 15 (Best Practices) — best practices tell you what to do, this chapter shows you what happens when you don't.

---

## Mistake 1: No Resource Requests or Limits

```yaml
# WRONG — no requests, no limits
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: myorg/api:1.4.0
          # no resources block at all
```

**Why it happens:** A Pod without a `resources` block still schedules and runs fine in a dev cluster with plenty of spare capacity. The problem is invisible until the cluster is under real load.

**The correct fix:**

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

**Impact:** A Pod with no requests/limits gets the `BestEffort` QoS class — the first thing killed when a node runs low on memory, even if it's your most important service. Worse, without limits a single leaky container can consume an entire node's CPU or memory, starving every other Pod scheduled there (the "noisy neighbor" problem). Without requests, the scheduler has no idea how much room the Pod actually needs, so it can over-pack nodes and trigger cascading evictions under pressure.

**Prevention:** Enforce a `LimitRange` per namespace so Pods without explicit resources get sane defaults, and add an admission policy (OPA/Gatekeeper, Kyverno, or a CI manifest linter) that rejects any Deployment missing `resources.requests` and `resources.limits`.

---

## Mistake 2: No Liveness or Readiness Probes

```yaml
# WRONG — Kubernetes has no way to know if this container is actually healthy
containers:
  - name: api
    image: myorg/api:1.4.0
    ports:
      - containerPort: 8080
```

**Why it happens:** The container starts, the process is running, `kubectl get pods` shows `Running` — it looks healthy. Without probes, Kubernetes assumes "process alive" means "ready to serve traffic," which is often false (the app may still be warming a cache or connecting to a DB).

**The correct fix:**

```yaml
containers:
  - name: api
    image: myorg/api:1.4.0
    ports:
      - containerPort: 8080
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

**Impact:** Without a readiness probe, a new Pod is added to the Service's Endpoints the moment its process starts — even if it isn't ready to accept connections yet — so users hit connection errors during every rollout. Without a liveness probe, a Pod that has deadlocked or hung (process alive, but stuck) is never restarted; it silently serves errors or times out forever because `Running` status alone tells Kubernetes nothing about application health.

**Prevention:** Treat probes as mandatory in every Deployment template, same as `resources`. Add a dedicated lightweight `/healthz` endpoint to every service that doesn't do expensive work (no DB calls in the liveness check — that causes cascading restarts when the DB is briefly slow).

---

## Mistake 3: Using `:latest` or No Explicit Tag

```yaml
# WRONG
containers:
  - name: api
    image: myorg/api:latest
  # or worse, no tag at all — Docker/K8s defaults to :latest silently
```

**Why it happens:** `:latest` is what every "hello world" tutorial uses. It seems convenient — no version bumps to think about.

**The correct fix:**

```yaml
containers:
  - name: api
    image: myorg/api:1.4.0
    # even better: pin by digest for full immutability
    # image: myorg/api@sha256:3b8b9b...
```

**Impact:** When a Pod is recreated (node failure, eviction, manual delete, HPA scale-out), Kubernetes re-pulls the image reference exactly as written. If that reference is `:latest`, the new Pod could pull a completely different image than its siblings — pushed to the registry at any point since the last deploy — with no Deployment change, no rollout history entry, and no way to `kubectl rollout undo` back to the old version because Kubernetes never recorded a change happened. You get version skew across replicas of the "same" Deployment and an untraceable production incident.

**Prevention:** Pin every image to an explicit, immutable tag (ideally the CI build's git SHA or semantic version), and set `imagePullPolicy: IfNotPresent` for pinned tags in production so a stale local cache doesn't accidentally get bypassed unpredictably. Add a CI check that rejects manifests referencing `:latest`.

---

## Mistake 4: Running a Single Replica in "Production"

```yaml
# WRONG
spec:
  replicas: 1
```

**Why it happens:** One replica is enough to prove the app runs. It's easy to forget to scale it up before calling the environment "production."

**The correct fix:**

```yaml
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

**Impact:** With one replica, a rolling update, a node drain, a crash, or even a routine `kubectl delete pod` for debugging causes a complete outage — there's no other Pod to serve traffic while the first one restarts. A single Pod also means zero protection from a single bad node going down.

**Prevention:** Never ship a production Deployment with `replicas: 1`. Combine with a `PodDisruptionBudget` (Mistake 8) and pod anti-affinity so replicas land on different nodes.

---

## Mistake 5: Deploying Bare Pods Instead of Deployments

```yaml
# WRONG — a standalone Pod with no controller behind it
apiVersion: v1
kind: Pod
metadata:
  name: api
spec:
  containers:
    - name: api
      image: myorg/api:1.4.0
```

**Why it happens:** A bare Pod is the simplest possible object and is exactly what most tutorials show first, so it feels like the "normal" way to run something.

**The correct fix:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: myorg/api:1.4.0
```

**Impact:** A bare Pod has no self-healing at all. If the node it's scheduled on dies, or the container crashes and exceeds its restart backoff, or someone runs `kubectl delete pod` by mistake, nothing recreates it — it is simply gone. There is no ReplicaSet reconciling desired state back to reality.

**Prevention:** Never write `kind: Pod` directly for application workloads. Use a Deployment (stateless), StatefulSet (stateful), DaemonSet (per-node agent), or Job/CronJob (batch) — every one of these creates and owns Pods for you, with a controller that recreates them automatically.

---

## Mistake 6: ConfigMap/Secret Changes Not Propagating to Running Pods

```yaml
# WRONG — Pod reads a ConfigMap value as an environment variable
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: api-config
        key: log_level
```

You update the ConfigMap: `kubectl edit configmap api-config`. Nothing happens to the running Pods.

**Why it happens:** Environment variables are injected into a container's process **only once, at container start**. Kubernetes has no mechanism to "push" a changed env var into an already-running process — that's how Linux processes work, not a Kubernetes limitation. Volume-mounted ConfigMaps/Secrets do eventually sync to disk, but the application still needs to notice the file changed and reload — most apps don't.

**The correct fix:** Force a rollout whenever the ConfigMap changes, using a checksum annotation so the Pod template hash changes and triggers a real rolling update:

```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: "{{ sha256sum-of-configmap-contents }}"
```

```bash
# Manual equivalent while learning: force a rollout after editing a ConfigMap
kubectl rollout restart deployment/api
```

**Impact:** Teams "fix" a config value, see no error, and assume the fix is live — but every existing Pod is still running with the old value. Only Pods created *after* the edit (e.g., during a future unrelated scale-up) pick up the new value, creating a fleet running a mix of old and new configuration with no obvious explanation.

**Prevention:** Automate the checksum-annotation pattern in your Helm charts or Kustomize overlays so a config change always forces a rollout. Never assume editing a ConfigMap alone is enough.

---

## Mistake 7: Secrets Stored as Plaintext or Hardcoded in Git

```yaml
# WRONG — this is a ConfigMap, not a Secret, committed straight to Git
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_PASSWORD: "SuperSecret123!"
```

**Why it happens:** ConfigMaps and Secrets look and behave almost identically in YAML — the difference (base64 encoding, RBAC treatment) isn't visually obvious, so it's easy to reach for the wrong object out of habit.

**The correct fix:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:            # stringData lets you write plaintext; K8s encodes it for you
  DB_PASSWORD: "SuperSecret123!"
```

...and never commit that file. Use a secrets manager integration instead (Sealed Secrets, External Secrets Operator pulling from AWS Secrets Manager/Vault, or SOPS-encrypted files in Git).

**Impact:** Anyone with `kubectl get configmap -o yaml` access (a much wider RBAC surface than Secret access usually is) sees the password in cleartext. Anyone with Git repo access sees it forever in history, even after a later commit "removes" it. Base64 in a real Secret is not encryption either — it's an encoding — so a Secret committed to Git is exactly as exposed as a ConfigMap.

**Prevention:** Never commit raw Secret manifests to Git at all, encoded or not. Use a GitOps-safe secret pattern (Sealed Secrets, SOPS, External Secrets Operator) and add `gitleaks`/`detect-secrets` as a pre-commit and CI check.

---

## Mistake 8: Ignoring PodDisruptionBudget

```yaml
# WRONG — no PDB exists for a critical 3-replica service
# During a node drain (e.g. cluster upgrade), Kubernetes is free to evict
# all 3 replicas at once if they happen to land on the node(s) being drained
```

**Why it happens:** `PodDisruptionBudget` is an easy object to forget entirely — nothing fails at deploy time without one, and its absence only matters during *voluntary* disruptions (node drains, cluster upgrades, `kubectl drain`), which happen infrequently enough to be forgotten.

**The correct fix:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2          # or: maxUnavailable: 1
  selector:
    matchLabels:
      app: api
```

**Impact:** Without a PDB, `kubectl drain` (routinely run during node upgrades, cluster autoscaler scale-downs, or maintenance) has no constraint telling it how many replicas of your service must stay up. If two of your three replicas happen to sit on the node(s) being drained, Kubernetes evicts both at the same moment — full outage during routine maintenance, not even an incident, just Tuesday's cluster upgrade.

**Prevention:** Add a `PodDisruptionBudget` for every Deployment/StatefulSet that must remain available during voluntary disruptions — this is a five-line YAML file with an outsized safety payoff.

---

## Mistake 9: Overusing `hostPath` Volumes

```yaml
# WRONG
volumes:
  - name: data
    hostPath:
      path: /data/myapp
      type: DirectoryOrCreate
```

**Why it happens:** `hostPath` feels like the simplest possible storage — "just write to a folder on the machine" — especially coming from a single-Docker-host mental model.

**The correct fix:** Use a PersistentVolumeClaim backed by a StorageClass (see Chapter 8) so storage is provisioned dynamically and is not tied to a specific node:

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: myapp-data
```

**Impact:** A Pod using `hostPath` can only run correctly on the specific node where that path has the expected data — if the scheduler places it on a different node (which it will, eventually, after any reschedule), the Pod starts with an empty or wrong directory. This defeats the entire point of a cluster (portability across nodes) and is also a security risk: `hostPath` gives a container direct access to the host filesystem, which is a common container-escape vector if the Pod is ever compromised.

**Prevention:** Reserve `hostPath` for true node-local system tooling (e.g., a log-collecting DaemonSet reading `/var/log`) — never for application data. Use PVCs with a proper StorageClass for anything that needs to persist or move with the Pod.

---

## Mistake 10: Mismatched Service Selector and Pod Labels

```yaml
# WRONG — Service selects "app: api" but the Pods are labeled "app: api-server"
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    metadata:
      labels:
        app: api-server   # doesn't match the Service selector above!
```

**Why it happens:** The Deployment name, the Service name, and the label value all look similar and are easy to type slightly differently without noticing — there's no schema validation that catches a mismatch, because Kubernetes doesn't know your *intent*, only the literal strings.

**The correct fix:** Make the label value used in `spec.selector.matchLabels`, `spec.template.metadata.labels`, and the Service's `spec.selector` all identical:

```yaml
# Service
spec:
  selector:
    app: api
---
# Deployment
spec:
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
```

**Impact:** This is one of the most common silent bugs in Kubernetes. The Deployment reports `3/3 Ready`, the Service exists and has a ClusterIP, DNS resolves the Service name fine — but every request times out, because the Service has **zero Endpoints**. Nothing in `kubectl get svc` reveals this; you have to check Endpoints directly.

```bash
kubectl get endpoints api
# NAME   ENDPOINTS   AGE
# api    <none>      3m      ← the smoking gun
```

**Prevention:** Always define the label value once (e.g., as a Helm template variable or Kustomize common label) and reference it everywhere, rather than typing the string three separate times. When debugging "Service exists but nothing works," check `kubectl get endpoints` before anything else.

---

## Mistake 11: Ignoring SIGTERM / No `terminationGracePeriodSeconds`

```yaml
# WRONG — app has no SIGTERM handler, and default grace period (30s) is
# either too short for in-flight requests to drain, or masks the real problem
containers:
  - name: api
    image: myorg/api:1.4.0
```

**Why it happens:** This is the Kubernetes-layer version of the classic Docker shell-form `CMD` mistake — but even with a correct exec-form entrypoint, most application frameworks do not gracefully handle `SIGTERM` out of the box. Developers test locally with `Ctrl+C` and never notice the app doesn't shut down cleanly, because during a rolling update the traffic-draining step is invisible unless you're watching closely.

**The correct fix:** Handle `SIGTERM` in the application (stop accepting new connections, finish in-flight requests, then exit), add a `preStop` hook to give load balancers time to deregister the Pod before it actually stops receiving traffic, and set an explicit grace period sized to your slowest request:

```yaml
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: api
      image: myorg/api:1.4.0
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]   # let endpoints removal propagate first
```

**Impact:** During every rolling update, scale-down, or node drain, Kubernetes sends `SIGTERM` to the Pod and waits `terminationGracePeriodSeconds` before sending `SIGKILL`. If the app ignores `SIGTERM`, it is hard-killed mid-request every single time — dropped connections, incomplete writes, and client-visible errors on a schedule that correlates exactly with your deploy frequency.

**Prevention:** Treat graceful shutdown as a first-class application requirement, not an infrastructure afterthought. Test it explicitly: `kubectl delete pod <pod>` while sending live traffic and confirm zero failed requests.

---

## Mistake 12: Deleting a Namespace Without Realizing the Blast Radius

```bash
# WRONG — a "quick cleanup" that cascades
kubectl delete namespace staging
```

**Why it happens:** Deleting a namespace feels like deleting one object. It isn't — a Namespace is a container for every resource inside it, and the deletion cascades to all of them.

**The correct fix:** Before deleting a namespace, always inventory what's inside it, and prefer deleting specific resources when that's really the intent:

```bash
kubectl get all,configmap,secret,pvc,ingress -n staging
kubectl delete deployment my-app -n staging   # targeted deletion instead
```

If you genuinely intend to remove the whole namespace, treat it as a destructive production action requiring the same review as a database drop.

**Impact:** `kubectl delete namespace` deletes every Deployment, Service, ConfigMap, Secret, PVC (and, depending on the StorageClass's reclaim policy, the underlying storage volume and its data), Ingress, and RBAC binding inside it — often silently and irreversibly. Teams have deleted the wrong namespace (e.g., "staging" instead of "staging-old") and lost a database's worth of data because the PVC's reclaim policy was `Delete`, not `Retain`.

**Prevention:** Set `reclaimPolicy: Retain` on StorageClasses used for anything with real data (see Chapter 8), require a second approver for any `delete namespace` in CI/CD tooling, and never give broad `delete` RBAC on namespaces to individual contributors in shared clusters.

---

## Mistake 13: `kubectl apply` Stomping Fields Managed by Other Tools

```bash
# WRONG — a stale manifest with `replicas: 3` is re-applied, overwriting the
# live replica count that the HPA had scaled up to 12 under load
kubectl apply -f deployment.yaml
```

**Why it happens:** `kubectl apply` is declarative and computes a three-way merge (last-applied-config, live state, new config) — but if a field like `replicas` is present in your manifest at all, `apply` treats it as something *you* own and want enforced, even though the HPA is the one actually managing it moment to moment.

**The correct fix:** Omit fields from your manifest that another controller owns, or use `kubectl apply --server-side` with field management, which is aware of which controller owns which field:

```yaml
# Deployment manifest — no `replicas` field at all when an HPA manages it
spec:
  # replicas: 3          ← removed; let the HPA own this field entirely
  selector: { ... }
```

```bash
# Server-side apply tracks field ownership explicitly and warns on conflicts
kubectl apply --server-side -f deployment.yaml
```

**Impact:** A well-meaning re-deploy from a slightly stale manifest silently undoes autoscaling — the Deployment snaps back to whatever `replicas` value is in the file, right in the middle of a traffic spike, causing a self-inflicted outage that looks like a capacity problem but is actually a GitOps/manifest hygiene problem. The same class of bug happens with `kubectl edit` (a live, uncommitted change gets silently reverted by the next `apply`) and `kubectl replace` (which fully overwrites the object, dropping anything not in your file).

**Prevention:** Know the difference: `kubectl edit` changes live state only (until the next apply overwrites it); `kubectl apply` does a smart merge against your last applied config (dangerous for fields owned elsewhere); `kubectl replace` fully replaces the object (drops everything not explicitly specified). Never let a manifest and an autoscaler both claim ownership of `replicas` — pick one source of truth.

---

## Mistake 14: Treating `Pending` Pods as a Mystery

```bash
# WRONG — staring at kubectl get pods, which gives almost no information
kubectl get pods
# NAME          READY   STATUS    RESTARTS   AGE
# api-7d9f8-x   0/1     Pending   0          10m
```

**Why it happens:** `kubectl get pods` is the first command everyone learns, and its output for a `Pending` Pod tells you nothing about *why* — no error message, just a status word. Beginners assume something is silently broken rather than knowing where to look next.

**The correct fix:** `kubectl describe pod` surfaces the actual scheduling decision in its `Events` section:

```bash
kubectl describe pod api-7d9f8-x
# ...
# Events:
#   Type     Reason            Message
#   ----     ------            -------
#   Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient memory.
```

**Impact:** Common root causes hiding behind an uninformative `Pending` status include: insufficient CPU/memory across all nodes for the Pod's requests, no node tolerates the Pod's required taints, a `nodeSelector`/affinity rule that matches no node, and — very commonly — a PVC that is itself stuck `Pending` because no StorageClass can satisfy it, which blocks the Pod that mounts it from ever being scheduled.

**Prevention:** Make `kubectl describe pod` (specifically the `Events` section) the automatic first move for any non-`Running` status, before assuming anything is a "mystery." Pair it with `kubectl get events --sort-by=.lastTimestamp` for a cluster-wide timeline when multiple things are failing at once.

---

## Emergency Recovery Commands

```bash
# Pod stuck Pending / CrashLooping / anything unclear — always start here
kubectl describe pod <pod-name>
# Read the Events section at the bottom — it names the exact reason

# Container crashed — see the logs from the PREVIOUS crashed instance,
# not the fresh restart (which may have no output yet)
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c <container-name> --previous   # multi-container pod

# Cluster-wide timeline of what happened recently, oldest first
kubectl get events --sort-by=.lastTimestamp -A

# Live resource usage (requires metrics-server installed in the cluster)
kubectl top pod -A
kubectl top node

# Drop into a running container to poke around directly
kubectl exec -it <pod-name> -- sh
kubectl exec -it <pod-name> -c <container-name> -- sh   # multi-container pod

# Full raw status — everything kubectl knows about the object
kubectl get pod <pod-name> -o yaml

# ImagePullBackOff — almost always one of:
#   - typo in image name/tag
#   - private registry with no/expired imagePullSecrets
#   - rate-limited by the registry (e.g. Docker Hub anonymous pulls)
kubectl describe pod <pod-name> | grep -A5 Events

# CrashLoopBackOff — the container starts and exits repeatedly. Check:
#   1. What exit code? (0 = clean exit but process ended; 1 = app error; 137 = OOMKilled)
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
#   2. What did it log right before dying?
kubectl logs <pod-name> --previous
#   3. Is it a bad readiness/liveness probe killing an otherwise-healthy app?
kubectl describe pod <pod-name> | grep -A3 Liveness

# Rollout appears stuck — see exactly which replicas are the problem
kubectl rollout status deployment/<name>
kubectl get rs -l app=<name>          # look for an old ReplicaSet still scaling down
```

---

## Summary

| # | Mistake | Key Fix |
|---|---------|---------|
| 1 | No resource requests/limits | Always set `resources.requests` and `resources.limits` |
| 2 | No liveness/readiness probes | Add both probes with a dedicated `/healthz` endpoint |
| 3 | `:latest` tag / no tag | Pin to an explicit, immutable tag or digest |
| 4 | Single replica in production | `replicas: 3`+ with a rolling update strategy |
| 5 | Bare Pods, no controller | Always use Deployment/StatefulSet/DaemonSet/Job |
| 6 | ConfigMap changes not propagating | Checksum annotation to force a rollout |
| 7 | Secrets as plaintext ConfigMaps | Real `Secret` objects + external secrets manager |
| 8 | No PodDisruptionBudget | `minAvailable`/`maxUnavailable` PDB per critical service |
| 9 | Overusing `hostPath` | PVC + StorageClass instead |
| 10 | Selector/label mismatch | Single source of truth for label values; check `kubectl get endpoints` |
| 11 | Ignoring SIGTERM | Handle SIGTERM, add `preStop`, size `terminationGracePeriodSeconds` |
| 12 | Deleting a Namespace | Inventory first; target specific resources; `Retain` reclaim policy |
| 13 | `apply` stomping other fields | Omit externally-managed fields; use server-side apply |
| 14 | `Pending` treated as a mystery | `kubectl describe pod` Events section, every time |

---

## Knowledge Check

1. Why does editing a ConfigMap not update the environment variables of a Pod that is already running, and what is the standard pattern to force the update?
2. A Deployment shows `3/3 Ready` and its Service has a valid ClusterIP, but every request times out. What is the very first command you should run, and what would confirm the root cause?
3. Why is `:latest` dangerous specifically in Kubernetes, beyond the general Docker version-drift risk covered in the Docker course?
4. What is the difference in blast radius between `kubectl delete deployment my-app` and `kubectl delete namespace my-namespace`?
5. An HPA has scaled a Deployment to 10 replicas, but a CI pipeline just ran `kubectl apply -f deployment.yaml` and it dropped back to 3. What field ownership problem caused this, and how do you prevent it?

---

## Hands-On Exercise

**Fix the Deliberately Broken Deployment**

The manifest below contains at least 6 of the mistakes covered in this chapter. Find every one of them, explain why it is harmful, and rewrite the file correctly.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: web
          image: myorg/web:latest
          ports:
            - containerPort: 3000
          env:
            - name: DB_PASSWORD
              value: "hunter2"
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 3000
```

Steps:

1. List every mistake you can find (aim for at least 6). Hint: check the labels against the selectors very carefully, and look at what's injected via `env`.
2. Rewrite the Deployment: pin the image tag, scale to at least 3 replicas, fix the label mismatch, add resource requests/limits, add liveness and readiness probes, move the secret into a real `Secret` object, and add a `terminationGracePeriodSeconds` appropriate for a web app.
3. Add a `PodDisruptionBudget` for the corrected Deployment.
4. Apply the fixed manifests to a local `kind` cluster and confirm with `kubectl get endpoints web` that the Service now has real Endpoints listed (it had none before your fix).
5. Delete one Pod manually and confirm a replacement is created automatically and the Service never drops below 2 healthy Endpoints.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./17-projects.md">Next: Hands-On Projects →</a>
</div>
