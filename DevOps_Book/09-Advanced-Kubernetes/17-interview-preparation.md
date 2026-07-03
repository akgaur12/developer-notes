# Chapter 17 — Interview Preparation

**Learning Objectives**

By the end of this chapter you will be able to confidently answer platform-engineer/senior-level questions on RBAC, admission control, extensibility, multi-tenancy, service mesh, GitOps, and cluster operations — spanning foundational concepts, architecture/internals, incident scenarios, and system design — and articulate your own platform engineering experience in a structured way.

---

## 17.1 Foundational Questions

**Q: What's the difference between authentication and authorization in Kubernetes?**
> Authentication answers "who are you?" — the API server verifies the identity of the caller via client certificates, bearer tokens (ServiceAccount tokens, OIDC tokens), or a webhook, and produces a username and group list. Authorization answers "is this identity allowed to do this?" — using that username/group list, an authorization mode (almost always RBAC in modern clusters) checks whether any Role/ClusterRole bound to that identity grants the requested verb on the requested resource. The two are separate, sequential stages of every single API request: a request that fails authentication never reaches authorization at all, and a request that authenticates successfully can still be denied at authorization if no binding grants it the needed permission.

**Q: Explain the difference between a Role and a ClusterRole.**
> A Role is namespace-scoped — its rules only apply, and can only be bound, within the one namespace it's created in. A ClusterRole is cluster-scoped — it can grant access to cluster-scoped resources (Nodes, PersistentVolumes, Namespaces themselves), non-resource URLs (like `/healthz`), *or* it can be bound with a plain RoleBinding to restrict its use to a single namespace even though the ClusterRole itself is reusable across many. This is the pattern behind "define once, bind many times": a platform team defines one `ClusterRole` called `app-developer` with a sensible set of verbs, then creates a `RoleBinding` (not a `ClusterRoleBinding`) in each team's namespace referencing that same ClusterRole — giving every team identical permissions, but each strictly confined to their own namespace.

**Q: What is a Custom Resource Definition and how does it relate to an Operator?**
> A CRD registers a brand-new API type with the Kubernetes API server — after applying one, `kubectl get <yourkind>` works exactly like `kubectl get pods`, with the same validation, storage in etcd, `kubectl apply` support, and RBAC-ability as any built-in type. On its own, a CRD is just a schema — creating a Custom Resource from it stores data but nothing *acts* on it. An Operator is the controller that watches those Custom Resources and reconciles the real world to match them, the same reconciliation-loop pattern from Kubernetes Basics applied to a domain the built-in controllers know nothing about. The CRD is the API; the Operator is the operational knowledge encoded as code — e.g., "how to fail over a Postgres primary safely" — that would otherwise live in a runbook a human executes by hand.

**Q: What problem does a service mesh solve that Ingress + Services don't?**
> Services and Ingress solve "how does traffic get routed to the right Pods," both from outside the cluster (Ingress) and between Pods (Service + kube-proxy/cluster DNS). Neither solves service-to-service *security* (mutual TLS between every Pod without every application implementing it itself), fine-grained traffic control (canary splits by percentage, retries, circuit breaking, timeout policies applied uniformly without app code changes), or deep observability (per-request latency/error/traffic metrics between every pair of services, automatically, without instrumenting each app). A service mesh injects a sidecar proxy next to every Pod that intercepts all in/out traffic, and a control plane configures every sidecar identically — turning those three problems from "something every team's app must implement" into "something the platform provides for free." The tradeoff is real added complexity and latency, which is why Chapter 7 frames the mesh decision as "do you actually have the problem it solves" rather than a default choice.

**Q: What is GitOps and how does it differ from a traditional CI/CD push pipeline?**
> In a traditional push pipeline, CI builds an artifact and then a pipeline job authenticates to the cluster and runs `kubectl apply`/`helm upgrade` directly against it — the cluster is a passive target that credentials from outside are pushed into. In GitOps, a controller (Argo CD or Flux) runs *inside* the cluster, continuously watches a Git repository, and pulls changes to reconcile the cluster to match — no external system ever needs cluster credentials, because the direction of control is reversed. This also makes Git the single source of truth in a way a push pipeline can't guarantee: a push pipeline has no way to detect or correct someone's manual `kubectl edit` in prod, while a GitOps controller's continuous reconciliation loop reverts drift automatically (`selfHeal`), and the entire deployment history becomes Git's commit history rather than being scattered across CI job logs.

**Q: What's the difference between NetworkPolicy default-deny for ingress vs egress?**
> A default-deny **ingress** policy (an empty `podSelector` with `policyTypes: [Ingress]` and no `ingress` rules) blocks all *incoming* traffic to the selected Pods unless an explicit rule allows it — it controls who can reach this Pod. A default-deny **egress** policy does the same for *outgoing* traffic — it controls what this Pod is allowed to initiate connections to, including DNS lookups, which is why a default-deny-egress namespace needs an explicit allow rule for port 53 to `kube-system` or Pods break in a confusing way (DNS resolution silently times out). They're independent controls: a Pod can be locked down on ingress but wide open on egress, or vice versa, and a full zero-trust posture (Chapter 4) requires both default-denies plus explicit allows for every real path in both directions.

**Q: Why would you choose Karpenter over the Cluster Autoscaler?**
> The Cluster Autoscaler scales existing node groups/ASGs up or down by adjusting their desired count — it's constrained to whatever instance types and sizes those pre-defined groups were configured with, so it can't right-size a node to a Pod's actual shape. Karpenter provisions nodes directly against the cloud provider API, choosing an instance type on the fly based on the actual pending Pods' resource requests, so a cluster with wildly different workload shapes (some Pods wanting 16 vCPUs, others wanting 0.5) gets appropriately sized nodes for each rather than being forced into one node-group shape. Karpenter also tends to scale down faster and more aggressively consolidate underutilized nodes. The tradeoff is that Karpenter is AWS/cloud-provider-specific (with growing multi-cloud support) and newer/less universally battle-tested than the Cluster Autoscaler, which works anywhere ASG-like node groups exist.

**Q: What are RTO and RPO, and why do they matter for Kubernetes backup strategy?**
> RTO (Recovery Time Objective) is how long you can tolerate being down before service is restored. RPO (Recovery Point Objective) is how much data you can tolerate losing, measured as time since the last good backup. They matter because they drive concrete, opposite-cost technical decisions: a low RPO (near-zero data loss) demands continuous replication or frequent WAL-shipping backups, which costs more storage/bandwidth than nightly snapshots; a low RTO demands pre-provisioned standby capacity or automated restore tooling (Velero, an Operator's built-in recovery), which costs more than "restore from S3 manually when needed." Without stating RTO/RPO numbers up front, "we have backups" is meaningless — a nightly backup with a 4-hour manual restore process might be completely fine for an internal tool and completely unacceptable for a payments database, and that's a business decision the platform team needs stated explicitly, not assumed.

**Q: What is the difference between soft multi-tenancy and hard multi-tenancy?**
> Soft multi-tenancy shares one cluster's control plane and worker nodes across multiple teams, using Namespaces, RBAC, ResourceQuotas, NetworkPolicies, and Pod Security Standards to create logical isolation — it's cost-efficient and operationally simple but trusts that a kernel-level compromise or a control-plane bug can't cross tenant boundaries, so it's appropriate for teams inside the same trust boundary (departments of one company), not for mutually adversarial or externally-billed tenants. Hard multi-tenancy gives each tenant a dedicated cluster (or at minimum, dedicated nodes with strong sandboxing like gVisor/Kata Containers and separate control planes), trading cost and operational overhead for a much stronger isolation guarantee suitable for hosting untrusted or paying external customers. Most internal platform teams run soft multi-tenancy by default and escalate to hard multi-tenancy only for the specific tenants that actually need it.

---

## 17.2 Architecture and Internals Questions

**Q: Explain the full admission control pipeline, in order.**
> A request that has already passed authentication and authorization does not go straight to etcd — it passes through admission control first, in a fixed order: **mutating admission webhooks** run first (in parallel with each other, results merged), because they can modify the object — injecting a sidecar, adding a label, setting a default resource limit. Then **Pod Security Admission** (or the older PodSecurityPolicy, now removed) evaluates the now-final object against the namespace's enforce level. Then **validating admission webhooks** run last, seeing the fully-mutated object and either allowing or rejecting it outright — this ordering is deliberate, because a validating webhook checking "does this Pod have resource limits" needs to see them *after* any mutating webhook that injects defaults, not before. Only after every admission stage passes does the object actually get persisted to etcd.

**Q: What happens if etcd loses quorum during a control-plane upgrade?**
> etcd requires a majority of its members (2 of 3, or 3 of 5) to agree before committing any write, because that's what Raft consensus requires to guarantee consistency. If quorum is lost mid-upgrade — say, two of three etcd members are being upgraded simultaneously instead of one at a time — the API server can no longer accept *any* write: no new Pods, no updates, no deletions, though already-running Pods keep running because kubelets don't depend on etcd directly. This is exactly why the correct upgrade procedure (Chapter 9) is strictly sequential — upgrade or restart one etcd member at a time, confirm it rejoins and quorum holds, before touching the next — and why production clusters run an odd number of etcd members with a pre-upgrade snapshot (`etcdctl snapshot save`) taken immediately before starting, so a quorum-loss incident has a known-good recovery point rather than requiring cluster rebuild from scratch.

**Q: How does Pod Security Admission actually get enforced?**
> Pod Security Admission is a built-in admission controller (not a separate install, unlike Gatekeeper/Kyverno) that reads labels on the Namespace object itself — `pod-security.kubernetes.io/enforce: baseline` (or `restricted`/`privileged`) — and validates every Pod created in that namespace against the corresponding predefined policy level. Unlike OPA/Kyverno, there's no custom rule authoring: the three levels (`privileged`, `baseline`, `restricted`) are fixed, versioned definitions maintained upstream, and enforcement is purely a namespace label plus whatever the built-in admission controller checks against that Pod's `securityContext` fields (running as root, privileged containers, allowed capabilities, host namespace usage, and so on). The tradeoff for that simplicity is inflexibility — if you need a custom rule Pod Security Admission's three levels don't express (like "images must come from our internal registry"), that's exactly the gap Kyverno or OPA/Gatekeeper fill.

**Q: What does a mutating webhook do differently from a validating webhook, and why does order matter between them?**
> A mutating webhook can change the object it receives before it's persisted — injecting a service mesh sidecar container, adding a default `securityContext`, setting an annotation — and returns a JSON patch describing the change. A validating webhook can only accept or reject the object as-is; it cannot modify it. Order matters because mutating webhooks run before validating webhooks specifically so validation sees the *final*, fully-mutated object: if a mutating webhook's job is to inject default resource limits onto any Pod missing them, and a validating webhook's job is to reject Pods without resource limits, running validation first would reject Pods that the mutating webhook was about to fix, which defeats the entire point of having the mutator. Within each category (multiple mutating webhooks, or multiple validating webhooks), ordering is not guaranteed unless explicitly configured, which is why webhooks that must run in a specific relative order to each other are a design smell worth avoiding.

**Q: How does an Operator actually decide what action to take — walk through one reconcile loop.**
> On every reconcile trigger (an event on the watched Custom Resource, a related owned resource, or a periodic resync), the controller's reconcile function runs: fetch the current CR spec (desired state) and the current state of whatever it manages (e.g., query the actual Postgres cluster's replica count and health), diff the two, and take the minimal action needed to close the gap — scale up a replica, trigger a failover, kick off a backup — then update the CR's `status` subresource to reflect what's now true. Crucially this is the same reconcile function every time, regardless of *why* it was triggered — a human editing the CR, a Pod crashing, or nothing changing at all (a routine resync) — because the controller never trusts that it "remembers" what it did last; it always re-derives the correct action from comparing current state to desired state, which is what makes it resilient to controller restarts and missed events.

**Q: Why is it dangerous to run `kubectl edit` directly against a resource that's managed by Argo CD, and what actually happens?**
> If `selfHeal` is enabled, Argo CD's controller notices the live object no longer matches Git on its next reconcile pass and reverts the edit automatically — the manual change simply disappears, often to the confusion of whoever made it. If `selfHeal` is disabled, the resource is flagged `OutOfSync` but not reverted, silently accumulating drift between what Git says and what's actually running, which defeats the entire purpose of GitOps (Git as source of truth) and eventually causes a nasty surprise when someone finally does sync and unexpectedly reverts weeks of undocumented manual changes. The correct fix in either case is the same: make the change in Git, via a PR, and let Argo CD apply it — `kubectl edit` against a GitOps-managed resource should be treated as a debugging-only, temporary action, never a real fix.

---

## 17.3 Scenario-Based Questions

**Scenario 1: "RBAC is denying an operation that should be allowed — how do you debug it?"**
```
1. Reproduce the exact denial as the exact identity experiencing it:
   kubectl auth can-i <verb> <resource> --as=<user-or-sa> -n <namespace>
   # for a ServiceAccount: --as=system:serviceaccount:<ns>:<sa-name>
   # add --as-group=<group> if the identity relies on group membership

2. List every binding that could plausibly grant this, cluster-wide:
   kubectl get rolebindings,clusterrolebindings -A -o json | \
     jq '.items[] | select(.subjects[]?.name=="<identity>")'

3. For each matching binding, inspect the Role/ClusterRole it actually
   points at — the binding might exist but reference the wrong role,
   or a role that doesn't cover this resource/verb combination:
   kubectl describe role <name> -n <namespace>
   kubectl describe clusterrole <name>

4. Check for a resourceName restriction that narrows the rule to specific
   object names rather than all objects of that kind — a common surprise:
   kubectl get role <name> -n <namespace> -o yaml | grep -A5 resourceNames

5. If using OIDC/group-based auth, confirm the identity's token actually
   carries the expected group claims:
   kubectl auth whoami   # (or decode the token/id-token directly)

6. Remember RBAC is purely additive — there's no explicit "deny" rule to
   search for. If nothing grants it, the fix is adding a rule, not finding
   and removing a conflicting deny.
```

**Scenario 2: "A NetworkPolicy was applied but traffic still isn't blocked"**
```
1. Confirm the CNI plugin actually enforces NetworkPolicy at all — this is
   the single most common root cause. Flannel alone does NOT enforce
   NetworkPolicy; Calico, Cilium, and the AWS VPC CNI (with the network
   policy add-on) do:
   kubectl get pods -n kube-system   # identify the CNI in use
   # check the CNI's own docs/release notes for NetworkPolicy support

2. If the CNI does support it, confirm the policy is actually selecting
   the Pods you think it is:
   kubectl get networkpolicy <name> -n <namespace> -o yaml
   kubectl get pods -n <namespace> --show-labels
   # compare podSelector's matchLabels against the Pods' actual labels by eye

3. Confirm policyTypes includes the direction you're trying to restrict —
   a policy with only `ingress` rules and no `policyTypes: [Egress]` does
   NOT restrict egress at all, even if that seems intuitive:
   kubectl get networkpolicy <name> -o yaml | grep -A2 policyTypes

4. Check for another, more permissive NetworkPolicy in the same namespace
   that also selects these Pods — NetworkPolicies are additive (like RBAC),
   so ANY policy allowing the traffic makes it succeed, regardless of how
   restrictive your other policy is:
   kubectl get networkpolicy -n <namespace>

5. Test directly with an ephemeral Pod carrying an intentionally
   non-matching label, to isolate CNI enforcement from policy-selector bugs:
   kubectl run probe --rm -it --image=busybox -n <namespace> \
     --overrides='{"metadata":{"labels":{"test":"true"}}}' \
     --command -- wget -qO- --timeout=2 http://<target-svc>
```

**Scenario 3: "Argo CD shows a resource as perpetually OutOfSync"**
```
1. Get the actual diff Argo CD is seeing, not just the status badge:
   argocd app diff shop
   # or: kubectl get application shop -n argocd -o jsonpath='{.status.sync}'

2. Common cause #1 — a controller other than Argo CD is mutating the live
   object every time it's synced (e.g., an HPA changing .spec.replicas,
   or a mutating webhook injecting a sidecar/annotation not present in Git):
   kubectl get deployment <name> -o yaml | grep -A2 managedFields
   # look for a field manager other than argocd that's rewriting the field

3. If it's the HPA-vs-replicas conflict specifically, ignore that field in
   the Application's diff config rather than fighting the HPA:
   spec:
     ignoreDifferences:
       - group: apps
         kind: Deployment
         jsonPointers: ["/spec/replicas"]

4. Common cause #2 — someone ran kubectl edit/patch directly against the
   cluster and selfHeal is disabled, so the drift persists indefinitely
   until a manual sync:
   argocd app sync shop
   # then enable selfHeal going forward so this can't recur silently

5. Common cause #3 — a Helm chart or Kustomize overlay renders
   nondeterministic output (e.g., a checksum annotation that changes every
   render even with no real change) — fix the template, not the symptom
```

**Scenario 4: "A cluster upgrade is stuck because a node won't drain"**
```
1. Get the specific blocking reason, not just "it's stuck":
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   # the error message names the exact Pod and constraint blocking eviction

2. Most common cause — a PodDisruptionBudget that can't be satisfied because
   evicting this Pod would drop available replicas below minAvailable:
   kubectl get pdb -A
   kubectl describe pdb <name> -n <namespace>
   # check current healthy replica count vs the PDB's minAvailable/maxUnavailable

3. If the PDB is correctly configured but the Deployment simply doesn't
   have enough replicas to satisfy it during a single-node drain, that's a
   capacity problem, not a PDB bug — temporarily scale up before draining:
   kubectl scale deployment <name> --replicas=<n+1> -n <namespace>

4. Check for a bare Pod (no owning controller) on the node — these are
   never rescheduled and drain will refuse to evict them without
   --force, which then loses them permanently:
   kubectl get pods -o wide --field-selector spec.nodeName=<node> | \
     grep -v -E 'Deployment|ReplicaSet|StatefulSet|DaemonSet|Job'

5. Check for a Pod using local emptyDir data the drain is protecting via
   --delete-emptydir-data being deliberately omitted — confirm data loss
   is actually acceptable before adding that flag

6. As a last resort for a stuck non-controller Pod blocking a critical
   upgrade window, force it — understanding this permanently deletes
   that Pod's local state:
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
```

**Scenario 5: "HPA and VPA seem to be fighting each other"**
```
1. Confirm they're actually both targeting the same workload on the same
   metric — this is the textbook conflict:
   kubectl get hpa,vpa -n <namespace>
   # HPA scaling on CPU utilization + VPA changing CPU requests underneath
   # it means the HPA's target percentage is a moving denominator, causing
   # oscillation (scale out, then in, then out again) as VPA rewrites requests

2. Check VPA's updateMode — "Auto" actively evicts and recreates Pods with
   new resource values, which is the aggressive setting most likely to
   collide with HPA:
   kubectl get vpa <name> -o jsonpath='{.spec.updatePolicy.updateMode}'

3. The supported combination: VPA in "Off" or "Initial" mode (recommends
   or applies only on Pod creation, never live-mutates running Pods) for
   right-sizing requests over time, while HPA independently handles
   replica count — never let both fight over the same axis:
   spec:
     updatePolicy:
       updateMode: "Off"   # VPA only recommends via its status; nothing auto-applies

4. Alternative: split responsibility by metric — VPA manages memory
   (rarely spiky, safe to right-size slowly) while HPA scales on a custom
   request-rate or CPU metric, since scaling replica count for CPU
   pressure is usually more appropriate than resizing the container anyway

5. Watch kubectl get events -n <namespace> for repeated Evicted/back-off
   cycles as direct evidence of the two controllers actively conflicting,
   confirming the diagnosis rather than assuming it
```

**Scenario 6: "You need to recover from an accidental `kubectl delete namespace` in production"**
```
1. Immediately confirm scope — deleting a Namespace cascades and deletes
   EVERY object inside it, including PVCs (and, if the StorageClass's
   reclaim policy is Delete rather than Retain, the underlying cloud
   volumes too — check this BEFORE the drill, not during):
   kubectl get namespace <name>   # likely stuck "Terminating" briefly, then gone

2. Restore from the most recent Velero backup, into either the same
   namespace name (if nothing else recreated it in the meantime) or a
   scratch name to inspect before cutting back over:
   velero backup get                              # find the latest good backup
   velero restore create --from-backup <name> \
     --namespace-mappings <old-ns>:<old-ns>-restored --wait

3. For stateful data specifically, Velero restores the PVC object, but the
   underlying volume snapshot/data depends on how the backup was taken
   (CSI snapshots vs. an Operator's own backup mechanism) — for a database
   running under an Operator (Chapter 5), prefer the Operator's own
   point-in-time recovery over relying on Velero for the data itself,
   and use Velero for everything else in the namespace (RBAC, ConfigMaps,
   Deployments, NetworkPolicies)

4. Validate application-level correctness after restore, not just object
   existence — kubectl get all looking "normal" doesn't confirm the
   database has the expected rows or the app can actually serve traffic

5. Once confirmed good, cut traffic back (DNS/Ingress) and only then
   decommission the "-restored" scratch copy

6. Post-incident: this is exactly why Chapter 16's "no bare kubectl delete
   namespace in prod without confirmation guardrails" mistake exists —
   add an OPA/Kyverno policy or admission webhook requiring an annotation
   or a second approval step before Namespace deletion is permitted
```

---

## 17.4 System Design Questions

**"Design a multi-tenant platform for 20 internal teams sharing one cluster."**

Key points to cover:
```
1. Soft multi-tenancy (Chapter 6) is the right default for internal teams
   under one trust boundary — one cluster, Namespace-per-team isolation

2. Per-team Namespace provisioned via GitOps (a template repo/ApplicationSet
   per team, not manual kubectl create namespace)

3. RBAC: one reusable ClusterRole per role type (developer, viewer, admin),
   bound per-namespace via RoleBindings — never a blanket ClusterRoleBinding

4. ResourceQuota + LimitRange per namespace sized from each team's actual
   usage pattern, reviewed periodically, not copy-pasted defaults

5. Default-deny NetworkPolicy in every namespace by default (enforced via
   Kyverno/Gatekeeper so a team can't accidentally omit it), plus a
   documented process for teams to request explicit allow rules for
   legitimate cross-namespace traffic

6. Pod Security Standards at baseline or restricted, enforced cluster-wide,
   with an documented, audited exception process for the rare workload
   that genuinely needs elevated privileges

7. Shared platform services (ingress controller, cert-manager, metrics)
   live in dedicated platform namespaces teams cannot modify, with
   NetworkPolicies allowing only the necessary paths (e.g., ingress
   controller → team namespaces on Service ports only)

8. Cost/usage visibility per namespace (resource requests as a rough proxy,
   or a proper tool like Kubecost) so teams see the consequence of their
   quota requests

9. Escalation path to hard multi-tenancy (dedicated node pools with taints,
   or a dedicated cluster) documented for the rare team that needs it —
   e.g. a team handling regulated data
```

**"Design a GitOps-based deployment pipeline with progressive delivery and automated rollback."**
```
1. Separate the application source repo from the GitOps/manifest repo
   (Chapter 8) — CI in the source repo builds and pushes an image, then
   opens a PR against the GitOps repo bumping the image tag

2. Argo CD (or Flux) watches the GitOps repo with automated sync +
   selfHeal enabled, so merging that PR is the only trigger needed

3. Argo Rollouts manages the actual workload as a Rollout resource with a
   canary strategy: small initial traffic weight, a pause, an automated
   analysis step querying Prometheus for error rate/latency, then
   progressively larger weights, mirroring Project 2 of this course

4. Automated rollback: the AnalysisTemplate's failureLimit triggers Argo
   Rollouts to automatically abort and revert to the stable ReplicaSet the
   moment the error-rate query fails its successCondition — no human
   needs to be paged to catch a bad deploy within its first few minutes

5. Notifications wired to Slack/PagerDuty on rollout pause, promotion, and
   abort events, so the team has visibility without needing to watch a
   dashboard during every deploy

6. Environment promotion (dev → staging → prod) modeled as separate
   directories/branches in the GitOps repo, each with its own Argo CD
   Application, so promoting a build is itself a Git operation (a PR
   bumping the tag in the next environment's directory) with full audit
   history

7. RBAC on the GitOps repo itself (who can merge to the prod directory)
   matters as much as RBAC on the cluster — Git branch protection is now
   part of your deployment security model, not just your code review process
```

**"Design a disaster recovery strategy for a stateful, multi-region Kubernetes platform."**
```
1. Start from business requirements, not tooling — get explicit RTO/RPO
   numbers for each tier of the platform (a payments database's RTO/RPO
   looks nothing like an internal admin tool's)

2. Stateful data tier: use each database's native replication for the
   tightest RPO (synchronous or near-synchronous cross-region replication
   for the lowest-RPO tier), backed by an Operator (Chapter 5) that
   understands failover semantics, PLUS independent backups to object
   storage in a separate region from the primary (never rely on
   replication alone — it replicates corruption and accidental deletes too)

3. Stateless/config tier: Velero scheduled backups (Chapter 10) of every
   namespace, replicated to a DR-region bucket, restorable into a standby
   cluster in the secondary region

4. Cluster tier: the cluster itself is defined as code (Terraform, from
   Topic 7) so a full replacement cluster in a new region is a
   `terraform apply` away, not a manual rebuild — combined with the
   GitOps repo, a brand-new cluster can reconcile itself back to the
   full application state automatically once it registers with Argo CD

5. Multi-cluster architecture (Chapter 11): either active-passive (a warm
   standby cluster in the DR region, scaled down, ready to take traffic)
   or active-active (both regions serving live traffic, with the harder
   problem of keeping stateful data consistent across both) — the choice
   is driven directly by the RTO number from step 1, since active-active
   gets close to zero RTO at much higher operational and data-consistency cost

6. Regularly drill the failover, not just document it — an untested DR
   plan is a hypothesis, and Project 3 of this course (a real restore
   drill) is exactly the muscle memory this requires at platform scale

7. DNS/traffic-management layer to actually redirect traffic during
   failover (global load balancer or DNS failover), tested as part of the
   same drill, since a perfect data recovery with no traffic redirection
   plan still means an outage
```

---

## 17.5 Quick-Fire Questions

| Question | Answer |
|----------|--------|
| Default Kubernetes authorization mode in most managed clusters? | RBAC |
| What binds a ClusterRole to a single namespace's subjects? | A RoleBinding referencing a ClusterRole |
| What command checks a specific identity's permissions? | `kubectl auth can-i ... --as=<identity>` |
| What CNI feature is required for NetworkPolicy to have any effect? | NetworkPolicy enforcement support (not all CNIs provide it) |
| What object type extends the Kubernetes API with a new Kind? | CustomResourceDefinition (CRD) |
| What runs the reconciliation logic for a Custom Resource? | An Operator (controller) |
| What does a service mesh sidecar typically provide? | mTLS, traffic shaping, retries/circuit breaking, telemetry |
| What's the pull-based deployment tool most associated with GitOps? | Argo CD (or Flux) |
| What Argo Rollouts strategy gradually shifts traffic to a new version? | Canary |
| What consensus algorithm does etcd use? | Raft |
| What prevents a node drain from violating availability guarantees? | PodDisruptionBudget |
| What tool provisions right-sized nodes on demand rather than scaling fixed node groups? | Karpenter |
| What tool is used for Kubernetes-native backup and restore? | Velero |
| What admission mechanism enforces the built-in `restricted`/`baseline` policy levels? | Pod Security Admission |
| What log captures every request made to the API server? | The audit log |

---

## 17.6 "Walk Me Through Your Platform Engineering Experience"

STAR format example:

```
Situation: Our organization had 8 application teams all deploying directly
to a single shared Kubernetes cluster via personal kubeconfigs and manual
`kubectl apply`/`helm upgrade`. There was no consistent RBAC model, no
NetworkPolicies (any Pod could reach any other Pod, including across
teams), and a cluster upgrade six months earlier had caused a
multi-hour outage when a node drain evicted every replica of a critical
service at once.

Task: Turn the cluster into a governed, self-service platform that
application teams could deploy to safely and independently, without the
platform team becoming a bottleneck for every deploy or a single point of
security failure.

Action:
1. Designed a soft multi-tenancy model (Namespace-per-team) with
   ResourceQuotas sized from each team's actual historical usage, and
   RBAC built from reusable ClusterRoles bound per-namespace so every
   team had identical, least-privilege permissions scoped to their own
   namespace only
2. Rolled out default-deny NetworkPolicies cluster-wide via a Kyverno
   policy that auto-generated one for any new namespace, plus a
   lightweight request process for teams needing explicit cross-namespace
   allow rules
3. Migrated deployment from manual kubectl/helm to Argo CD, with each
   team's manifests living in their own directory of a shared GitOps
   repo — deploys became "merge a PR," and Argo CD's selfHeal eliminated
   the drift that used to accumulate from ad hoc kubectl edits
4. Added Argo Rollouts canary strategies for the platform's highest-risk
   services, with an automated analysis step against existing Prometheus
   metrics, catching two bad deploys automatically before they reached
   full traffic in the following quarter
5. Added PodDisruptionBudgets to every multi-replica service and made
   node drains part of a documented, PDB-respecting runbook — directly
   addressing the earlier outage's root cause
6. Introduced Velero scheduled backups and ran a quarterly restore drill
   into a scratch namespace, turning "we have backups" into a measured,
   tested RTO number the team could actually stand behind

Result: Deploy frequency for application teams roughly tripled once they
could self-serve via Git PRs instead of filing tickets against the
platform team. Zero cross-team security incidents in the twelve months
after NetworkPolicies rolled out, versus an open blast radius before. The
canary rollout process caught two regressions automatically with zero
customer-facing impact. The next scheduled cluster upgrade, run against
the same architecture, completed with zero unplanned downtime because
PDBs did exactly what they were designed to do.
```

**Self-Check Before Your Interview**

- Can you explain, without notes, the full request lifecycle from `kubectl apply` through authentication, authorization, admission control, and persistence to etcd?
- Can you describe the actual difference between how RBAC and NetworkPolicy handle "multiple applicable rules" (both are purely additive — there's no explicit deny)?
- Can you walk through, out loud, why a mutating webhook must run before a validating webhook and give a concrete example where reversing that order breaks something?
- Can you talk through a real incident (from this course's projects or your own experience) using the diagnostic flow from section 17.3, narrating your reasoning rather than just stating the final answer?
- Can you state a defensible RTO/RPO number for a system you've actually operated, and explain what technical choice produced that number?

No separate hands-on exercise for this chapter — working through the scenarios and system design questions above out loud, from memory, and defending your reasoning under follow-up questions, is the exercise.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./16-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./18-course-summary.md">Next: Course Summary →</a>
</div>
