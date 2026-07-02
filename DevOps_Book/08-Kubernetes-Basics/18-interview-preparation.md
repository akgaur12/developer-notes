# Chapter 18 — Interview Preparation

**Learning Objectives**

By the end of this chapter you will be able to confidently answer foundational, architectural, scenario-based, and system-design Kubernetes interview questions, and articulate your own hands-on experience in a structured way.

---

## 18.1 Foundational Questions

**Q: What is a Pod, and why not just run containers directly?**
> A Pod is the smallest deployable unit in Kubernetes — one or more containers that share network namespace (same IP, same port space, reachable via `localhost` between each other) and can share storage volumes. Kubernetes never schedules a bare container; it always schedules a Pod, because the platform needs an atomic unit for scheduling, networking, and lifecycle decisions, and a Pod is that unit even when it wraps only a single container. Multi-container Pods exist for tightly-coupled helper patterns (sidecar, ambassador, adapter) that must live and die together with the main container.

**Q: What is the difference between a Deployment and a ReplicaSet?**
> A ReplicaSet's only job is to ensure a specified number of identical Pod replicas are running at all times — it has no concept of versioned rollout history. A Deployment sits on top of ReplicaSets and manages them: it creates a new ReplicaSet on every template change, scales the new one up and the old one down (a rolling update), and keeps old ReplicaSets around (scaled to zero) so `kubectl rollout undo` can instantly roll back. In practice you almost never create a ReplicaSet directly — you create a Deployment, and it creates and owns ReplicaSets for you.

**Q: What's the difference between a ConfigMap and a Secret?**
> Both store key-value configuration data and can be consumed as environment variables or mounted as files. The difference is intent and handling: Secrets are base64-encoded (not encrypted by default — encoding, not encryption), Kubernetes avoids printing their values in `kubectl get`/`describe` output by default, they can be encrypted at rest in etcd if the cluster is configured for it, and RBAC commonly restricts access to Secrets more tightly than ConfigMaps. ConfigMaps are for non-sensitive configuration (log levels, feature flags, URLs); Secrets are for credentials, tokens, and keys — though real production secrets should come from an external secrets manager, not raw Kubernetes Secret YAML checked into Git.

**Q: Explain liveness vs readiness probes.**
> A liveness probe answers "is this container still healthy, or should it be restarted?" — if it fails past the failure threshold, the kubelet kills and restarts the container. A readiness probe answers "is this container ready to receive traffic right now?" — if it fails, the Pod is removed from the Service's Endpoints (traffic stops flowing to it) but the container is *not* restarted; it's just temporarily taken out of rotation until it passes again. A Pod can be `Running` and failing readiness at the same time — for example, while it's still warming a cache — without ever being restarted, because being unready and being unhealthy are different problems needing different responses. There's also a startup probe, which delays the other two from running until a slow-starting app finishes initializing.

**Q: What is a Service and how does it find its Pods?**
> A Service is a stable virtual IP and DNS name that load-balances traffic across a dynamic, changing set of Pods. It finds its Pods via a label selector: any Pod whose labels match the Service's `spec.selector` is automatically added to the Service's Endpoints (or EndpointSlices in newer clusters). This is why Service discovery in Kubernetes needs no service registry — the control plane continuously reconciles Endpoints from live label matches, so as Pods come and go (scaling, rolling updates, crashes), the Service's target list updates automatically.

**Q: What happens when you run `kubectl apply -f deployment.yaml`?**
> `kubectl` sends the manifest to the API server, which validates it against the OpenAPI schema, and (on `apply` specifically) computes a three-way merge between the last-applied-configuration annotation, the live object state, and the new manifest, then patches only the fields that changed. The API server persists the desired state to etcd. The Deployment controller notices the new desired state and, if the Pod template changed, creates a new ReplicaSet and begins a rolling update — scaling the new ReplicaSet up and the old one down according to `maxSurge`/`maxUnavailable` — while the scheduler places any new Pods onto nodes with sufficient capacity.

**Q: What is the difference between `kubectl delete`+recreate and `kubectl apply` for updates?**
> `kubectl apply` performs an in-place, declarative update — Kubernetes computes the diff and only changes what's different, and for a Deployment this triggers a controlled rolling update with zero downtime (old Pods stay serving traffic until new ones are ready). `kubectl delete` followed by `kubectl create` (or a resource that requires immutable-field changes) destroys the object first — for a Deployment this means all Pods are torn down before new ones exist, causing an outage. Some fields on some resources genuinely are immutable (e.g., a StatefulSet's `serviceName`), which occasionally forces a delete+recreate — but it should never be the default way to "update" something.

**Q: What is a StatefulSet used for that a Deployment can't do?**
> A StatefulSet provides three things a Deployment does not: stable, predictable Pod names (`postgres-0`, `postgres-1`, not random suffixes), stable per-Pod storage (each replica gets its own PersistentVolumeClaim that follows it across rescheduling, rather than a shared or ephemeral volume), and ordered, sequential deployment/scaling/termination (Pod 0 is created and becomes ready before Pod 1 starts, and terminated last during scale-down). This matters for anything where replica identity matters — databases, distributed systems with leader election (Kafka, Elasticsearch, etcd itself) — where "any interchangeable replica" (the Deployment model) is the wrong mental model.

**Q: What is the difference between a Job and a CronJob?**
> A Job runs a Pod (or several, per `completions`/`parallelism`) to completion exactly once and tracks success/failure — it's for one-off or triggered batch work like a database migration. A CronJob wraps a Job template with a cron schedule, creating a new Job on each scheduled tick — it's for recurring work like nightly backups or periodic cleanup tasks. Neither is meant for long-running services; that's what a Deployment is for.

---

## 18.2 Architecture and Internals Questions

**Q: Explain the Kubernetes control plane components.**
> The **API server** (`kube-apiserver`) is the single entry point — every read and write, including from other control plane components, goes through it, and it validates and persists objects to **etcd**, a distributed key-value store that is the cluster's single source of truth. The **scheduler** (`kube-scheduler`) watches for Pods with no assigned node and decides placement based on resource requests, affinity/anti-affinity, taints/tolerations, and other constraints. The **controller manager** (`kube-controller-manager`) runs the reconciliation loops (Deployment controller, ReplicaSet controller, Node controller, etc.) that continuously compare desired state to actual state and act to close any gap. On managed cloud offerings (EKS, GKE, AKS) these components are run and maintained by the provider — you only see the API server endpoint.

**Q: What is etcd, and why does it need quorum?**
> etcd is a distributed, consistent key-value store using the Raft consensus algorithm, and it holds every object in the cluster — every Pod spec, Secret, ConfigMap, and their current status. It needs quorum (a majority of its members, typically running as an odd-numbered cluster of 3 or 5) to accept writes because Raft requires majority agreement before committing an entry, which is what guarantees the cluster never has two nodes disagreeing about what the "true" state is, even during a network partition. Lose quorum and etcd — and therefore the entire control plane's ability to accept changes — goes read-only or unavailable, even though already-running Pods on nodes keep running.

**Q: What does `kube-scheduler` actually do, step by step?**
> For each unscheduled Pod, the scheduler runs a two-phase process: **filtering**, which eliminates every node that can't run the Pod at all (insufficient resources for the Pod's requests, a taint the Pod doesn't tolerate, a `nodeSelector`/affinity rule that doesn't match, port conflicts), and **scoring**, which ranks the remaining feasible nodes using a set of priority functions (spreading Pods across nodes/zones, bin-packing to consolidate load, honoring pod anti-affinity preferences) and picks the highest-scoring node. The scheduler only makes the *decision* — it writes the chosen node into the Pod's `spec.nodeName`; the kubelet on that node is what actually pulls the image and starts the container.

**Q: What is the CNI, and why does Kubernetes need it?**
> CNI (Container Network Interface) is the plugin specification Kubernetes uses to actually wire up Pod networking — assigning each Pod a unique IP, setting up routes so any Pod can reach any other Pod across any node without NAT, and implementing NetworkPolicy enforcement where supported. Kubernetes itself defines the networking *model* (flat, routable Pod network) but deliberately does not implement it — that's delegated to a CNI plugin (Calico, Cilium, Flannel, the AWS VPC CNI on EKS, etc.), which is why choosing a CNI plugin is one of the first decisions made when standing up any cluster, and why cloud providers pick sensible defaults for their managed offerings.

**Q: What happens if the API server goes down temporarily — does the cluster keep running existing workloads?**
> Yes, mostly. Already-running Pods keep running on their nodes — the kubelet on each node has already received its Pod specs and continues managing those containers locally, restarting them on crash per their restart policy, independent of the API server being reachable. What breaks during an API server outage is anything that requires *new* decisions: no new Pods can be scheduled, no `kubectl` command works, controllers can't reconcile drift (so a node failure during the outage won't trigger replacement Pods), and Services relying on freshly updated Endpoints may serve stale routing until the API server returns. This is precisely why production clusters run the API server (and etcd) as a highly available, multi-replica control plane rather than a single instance.

**Q: What is the reconciliation loop / controller pattern, and why is it central to how Kubernetes works?**
> Every Kubernetes controller follows the same loop: observe the current state of the cluster (via watches on the API server), compare it to the desired state stored in etcd, and take action to move current state toward desired state — then repeat forever. This is why Kubernetes is described as declarative rather than imperative: you never tell it "start 3 Pods," you tell it "I want 3 Pods to exist," and a controller continuously makes that true regardless of what fails in between. It's also why the system self-heals: a node dying doesn't need special-cased failure handling, because the ReplicaSet controller's next reconciliation pass simply notices "3 desired, 2 actual" and creates one more, the same logic it always runs.

---

## 18.3 Scenario-Based Questions

**Scenario 1: "A Pod is stuck in `Pending` and never schedules"**
```
1. kubectl describe pod <pod>
   → read the Events section — it names the exact reason, e.g.:
     "0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had taint..."

2. If insufficient resources:
   kubectl top nodes                 # is the cluster actually full?
   kubectl describe nodes | grep -A5 "Allocated resources"

3. If taint-related:
   kubectl describe node <node> | grep Taints
   # does the Pod have a matching toleration, or should it target a different node?

4. If affinity/nodeSelector related:
   kubectl get pod <pod> -o yaml | grep -A10 affinity
   kubectl get nodes --show-labels    # do any nodes actually have the required label?

5. If storage-related (Pod mounts a PVC):
   kubectl get pvc
   # if the PVC itself is Pending, the Pod can never schedule — fix the PVC first
   kubectl describe pvc <pvc-name>    # no matching StorageClass/PV, or provisioner stuck
```

**Scenario 2: "A Pod is stuck in `CrashLoopBackOff`"**
```
1. Check the exit code and reason from the last crash:
   kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
   # exitCode 0   → process exited cleanly but shouldn't have (bad entrypoint/command)
   # exitCode 1   → application-level error
   # exitCode 137 → SIGKILL, usually OOMKilled — check .lastState.terminated.reason

2. Read the logs from the crashed instance, not the fresh restart:
   kubectl logs <pod> --previous

3. If OOMKilled: check requests/limits vs actual usage
   kubectl describe pod <pod> | grep -A3 Limits
   kubectl top pod <pod>              # if it's even up long enough to sample

4. If it's a bad liveness probe killing an otherwise-healthy slow-starting app:
   kubectl describe pod <pod> | grep -A5 Liveness
   # consider adding a startupProbe to delay liveness checks during boot

5. Reproduce interactively if logs aren't enough:
   kubectl run debug --rm -it --image=<same-image> --command -- sh
   # manually run the entrypoint command and watch what actually fails
```

**Scenario 3: "A Pod is stuck in `ImagePullBackOff`"**
```
1. kubectl describe pod <pod>
   → Events will show the exact pull error string

2. Common causes, in order of likelihood:
   a. Typo in image name or tag        → verify against the registry directly
   b. Private registry, no credentials  → check imagePullSecrets is set and valid
      kubectl get secret <pull-secret> -o yaml
      kubectl describe sa default -n <namespace>   # is the secret attached?
   c. Expired registry credentials      → re-create the imagePullSecret
   d. Registry rate limiting             → e.g. Docker Hub anonymous pull limits;
      switch to an authenticated pull or a mirror
   e. Wrong architecture (arm64 image on an amd64 node or vice versa)

3. Test the pull manually from a node or a throwaway Pod to isolate
   cluster-networking issues from credential issues:
   kubectl run pulltest --rm -it --image=<the-exact-image:tag> --command -- true
```

**Scenario 4: "Service exists but I can't reach my app"**
```
1. Confirm the Service actually has live targets — this is the #1 root cause:
   kubectl get endpoints <service-name>
   # empty output ("<none>") means the Service selector doesn't match any Pod's labels

2. If Endpoints are empty:
   kubectl get pods --show-labels
   kubectl get svc <service-name> -o yaml | grep -A3 selector
   # compare the two by eye — this is almost always a label typo

3. If Endpoints ARE populated but requests still fail:
   kubectl run debug --rm -it --image=busybox --command -- \
     wget -qO- http://<service-name>.<namespace>.svc.cluster.local:<port>
   # tests DNS + routing from inside the cluster, ruling out client-side issues

4. Check the container is actually listening on the port the Service targets:
   kubectl exec -it <pod> -- ss -tlnp
   # targetPort in the Service must match the port the app binds inside the container

5. If it's an Ingress (not just a Service) that's unreachable:
   kubectl describe ingress <name>       # check the backend resolved correctly
   kubectl get pods -n ingress-nginx     # is the controller itself healthy?
```

**Scenario 5: "A rolling update is stuck and won't complete"**
```
1. Check overall rollout status:
   kubectl rollout status deployment/<name>
   # "Waiting for deployment ... to progress" hanging = new Pods aren't becoming Ready

2. Find the new ReplicaSet and inspect its Pods directly:
   kubectl get rs -l app=<name>
   kubectl describe pod <new-pod>
   # most common cause: new Pods fail their readiness probe and never count as
   # "available," so the rollout can never proceed past maxUnavailable

3. Check if maxSurge/maxUnavailable math is blocking progress on a small
   replica count (e.g. replicas=1 with defaults can't roll at all):
   kubectl get deployment <name> -o yaml | grep -A3 rollingUpdate

4. Check for a resource crunch preventing new Pods from even scheduling:
   kubectl get pods -l app=<name>    # any stuck Pending alongside the CrashLooping ones?

5. If the new version is simply broken, don't wait — roll back immediately:
   kubectl rollout undo deployment/<name>
   kubectl rollout history deployment/<name>   # to pick a specific earlier revision
```

**Scenario 6: "A node suddenly goes `NotReady` — what do you check, and what happens to its Pods?"**
```
1. Confirm the node's condition and the reason:
   kubectl get nodes
   kubectl describe node <node>
   # look at the Conditions block: MemoryPressure, DiskPressure, PIDPressure,
   # NetworkUnavailable, and Ready — each has a Status and a human-readable Reason

2. Check kubelet health on the node itself if you have SSH/SSM access:
   systemctl status kubelet
   journalctl -u kubelet -n 100 --no-pager

3. Understand the timeline Kubernetes follows automatically:
   - Node stops sending heartbeats to the API server
   - After `node-monitor-grace-period` (default 40s), the node is marked NotReady
   - After `pod-eviction-timeout` (default 5 minutes), the control plane starts
     evicting Pods scheduled on that node so their ReplicaSets can recreate them
     elsewhere — but only for Pods managed by a controller; bare Pods are not
     rescheduled, another reason to avoid Mistake 5 from Chapter 16

4. If it's a cloud-managed node group, check whether the cloud provider's node
   health checks / Cluster Autoscaler have already flagged it for replacement

5. Decide: wait for auto-recovery, or manually drain and investigate:
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   kubectl cordon <node>     # prevent new Pods without evicting existing ones
```

**Scenario 7: "You need to safely upgrade cluster nodes with zero application downtime"**
```
1. Confirm PodDisruptionBudgets exist for every critical multi-replica service
   (Chapter 16, Mistake 8) — without them, drain can evict too many replicas at once:
   kubectl get pdb -A

2. Cordon the target node so the scheduler stops placing new Pods on it:
   kubectl cordon <node>

3. Drain it, respecting PDBs (this call blocks/retries if a PDB would be violated):
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

4. Confirm evicted Pods rescheduled successfully elsewhere and are Ready:
   kubectl get pods -A -o wide --field-selector spec.nodeName!=<node>

5. Perform the node upgrade/replacement (OS patch, new AMI, k8s version bump)

6. Uncordon so the node can receive Pods again, or delete it if it's being
   replaced entirely by a fresh node from the node group:
   kubectl uncordon <node>

7. Repeat one node at a time — never drain multiple nodes backing the same
   service simultaneously unless your PDBs and replica count can tolerate it
```

---

## 18.4 System Design Questions

**"Design a scalable, resilient deployment for a web application on Kubernetes."**

Key points to cover:
```
1. Deployment with 3+ replicas, spread across nodes/zones via podAntiAffinity
2. Resource requests/limits sized from real load testing, not guesses
3. Readiness + liveness probes (+ startup probe if boot is slow)
4. HorizontalPodAutoscaler targeting CPU/memory or a custom metric (request rate)
5. PodDisruptionBudget so voluntary disruptions (node drains) can't take it all down
6. Rolling update strategy (maxSurge/maxUnavailable) for zero-downtime deploys
7. ConfigMap/Secret for configuration, sourced from an external secrets manager
8. Service (ClusterIP) + Ingress with TLS termination via cert-manager
9. PersistentVolume only if the app is stateful; otherwise keep it fully stateless
   so any replica can be killed and replaced without data loss
10. ResourceQuota at the namespace level so this app can't starve its neighbors
11. Observability: liveness/readiness alone aren't enough — plan for metrics
    (Prometheus) and centralized logs, covered in Topic 10
```

**"How would you handle secrets and config across dev/staging/prod environments in Kubernetes?"**
```
1. Same base manifests/Helm chart across all environments — only values differ
   (Kustomize overlays or per-environment values-{env}.yaml files)
2. Never commit raw Secret manifests to Git, in any environment — including dev
3. Use an external secrets manager (AWS Secrets Manager, HashiCorp Vault) with the
   External Secrets Operator or Sealed Secrets to sync into Kubernetes Secrets
4. Separate namespaces (or separate clusters) per environment, so RBAC can
   restrict who can read prod Secrets vs dev Secrets independently
5. ConfigMaps for non-sensitive, environment-specific values (API URLs, feature
   flags, log levels) layered via Kustomize patches or Helm values
6. CI/CD pipeline uses environment-scoped credentials (OIDC roles) to deploy —
   never a developer's personal kubeconfig for staging/prod
7. Rotate secrets on a schedule and on personnel changes; the secrets manager
   integration means rotation doesn't require a manifest change or redeploy
```

**"How would you migrate a stateful application (e.g., a database) into Kubernetes with minimal risk?"**
```
1. Start read-only: mirror data into a StatefulSet-managed Postgres/MySQL
   instance running alongside the existing database, using native replication
   or a change-data-capture tool — don't cut over yet

2. Choose the right storage: a PVC backed by a StorageClass with a reclaim
   policy of Retain (never Delete) for anything holding real data (Chapter 8,
   revisited in Chapter 16's Namespace-deletion mistake)

3. Validate durability before trusting it: kill the StatefulSet's Pod
   deliberately and confirm the PVC reattaches with zero data loss, well
   before any real cutover

4. Take the migration in stages: dual-write or replicate first, verify
   consistency, then flip the application's connection string in a
   maintenance window, then decommission the old instance only after a
   rollback window has passed with no issues

5. Back up before, during, and after — a CronJob-based backup (Chapter 12 and
   Project 3 in Chapter 17) should already be running before cutover, not
   added afterward

6. Set resource requests/limits and probes on the database Pod from day one —
   an under-resourced database StatefulSet is a worse outage than an
   under-resourced stateless API, because recovery isn't just "restart it"
```

---

## 18.5 Quick-Fire Questions

| Question | Answer |
|----------|--------|
| Default number of replicas if `replicas` is omitted? | 1 |
| What object type provides stable network identity for Pods? | StatefulSet |
| What command shows why a Pod is Pending? | `kubectl describe pod` |
| What's the difference between `ClusterIP` and `NodePort`? | ClusterIP is internal-only; NodePort also opens a static port on every node |
| What field selects which Pods a Service load-balances to? | `spec.selector` (label match) |
| What happens to a Pod's data on restart with no volume? | Lost — container filesystem is ephemeral |
| What is `kubectl get endpoints` used to check? | Whether a Service actually has any matching, healthy Pods behind it |
| What triggers a new ReplicaSet in a Deployment? | Any change to `spec.template` |
| What does `kubectl rollout undo` do? | Reverts to the previous (or a specified) ReplicaSet revision |
| What is a DaemonSet used for? | Running exactly one Pod per node (log agents, CNI, monitoring agents) |
| What does `imagePullPolicy: Always` do? | Re-checks the registry for the tag on every Pod start, even if cached locally |
| What port range does NodePort use by default? | 30000–32767 |

---

## 18.6 "Walk Me Through Your Kubernetes Experience"

STAR format example:

```
Situation: Our team ran a monolithic app on a handful of manually-managed EC2
instances behind a load balancer. Deploys were manual SSH-and-restart, took
20+ minutes, and any instance failure meant a 3 AM page.

Task: Migrate the application to Kubernetes to get automated recovery,
zero-downtime deploys, and horizontal scaling under load.

Action:
1. Containerized the app (building on our existing Docker setup) and wrote
   Deployment/Service manifests with proper resource requests/limits and
   liveness/readiness probes from day one
2. Externalized configuration into ConfigMaps and moved secrets into a
   Secrets-manager-backed integration instead of .env files on disk
3. Set up an Ingress with cert-manager for automatic TLS certificate renewal
4. Added an HPA on the API tier after identifying CPU as the scaling bottleneck
   under load testing
5. Packaged the whole thing as a Helm chart so dev/staging/prod used the
   identical templates with different values files
6. Added a PodDisruptionBudget after a cluster upgrade briefly took down both
   replicas of a service at once — turned a production incident into a
   permanent five-line safeguard

Result: Deploy time dropped from 20+ minutes of manual SSH work to under 3
minutes via a rolling update with zero downtime. A node failure that used to
page someone at 3 AM now self-heals within 30 seconds with no human involved.
The HPA absorbed a 4x traffic spike during a marketing campaign without any
manual intervention.
```

**Self-Check Before Your Interview**

- Can you explain, without notes, what happens end-to-end from `kubectl apply` to a Pod actually receiving traffic?
- Can you describe two different reasons a Service might have zero Endpoints?
- Can you name the exact difference between a liveness and a readiness probe failing?
- Can you talk through a production incident (real or a lab exercise from Chapter 17) using the debugging flow from section 18.3, not just the answer?

No separate hands-on exercise for this chapter — working through the scenarios above out loud, from memory, is the exercise.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./17-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./19-course-summary.md">Next: Course Summary →</a>
</div>
