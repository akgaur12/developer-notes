# Chapter 15 — Common Mistakes and Pitfalls

## Learning Objectives

By the end of this chapter, you will be able to:

- Identify the most common platform/cluster-admin-level Kubernetes mistakes by recognizing their symptoms
- Understand the organizational and technical misunderstanding that leads to each mistake
- Apply the correct pattern from earlier chapters of this course immediately, without having to look it up
- Recognize these mistakes in a manifest or cluster review before they cause an incident

---

## How to Read This Chapter

Each mistake is presented with four parts:

1. **The wrong pattern** — a manifest, command, or organizational habit you will encounter in the wild
2. **Why it happens** — the misunderstanding or pressure that leads to it
3. **The correct fix** — a drop-in replacement or corrected practice, pointing back to the chapter that covers it in depth
4. **Impact / Prevention** — what breaks in production, and how to stop it happening again

These are platform-level mistakes — made by the people operating the cluster as shared infrastructure, not by application teams deploying to it. If you're looking for app-deployment mistakes (missing probes, `:latest` tags, no resource limits), those are covered in Kubernetes Basics, Chapter 16 — this chapter assumes you already know and avoid all of those, and covers what goes wrong one altitude level up.

---

## Mistake 1: Granting `cluster-admin` Broadly to Unblock a Team

```yaml
# WRONG — the whole team, or worse, a CI ServiceAccount, gets cluster-admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: unblock-team-checkout
subjects:
  - kind: Group
    name: team-checkout
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: ci-deployer
    namespace: ci
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

**Why it happens:** A team is blocked mid-deploy by a permissions error, someone on the platform team is under pressure to unblock them *right now*, and `cluster-admin` is the one binding guaranteed to make any permissions error disappear immediately. It's meant to be temporary. It never gets revisited.

**The correct fix:** Scope a Role/ClusterRole to exactly the verbs and resources the team or CI pipeline actually needs, in exactly the namespaces it needs them in (Chapter 2):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: team-checkout-prod
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: team-checkout-prod
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: ci
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**Impact:** A `ClusterRoleBinding` to `cluster-admin` grants unrestricted access to every resource, in every namespace, including Secrets, RBAC objects themselves, and the ability to escalate further. A CI ServiceAccount with `cluster-admin` means anyone who can trigger that pipeline — or anyone who compromises the CI system itself — has full control of the entire cluster, not just the ability to deploy the one application the pipeline was built for. This is one of the largest single privilege-escalation surfaces a cluster can have, and it's usually created under time pressure with the explicit intention of "fixing it properly later."

**Prevention:** Treat any `cluster-admin` binding as an incident-worthy finding during a security review, not a normal configuration. Use `kubectl auth can-i --list` (Chapter 14) on a recurring schedule to catch these before an audit does. When a team is genuinely blocked, take the extra five minutes to scope a Role correctly instead of reaching for the binding that fixes everything.

---

## Mistake 2: Writing a NetworkPolicy Without Confirming the CNI Supports It

```yaml
# WRONG — this looks correct and applies without error...
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

**Why it happens:** `kubectl apply` on a NetworkPolicy manifest succeeds with no error, no warning, and no indication whatsoever that the CNI plugin actually installed on the cluster ignores NetworkPolicy objects entirely. The Kubernetes API happily stores any valid NetworkPolicy object regardless of whether anything in the cluster is watching it.

**The correct fix:** Confirm the CNI plugin actually enforces NetworkPolicy *before* relying on one (Chapter 4) — Calico, Cilium, and Weave Net do; the basic `kubenet` and some cloud default CNI configurations historically do not without additional configuration:

```bash
# Verify the CNI, then verify enforcement empirically, not just by reading docs
kubectl get pods -n kube-system -o wide | grep -iE 'calico|cilium|weave'

# The only real test: deploy the deny-all policy, then confirm traffic
# that should be blocked actually is
kubectl run test-client --image=busybox -it --rm -- wget -T2 target-service
# If this succeeds despite a deny-all NetworkPolicy in place, the CNI
# is not enforcing NetworkPolicy at all
```

**Impact:** A team believes their namespace is locked down because the NetworkPolicy manifest exists and was applied successfully — and builds their entire security posture around that belief. In reality, every Pod in the "isolated" namespace remains fully reachable from anywhere in the cluster. This is worse than having no NetworkPolicy at all, because it creates false confidence that gets discovered, if ever, only during a penetration test or a real breach.

**Prevention:** Verify NetworkPolicy enforcement empirically on every new cluster before depending on it, not just by checking which CNI is installed. Add an automated test (a small Job that attempts a connection that should be blocked) as part of cluster provisioning validation, so a misconfigured or non-enforcing CNI fails a check immediately instead of silently failing forever.

---

## Mistake 3: Rolling Out an Admission Policy Straight to `enforce`

```yaml
# WRONG — no dry-run period, applied directly to enforce, cluster-wide
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce   # goes live immediately, no warning period
  rules:
    - name: check-limits
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "CPU and memory limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

**Why it happens:** The policy looks obviously correct and the rule itself is good practice (Chapter 3, Chapter 14) — the mistake isn't the rule, it's skipping the rollout process. It's tempting to skip the audit period because "this is clearly a good rule, why would it break anything."

**The correct fix:** Deploy every new policy in `audit` mode first, review what it *would have* rejected across the existing fleet, fix those workloads, and only then flip to `enforce`:

```yaml
spec:
  validationFailureAction: Audit    # logs violations, does not block anything
```

```bash
# Review what the policy would have blocked, across every existing
# workload, before it can actually block anything
kubectl get events -A --field-selector reason=PolicyViolation
```

**Impact:** A cluster full of existing Deployments that predate the policy — some without resource limits, deployed months ago and still running fine — suddenly can't be updated, scaled, or have their Pods rescheduled the moment the policy goes to `enforce`, because every new Pod creation is now rejected at admission time. This typically surfaces at 9 AM when the first ordinary deploy of the day fails cluster-wide with a cryptic admission-webhook rejection, and the platform team spends the morning firefighting instead of the rollout being a non-event.

**Prevention:** Every new admission policy goes through `audit` mode for a defined minimum period (at least one full deploy cycle across all affected teams) before `enforce`. Treat "audit period completed with zero unexpected violations" as a hard gate before flipping the mode, the same way you'd gate a database migration on a successful dry run.

---

## Mistake 4: Building a Custom Operator for a Problem Already Solved

**Why it happens:** A team needs highly-available Postgres, or automated TLS certificate rotation, and the requirement looks simple enough on paper ("just watch for a CR and reconcile a StatefulSet") that writing a small custom controller feels faster than researching and adopting someone else's Operator, which comes with its own CRDs and configuration surface to learn.

**The correct fix:** Check OperatorHub and the broader ecosystem before writing anything custom (Chapter 5, Chapter 14):

```bash
# Before building custom Postgres HA reconciliation logic —
# CloudNativePG and the Zalando Postgres Operator already solve this
# Before building custom cert rotation — cert-manager already solves this
```

**Impact:** A custom-built Operator written in a sprint, without years of accumulated production hardening, handles the happy path fine and then fails in exactly the edge cases a mature Operator has already encountered and fixed: a failover that leaves two Postgres instances both believing they're primary (split-brain), a certificate renewal that races with an in-flight request and briefly serves an expired cert, a reconcile loop that doesn't handle a partially-applied Kubernetes object correctly. The team ends up maintaining bespoke, under-tested infrastructure software as a side project to their actual job.

**Prevention:** Make "check for an existing Operator" a mandatory first step, documented in your platform's engineering process, before any custom controller work is approved. Reserve custom Operators for genuinely novel, organization-specific operational logic — not for problems the broader Kubernetes ecosystem has already solved well.

---

## Mistake 5: Treating Namespaces as a Security Boundary on Their Own

```yaml
# WRONG — a namespace exists, but nothing else isolates it
apiVersion: v1
kind: Namespace
metadata:
  name: team-checkout-prod
# No RBAC scoped to this namespace, no NetworkPolicy, no ResourceQuota —
# "it's in its own namespace" is treated as isolation, full stop
```

**Why it happens:** A Namespace object visually looks like a boundary — resources inside it are grouped together, `kubectl get pods -n team-checkout-prod` only shows that team's Pods — and it's easy to conflate "logically grouped" with "isolated."

**The correct fix:** A namespace only becomes an actual isolation boundary when RBAC, NetworkPolicies, and ResourceQuotas are all layered on top of it deliberately (Chapter 6):

```yaml
# RBAC scoped to this namespace only
# + a default-deny NetworkPolicy (Chapter 4, Chapter 14)
# + a ResourceQuota capping this namespace's resource consumption
# all three, applied together, every time a namespace is created
```

**Impact:** Without RBAC scoping, any user or ServiceAccount with cluster-wide `get`/`list` permissions can read every Secret and object in the "isolated" namespace just as easily as their own. Without a NetworkPolicy, every Pod in the namespace is fully reachable from every other namespace in the cluster by default. Without a ResourceQuota, one misconfigured Deployment in the namespace can consume enough of the cluster's shared capacity to degrade every other team, defeating the entire point of separating them in the first place.

**Prevention:** Treat "namespace exists" as step one of isolation, never the whole thing. Bake all three layers into your namespace-provisioning template (Chapter 14) so no namespace is ever created without them, rather than relying on each team to remember and apply them individually.

---

## Mistake 6: Adopting a Service Mesh for Its Own Sake

**Why it happens:** Service mesh adoption is often driven by industry visibility rather than a concrete internal need — "Istio is what serious platform teams run" — rather than a specific, named problem the team currently has that a mesh actually solves.

**The correct fix:** Name the concrete problem first (Chapter 7, Chapter 14). If you can't name one beyond "best practice" or "everyone uses it," don't adopt a mesh yet:

```
Legitimate: "We need mTLS between every service for a compliance
requirement, and don't want to build it into every app individually."

Not legitimate: "We should probably have a service mesh."
```

**Impact:** A mesh adds a sidecar proxy to every single Pod in the cluster, meaning every request now takes an extra network hop through Envoy (or equivalent) on both the sending and receiving side, adding real latency at scale. It adds a substantial new piece of infrastructure the platform team must now operate, upgrade, and troubleshoot — control plane components, certificate rotation, CRDs, its own failure modes layered on top of Kubernetes' own. Teams that adopt a mesh without a concrete driving need end up paying this operational and latency cost indefinitely while using a small fraction of the mesh's actual feature set, and often can't articulate what would break if they removed it.

**Prevention:** Require a written justification — the specific problem, and why simpler alternatives (mTLS via cert-manager plus application-level enforcement, traffic splitting via Ingress/Argo Rollouts) don't solve it — before mesh adoption is approved. Revisit that justification periodically; if the original problem no longer exists or was solved another way, the mesh may no longer be earning its operational cost.

---

## Mistake 7: Allowing Manual Changes Against a GitOps-Managed Cluster

```bash
# WRONG — "just a quick temporary fix" directly against a cluster
# that Argo CD or Flux is actively reconciling
kubectl edit deployment api -n production
kubectl scale deployment api --replicas=10 -n production
```

**Why it happens:** During an incident, editing the live cluster directly feels faster than going through a Git commit, a PR, and a sync — and in the moment, it often is faster. The mistake is not the emergency edit itself; it's not immediately following up in Git.

**The correct fix:** Any change made directly against a GitOps-managed cluster must be reflected back into Git immediately, or reverted, within minutes — not left as a standing manual change (Chapter 8):

```bash
# Either: commit the change to Git right away so Argo CD/Flux's next
# sync doesn't fight it
git commit -am "Scale api to 10 replicas for incident mitigation"
git push

# Or: understand that the GitOps controller will revert it on its next
# sync cycle, and that's the expected, correct behavior if you don't
# intend the change to be permanent
```

**Impact:** Argo CD and Flux continuously reconcile the live cluster back to whatever Git declares. An engineer who makes a manual change and moves on to the next fire, intending to "commit it properly later," finds the GitOps controller silently reverting their fix on its next sync cycle — sometimes seconds later, sometimes minutes later — reintroducing the exact problem they thought they'd solved. This produces a uniquely confusing class of incident where a fix appears to work and then mysteriously un-happens, and the engineer often doesn't immediately connect the two events.

**Prevention:** Establish and communicate the rule clearly: manual `kubectl edit`/`apply`/`scale` against a GitOps-managed cluster is for the seconds it takes to mitigate an active incident, and is followed immediately by a Git commit reflecting the same change (or an explicit acceptance that it will be reverted). Some teams pause the GitOps controller's auto-sync explicitly during an active incident to avoid this race entirely, then resume it once the Git-based fix is in place.

---

## Mistake 8: Draining Nodes Without PodDisruptionBudgets or Cordoning Properly

```bash
# WRONG — force-deleting Pods during an upgrade instead of a proper drain,
# or draining a node with no PDB protecting a single-replica critical service
kubectl delete node worker-03
# or
kubectl drain worker-03 --force --delete-emptydir-data --ignore-daemonsets
# run against a node hosting the only replica of a critical service,
# with no PodDisruptionBudget in place to stop it
```

**Why it happens:** Routine node maintenance (an OS patch, a Kubernetes version upgrade per Chapter 9) feels mechanical and low-risk, so it's easy to skip verifying that every critical workload on the node actually has a PDB before draining — especially under time pressure to get through a batch of nodes quickly.

**The correct fix:** Cordon before draining, verify PDBs exist for anything critical, and let `kubectl drain` (without `--force`) respect them:

```bash
kubectl cordon worker-03                 # stop new Pods from scheduling here first
kubectl get pdb -A                       # confirm PDBs exist for what's running here
kubectl drain worker-03 --ignore-daemonsets --delete-emptydir-data
# a properly configured drain blocks/retries rather than violating a PDB
```

**Impact:** Draining a node with no PDB protecting a single-replica "critical" service (one that should never have been single-replica in the first place, but exists in production anyway) causes a complete, avoidable outage during what was supposed to be routine maintenance — the exact scenario a PDB exists to prevent. Using `--force` bypasses PDB protection entirely, which is sometimes genuinely necessary (a stuck Pod that won't evict cleanly) but is dangerous as a default habit rather than a deliberate, rare override.

**Prevention:** Make "PDB coverage check" a mandatory pre-flight step in your node upgrade runbook (Chapter 9, Chapter 14), before draining any node. Never use `--force` as a default flag; treat it as an explicit, justified exception each time.

---

## Mistake 9: Never Testing a Backup/Restore End to End

**Why it happens:** A Velero backup schedule shows a green checkmark every night. That's reassuring enough that nobody questions it further — until a real disaster forces an actual restore, for the first time, under maximum pressure.

**The correct fix:** Schedule regular, actual restore drills into a separate scratch cluster (Chapter 10, Chapter 14), and verify the restored application is genuinely healthy with intact data — not just that the restore command exited successfully:

```bash
# A successful backup schedule tells you almost nothing on its own
velero schedule create daily-prod-backup --schedule="0 2 * * *" --include-namespaces=production

# The only real verification: actually restore it, regularly, somewhere else
velero restore create --from-backup daily-prod-backup-20260701020000 \
  --namespace-mappings production:production-restore-test
kubectl get pods -n production-restore-test
# then actually query the restored database, check the data is really there
```

**Impact:** Backups silently missing a Secret the application needs to start (a common gap — some backup configurations exclude Secrets by default, or a namespace selector doesn't match one that was added later), a PVC snapshot that restores but with corrupted or stale data, or a restore process that was simply never run before and fails on a step nobody anticipated — all of these are discovered, in the worst version of this mistake, during the actual disaster the backup existed to protect against.

**Prevention:** Put restore drills on the same operational calendar as any other recurring maintenance — quarterly, at minimum — and treat a drill that reveals a gap as a success (it found the problem before a real disaster did), not a failure to be quietly ignored.

---

## Mistake 10: Reaching for Multi-Cluster Prematurely

**Why it happens:** Multi-cluster architectures (Chapter 11) solve real problems — blast-radius containment, regulatory data residency, regional latency — and reading about them makes the pattern sound like an obviously superior default. Teams adopt it speculatively, "for when we need it," rather than when a concrete need actually arrives.

**The correct fix:** Stay on a single, well-operated cluster until a specific, named requirement forces the move (Chapter 11, Chapter 14) — the same discipline as the multi-tenancy decision in Mistake 5's territory, one level up:

```
Legitimate driver: "Regulatory requirement — EU customer data must
stay in an EU region, and our current cluster is US-only."

Not yet a driver: "We might want this flexibility eventually."
```

**Impact:** Multi-cluster multiplies nearly every operational task by the number of clusters: N times the upgrade runbooks (Chapter 9) to execute and keep in sync, N times the RBAC configuration to audit (Mistake 1's territory, now repeated per cluster), N times the monitoring and alerting surface, N times the etcd backup/restore drills (Mistake 9). Teams that adopt this prematurely find their platform engineering capacity consumed by operating multiple clusters instead of improving the one cluster their actual current scale requires.

**Prevention:** Require the same kind of concrete, written justification demanded for service mesh adoption (Mistake 6) before approving a move to multi-cluster. A single cluster, properly operated with the practices in Chapter 14, comfortably serves far more scale than most teams reaching for multi-cluster actually have.

---

## Mistake 11: Running VPA in `Auto` Mode Together With HPA on CPU

```yaml
# WRONG — both controllers are simultaneously trying to manage the
# same Deployment's capacity, using conflicting mechanisms
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"      # VPA actively changes resource requests
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  metrics:
    - type: Resource
      resource:
        name: cpu                  # HPA scales based on % of the CPU *request*
        target:
          type: Utilization
          averageUtilization: 70
```

**Why it happens:** VPA and HPA look complementary on the surface — one scales resources per Pod, the other scales replica count — and it's reasonable to assume running both gives you the best of both worlds automatically (Chapter 12).

**The correct fix:** Never let VPA (in `Auto` mode) manage the same resource dimension (CPU) that HPA scales on. Use VPA in `Off`/recommendation mode to inform your static requests (Chapter 12, Chapter 14), or scope VPA `Auto` mode strictly to memory while HPA scales on CPU, if your workload's usage pattern genuinely supports splitting them that way:

```yaml
updatePolicy:
  updateMode: "Off"    # observe only; you apply the numbers deliberately
```

**Impact:** HPA's CPU-utilization calculation is a percentage *of the Pod's CPU request* (`current usage / requested CPU`). When VPA in `Auto` mode changes that request out from under HPA — evicting and recreating the Pod with a new request value — the percentage HPA is computing shifts immediately, with no actual change in real traffic or load. This causes the two controllers to fight: HPA scales out because the percentage spiked (due to VPA having just lowered the request, not because load increased), VPA then resizes again based on the new replica count's usage pattern, and the Deployment's replica count and resource requests oscillate unpredictably instead of settling into a stable, correctly-sized state.

**Prevention:** Pick one mechanism to own CPU-based scaling for a given Deployment — either HPA scales replicas on a stable, manually-set CPU request, or VPA manages the request in `Auto` mode and HPA scales on a different signal (custom metrics, not CPU). Never let both actively manage the same dimension simultaneously.

---

## Mistake 12: Not Enabling or Reviewing Audit Logs Until After an Incident

**Why it happens:** Audit logging (Chapter 13) is invisible infrastructure with no immediate payoff — it doesn't make deploys faster, doesn't show up on a feature roadmap, and its absence causes no symptoms at all, right up until the moment a forensic question needs answering.

**The correct fix:** Enable audit logging as part of initial cluster provisioning, ship it to a durable off-cluster log store immediately, and review it periodically as described in Chapter 14 — not only when an incident forces a search through it:

```yaml
# Enabled from day one, not retrofitted after an incident reveals its absence
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
```

**Impact:** "Who deleted the production Deployment at 2 AM" is only answerable if audit logging was already running *before* the deletion happened — there is no way to retroactively generate the record after the fact. Teams that discover audit logging wasn't enabled only during the incident that needed it are left with no forensic trail at all: no way to confirm whether it was an authorized action gone wrong, a compromised credential, or a bug in an automated tool, and no way to prevent a repeat because the actual cause is unknowable.

**Prevention:** Treat audit logging as a mandatory, non-optional part of cluster bring-up (the same category as RBAC and network policy, not an optional add-on) and verify during cluster provisioning validation that it's actually flowing to a durable log store, not just enabled in configuration.

---

## Mistake 13: Letting a Managed Cluster Get Force-Upgraded

**Why it happens:** Managed Kubernetes offerings (EKS, GKE, AKS) auto-upgrade control planes that fall too far out of the provider's supported version window, on the provider's schedule, not yours. Teams that don't proactively track their version's support end date get "upgraded" involuntarily, with no chance to test compatibility first.

**The correct fix:** Track your managed cluster's Kubernetes version support end date explicitly and schedule your own upgrade well before the provider's forced deadline (Chapter 9, Chapter 14):

```bash
# Know this date before the provider tells you via a forced-upgrade notice
aws eks describe-cluster --name prod --query 'cluster.version'
# cross-reference against your provider's published support window
```

**Impact:** A provider-forced upgrade happens on a schedule you don't control, potentially during a period that's inconvenient for your business, and skips the testing, staging validation, and rehearsed runbook execution (Chapter 9, Chapter 14) that a planned upgrade would normally go through. API deprecations between versions that would have been caught in a staging environment instead surface for the first time directly in production, at a moment you didn't choose.

**Prevention:** Put Kubernetes version end-of-support dates on the same operational calendar as certificate expirations and backup drills. Upgrade proactively, on your own tested schedule, comfortably ahead of any provider-forced deadline.

---

## Mistake 14: Identical Resource and Autoscaling Configuration Across All Environments

```yaml
# WRONG — the exact same values.yaml (or manifest) used unchanged
# across dev, staging, and production
resources:
  requests:
    cpu: 2
    memory: 4Gi
  limits:
    cpu: 4
    memory: 8Gi
# identical HPA thresholds and min/max replicas in every environment too
```

**Why it happens:** Copy-pasting the same manifest or Helm values file across environments is the path of least resistance, and "it worked in staging" feels like sufficient validation that the same configuration is correct everywhere.

**The correct fix:** Size each environment deliberately, using per-environment overrides (Kustomize overlays or Helm values files per environment), informed by that environment's actual traffic and VPA recommendation data (Chapter 12, Chapter 14):

```yaml
# values-dev.yaml — a fraction of production's traffic, sized down accordingly
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits: { cpu: 250m, memory: 256Mi }
autoscaling:
  minReplicas: 1
  maxReplicas: 2

# values-prod.yaml — sized from real observed production traffic
resources:
  requests: { cpu: 2, memory: 4Gi }
  limits: { cpu: 4, memory: 8Gi }
autoscaling:
  minReplicas: 5
  maxReplicas: 40
```

**Impact:** Applying production-sized resource requests in dev/staging wastes real cluster cost on environments with a tiny fraction of production's traffic, multiplied across every developer or feature-branch environment that copies the same values. Applying dev/staging-sized values (or dev's conservative autoscaling thresholds) in production under-provisions the environment that actually needs headroom, causing the throttling and eviction failure modes from Chapters 12 and 13 under real production load — while "it worked in staging" gave false confidence, because staging never generated the traffic pattern that would have exposed the problem.

**Prevention:** Never treat "works in staging" as validation of resource or autoscaling correctness for production — traffic volume and pattern differ fundamentally between environments. Maintain explicit per-environment overrides, and size production specifically from production's own observed data (VPA recommendation mode, Chapter 14), not from staging's numbers scaled up by guesswork.

---

## Summary

| # | Mistake | Key Fix |
|---|---------|---------|
| 1 | Broad `cluster-admin` grants to unblock a team/CI | Scoped Roles/RoleBindings; audit with `kubectl auth can-i --list` |
| 2 | NetworkPolicy applied without confirming CNI enforcement | Verify enforcement empirically before relying on it |
| 3 | Admission policy shipped straight to `enforce` | `audit` mode first, fix violations, then `enforce` |
| 4 | Custom Operator for an already-solved problem | Check OperatorHub before building custom controllers |
| 5 | Namespace treated as isolation on its own | Layer RBAC + NetworkPolicy + ResourceQuota on every namespace |
| 6 | Service mesh adopted without a concrete need | Require a named problem before adopting a mesh |
| 7 | Manual changes against a GitOps-managed cluster | Commit immediately, or accept the GitOps revert |
| 8 | Draining nodes without PDB coverage | Cordon, verify PDBs, drain without `--force` |
| 9 | Backups never restore-tested | Scheduled restore drills into a scratch cluster |
| 10 | Multi-cluster adopted prematurely | Require a concrete blast-radius/compliance/latency driver first |
| 11 | VPA `Auto` mode fighting HPA on CPU | One controller owns CPU scaling; VPA `Off` mode to inform requests |
| 12 | Audit logs not enabled/reviewed until an incident | Enable at cluster bring-up; review on a standing cadence |
| 13 | Managed cluster force-upgraded by the provider | Track support windows; upgrade proactively on your own schedule |
| 14 | Identical resource/autoscaling config across environments | Per-environment overrides sized from real observed data |

---

## Knowledge Check

1. Why is a `ClusterRoleBinding` granting `cluster-admin` to a CI ServiceAccount more dangerous than the same grant to a single human user?
2. A NetworkPolicy applies successfully with `kubectl apply` and shows no errors. Why does this tell you nothing about whether it's actually enforced?
3. What is the purpose of the `audit` mode on an admission policy, and what specifically should you verify before switching it to `enforce`?
4. Explain the mechanism by which running VPA in `Auto` mode and HPA on CPU utilization causes the two controllers to fight each other.
5. Why is a green checkmark on a nightly Velero backup schedule insufficient evidence that disaster recovery will actually work?
6. What distinguishes a legitimate reason to adopt a service mesh or multi-cluster architecture from a premature one, according to this chapter?

---

## Hands-On Exercise

**Find and Fix the Broken Platform Configuration**

The manifest set below contains at least 5 of the mistakes covered in this chapter. Find every one of them, explain why each is harmful, and rewrite the configuration correctly using the patterns from this course.

```yaml
# 1) CI pipeline access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ci-full-access
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: ci
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
# 2) "Isolation" for a new team namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: team-inventory-prod
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
# (this cluster's CNI is the basic default bridge plugin, which does not
# enforce NetworkPolicy — nobody has verified this)
---
# 3) New Kyverno policy, pushed straight to production
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce   # went live this morning, no audit period
  rules:
    - name: check-runasnonroot
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "must not run as root"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
---
# 4) A "critical" checkout service, about to be drained for a node upgrade
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: team-inventory-prod
spec:
  replicas: 1
  selector:
    matchLabels: { app: checkout }
  template:
    metadata:
      labels: { app: checkout }
    spec:
      containers:
        - name: checkout
          image: myorg/checkout:2.3.0
# no PodDisruptionBudget exists anywhere for this Deployment
---
# 5) Nightly backup schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: nightly-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - team-checkout-prod
      - team-payments-prod
      # team-inventory-prod (created last quarter) was never added here
      # and this has never been tested with an actual restore
```

Steps:

1. List every mistake you can find (aim for at least 5). Hint: check what the CNI actually supports, check for a PDB before assuming the checkout Deployment is safe to drain, and check the backup schedule's namespace list against every namespace that actually exists.
2. Rewrite item 1 as a scoped `Role`/`RoleBinding` granting only what the CI pipeline actually needs.
3. For item 2, describe the two steps you'd take before trusting this NetworkPolicy at all.
4. For item 3, rewrite it starting in `Audit` mode, and describe what you'd check before ever switching it to `Enforce`.
5. For item 4, rewrite the Deployment with a safe replica count and add a matching `PodDisruptionBudget`.
6. For item 5, fix the namespace list and describe the restore drill you'd run to actually validate this backup, not just trust its schedule's green checkmark.

---

## Further Reading

- [Kubernetes Documentation — RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes Documentation — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kyverno Documentation — Applying Policies](https://kyverno.io/docs/applying-policies/)
- [Velero Documentation — Backup Validation](https://velero.io/docs/main/disaster-case/)
- [Kubernetes Documentation — Horizontal and Vertical Autoscaling Interaction](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#known-limitations)
- [Kubernetes Documentation — Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-projects.md">Next: Hands-On Projects →</a>
</div>
