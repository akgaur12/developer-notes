# Chapter 14 — Best Practices

## Learning Objectives

By the end of this chapter, you will be able to:

- Apply a platform/cluster-admin-level checklist synthesizing Chapters 2–13 of this course
- Audit RBAC permissions on a running cluster instead of trusting what a manifest claims to grant
- Explain why policy should be enforced as code rather than relied on as human review discipline
- Choose the lightest isolation, mesh, and multi-cluster model that actually matches your trust boundary
- Build a platform that makes the secure/correct path the easy path for application teams

---

## Prerequisites for This Chapter

- RBAC and authentication — Chapter 2
- Admission control and Pod Security — Chapter 3
- NetworkPolicies — Chapter 4
- Custom resources and Operators — Chapter 5
- Multi-tenancy — Chapter 6
- Service mesh — Chapter 7
- GitOps — Chapter 8
- Cluster administration and upgrades — Chapter 9
- Backup and disaster recovery — Chapter 10
- Multi-cluster architectures — Chapter 11
- Autoscaling and performance tuning — Chapter 12
- Auditing and troubleshooting at scale — Chapter 13

This chapter assumes you've read all of the above — it doesn't reteach any of them, it tells you what to actually *do* with them on a real production platform.

---

## Apply Least-Privilege RBAC by Default, and Audit It Regularly

Chapter 2 covered how to write Roles and RoleBindings. In production, the discipline that matters more than any individual manifest is refusing to grant broad permissions "to unblock someone quickly" and then never revisiting it.

```bash
# Don't trust what a Role *says* it grants — verify what a ServiceAccount
# or user can *actually* do, which is what actually matters
kubectl auth can-i --list --as=system:serviceaccount:ci:deployer -n production

# Check a single, specific permission before assuming it's missing or present
kubectl auth can-i delete deployments --as=system:serviceaccount:ci:deployer -n production
```

Run `kubectl auth can-i --list` against every ServiceAccount with production access on a recurring schedule (a quarterly access review, at minimum, tied to your compliance calendar if you have one), not just at the moment a Role is first written. Permissions accumulate silently over time — someone adds `"*"` to a verb list to unblock a deploy under time pressure, and it never gets narrowed back down once the incident is over. An audit cadence is the only thing that catches this before it becomes the RBAC equivalent of technical debt with security consequences.

A practical audit script, run cluster-wide rather than one ServiceAccount at a time, makes this cheap enough to actually run on schedule instead of being skipped because it's tedious:

```bash
# Enumerate every ServiceAccount in every namespace and dump what it can
# actually do, rather than trusting the RoleBindings alone
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  for sa in $(kubectl get sa -n "$ns" -o jsonpath='{.items[*].metadata.name}'); do
    echo "== $ns/$sa =="
    kubectl auth can-i --list --as="system:serviceaccount:$ns:$sa" -n "$ns"
  done
done
```

Also watch for **implicit** privilege beyond what a Role literally lists: a Role that can `create` or `patch` `RoleBindings` in its own namespace can bind itself (or any ServiceAccount it can act as) to a more powerful ClusterRole later, which is a privilege-escalation path that a naive read of the Role's `rules` block won't reveal. Treat `rolebindings`/`clusterrolebindings` write access as sensitive as `cluster-admin` itself when auditing.

---

## Enforce Policy as Code, Not Tribal Knowledge

Chapter 3 introduced Gatekeeper and Kyverno as ways to enforce rules automatically at admission time. The production posture this enables: **rules that matter should never depend on a human remembering them during code review.**

```yaml
# Kyverno ClusterPolicy — reject any Pod requesting to run as root
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-runasnonroot
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Containers must set runAsNonRoot: true"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
```

The specific rules worth codifying as constraints, not checklist items in a PR template that people skim past under deadline pressure: no root containers, mandatory `resources.requests`/`resources.limits`, no `:latest` image tags, no privileged Pods, mandatory labels for cost allocation and ownership tracing. A PR reviewer who is tired, rushed, or simply reviewing their tenth Deployment of the day *will* eventually miss one of these. An admission controller never gets tired.

---

## Default-Deny Networking as the Baseline, Not an Afterthought

Chapter 4 covered NetworkPolicy syntax. The production practice: every namespace should start from a default-deny posture the moment it's created, with explicit allow rules layered on top — not the reverse, where teams add restrictions only after a security review flags the gap months later.

```yaml
# Applied automatically to every new namespace via GitOps or a namespace
# provisioning Operator — never left to individual teams to remember
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-checkout-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Bake this into namespace provisioning (a template, an Operator, or a GitOps overlay applied the moment a namespace is created) so "default-deny" is a platform guarantee every team inherits automatically, rather than a recommendation in a wiki page that only the security-conscious teams actually follow.

---

## Prefer an Existing Operator Over Building Custom Automation

Chapter 5 covered the Operator pattern and how much operational knowledge a good Operator encodes — failover, backup scheduling, version-aware upgrades, self-healing specific to the thing it manages. Before writing a custom controller for a problem that looks like "just some reconciliation logic," check whether that problem is already solved.

```bash
# Before building anything custom for Postgres HA, cert rotation,
# Kafka, Elasticsearch, etc. — check OperatorHub first
# https://operatorhub.io
```

A custom controller you build in a sprint will, almost by definition, handle far fewer edge cases than an Operator that's been hardened against real production failures across hundreds of organizations for years (the Zalando/CloudNativePG Postgres Operators, cert-manager, the Strimzi Kafka Operator, and similar). Writing your own is occasionally the right call when your requirements are genuinely novel — but that should be the conclusion of a deliberate evaluation, not the default because nobody checked OperatorHub first.

---

## Choose the Lightest Isolation Model That Matches Your Actual Trust Boundary

Chapters 6 and 11 covered the spectrum from namespace-based soft multi-tenancy up through hard multi-tenancy (virtual clusters, separate node pools) and all the way to separate physical clusters. Each step up that ladder costs real operational overhead: more RBAC surfaces to manage, more monitoring to wire up, more upgrade cycles to run.

The practice: match the isolation model to the actual trust boundary, not to whichever pattern is currently fashionable in the platform-engineering community.

| Trust boundary | Appropriate isolation |
|---|---|
| Different teams, same company, same compliance requirements, mutual trust | Namespaces + RBAC + NetworkPolicies + ResourceQuotas (soft multi-tenancy) |
| Different teams, need guaranteed resource/API isolation, still same org | Virtual clusters or dedicated node pools with taints (hard multi-tenancy) |
| Different customers, regulatory data residency, or genuinely adversarial trust | Separate clusters, possibly separate cloud accounts |
| Latency-bound regional requirements or a hard compliance/data-sovereignty need | Multi-cluster architecture (Chapter 11) |

Reaching for hard multi-tenancy or multi-cluster before an actual need materializes multiplies your operational burden (N times the upgrade runbooks, N times the RBAC audits, N times the monitoring surface) for isolation nobody is actually asking for yet. Start at the lightest level that satisfies today's real requirement, and move up the ladder when a concrete need — not a hypothetical one — arrives.

---

## Only Adopt a Service Mesh for a Concrete, Current Problem

Chapter 7 covered what a service mesh actually provides: mTLS everywhere, fine-grained traffic management, rich service-to-service observability. It also covered the real cost: sidecar resource overhead on every Pod, added request latency, and a genuinely non-trivial new piece of infrastructure to operate and upgrade.

Before adopting a mesh, name the specific problem it solves for you *today*:

- "We need mTLS between services for a compliance requirement, and we don't want to build that into every application" — legitimate, concrete.
- "We need canary traffic splitting more precise than what Argo Rollouts + Ingress annotations already give us" — legitimate, concrete.
- "Everyone else in the industry uses Istio" — not a reason.

A mesh adopted for the second reason absorbs real operational complexity and latency for a benefit nobody can articulate past "best practice," which is exactly the situation where the mesh itself becomes the next incident's root cause.

---

## Everything in Git, Nothing Applied by Hand

Chapter 8 covered GitOps with Argo CD and Flux. The production discipline is applying this completely, not partially: every manifest, every policy, every namespace definition, every admission-control constraint — not just application Deployments.

```
platform-gitops-repo/
├── clusters/
│   └── prod/
│       ├── namespaces/          # namespace + default-deny NetworkPolicy + ResourceQuota, per team
│       ├── policies/            # Kyverno/Gatekeeper ClusterPolicies
│       ├── rbac/                # ClusterRoles, RoleBindings
│       └── platform-operators/  # cert-manager, external-secrets, ingress-nginx, etc.
└── apps/
    └── team-checkout/
        └── prod/                # application manifests, managed by the team
```

A cluster where infrastructure-adjacent config (RBAC, policies, namespace scaffolding) is still applied by hand "because it's a one-time platform thing" is a cluster with no audit trail, no review process, and no rollback path for exactly the changes with the widest blast radius. If it's worth having in the cluster, it's worth having in Git.

This also means the GitOps controller itself (Argo CD/Flux) and its own configuration — its `Application`/`Kustomization` objects, its RBAC, its notification settings — should be bootstrapped from Git too (the "app of apps" or "app of Kustomizations" pattern), so that even the tool enforcing GitOps doesn't become the one hand-maintained exception to its own rule.

---

## Automate and Regularly Test Backups and DR Drills

Chapters 9 and 10 covered etcd backup/restore and Velero-based application backup. An untested backup is not a backup — it's an unverified assumption that will be tested for the first time, under the worst possible conditions, during a real disaster.

```bash
# Automate this — don't rely on someone remembering to run it manually
etcdctl snapshot save /backups/etcd-snapshot-$(date +%Y%m%d).db

# Schedule Velero backups declaratively
velero schedule create daily-prod-backup \
  --schedule="0 2 * * *" \
  --include-namespaces=production
```

Run an actual restore — into a separate scratch cluster, on a recurring schedule (quarterly is a reasonable minimum for most teams) — and verify the restored workloads actually come up healthy with the data intact. A backup schedule with a green checkmark in a dashboard tells you the backup *job succeeded*; it tells you nothing about whether the backup is *restorable*, which is the only property that actually matters.

---

## Stay Within a Supported Version Window, With a Tested Upgrade Runbook

Chapter 9 covered the mechanics of `kubeadm upgrade` and node draining. The practice: treat version currency as an ongoing operational responsibility, not a fire drill that happens only when a cloud provider forces the issue.

- Know your Kubernetes distribution's supported version skew policy and plan upgrades on a regular cadence (roughly aligned with the ~4-month minor release cycle) rather than waiting until you're several versions behind and facing a forced upgrade.
- Maintain a written, rehearsed upgrade runbook — control plane first, then nodes in batches, with PodDisruptionBudgets respected and health checks between each step — so an upgrade is a routine, low-drama operation, not a novel project every time.
- Test upgrades against a staging cluster that mirrors production's Kubernetes version and installed CRDs/Operators first; API deprecations between minor versions are the most common source of upgrade-time surprises.

---

## Tune Autoscaling Using Real Observed Data

Chapter 12 covered VPA's `Off`/recommendation mode as a way to observe real resource usage without VPA actually applying anything. Use it before guessing:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-recommender
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"     # observe and recommend only — never apply
```

```bash
kubectl describe vpa api-recommender
# Recommendation section shows target/lower/upper bound CPU and memory
# based on actual historical usage — use these numbers to set
# resources.requests/limits deliberately, instead of copy-pasting
# whatever value another team used for an unrelated service
```

Guessed resource values are either wasteful (over-provisioned, inflating cluster cost) or dangerous (under-provisioned, causing the throttling and eviction failure modes covered in Chapters 12 and 13). Real usage data, observed over at least one full traffic cycle (including your highest-traffic period, not just a quiet week), is the only reliable input for setting these values correctly the first time.

---

## Treat Audit Logs as a First-Class Operational Signal

Chapter 13 covered configuring audit logging and using it for forensics. In practice, most organizations enable it, ship it somewhere, and then nobody looks at it again until an incident forces a search through it. Treat it instead the way you'd treat any other operational telemetry:

- Build a small number of standing dashboards from audit log data — top API request volume by identity, unusual `delete` activity against production namespaces, RBAC changes — the same way you'd build dashboards from metrics.
- Alert on genuinely anomalous patterns (a sudden spike in requests from a single ServiceAccount, a `ClusterRoleBinding` change outside of your GitOps pipeline) rather than treating the log purely as a passive archive.
- Review it as part of routine security posture reviews, not only as a forensic tool reached for after something has already gone wrong.

---

## Document and Rehearse Incident Runbooks

Chapter 13's cluster-wide failure playbooks (API server slowness, cluster-wide DNS failure, a `NotReady` node, mass evictions) are only useful during a real incident if the on-call engineer has seen them before it starts. Treat them the way a well-run organization treats a fire drill:

- Keep the playbooks in a runbook system that's reachable even when the primary observability stack is degraded (a paged engineer needs the runbook precisely when things are going wrong, not only when they're going well).
- Run a scheduled game day periodically — deliberately degrade a staging cluster's etcd disk I/O, or kill CoreDNS, and have on-call practice the diagnostic path for real.
- Update the runbook after every real incident with whatever the actual playbook was missing; a runbook that never changes after being used in anger isn't being taken seriously.

---

## Invest in Platform Self-Service

Every practice above adds a constraint application teams must operate within: least-privilege RBAC, mandatory policy checks, default-deny networking, mandatory GitOps. Enforced without a corresponding investment in making the *correct* path easy, these constraints become friction that teams learn to route around — requesting broad exceptions, finding workarounds, or simply avoiding the platform team's involvement altogether.

The fix is building self-service into the platform itself:

- **Good namespace provisioning** — a new team's namespace arrives pre-wired with the default-deny NetworkPolicy, a sensible ResourceQuota, and correct RBAC bindings already in place, via a single Git PR against a namespace template, not a ticket to the platform team.
- **Sane defaults via admission policy** — Kyverno/Gatekeeper rules that don't just reject bad configuration but, where possible, *mutate* incoming manifests to add sensible defaults (a missing `resources` block gets a reasonable default injected rather than the request being flatly rejected with no guidance).
- **GitOps-based app onboarding** — a documented, templated path for a team to get a new application from "code exists" to "running in production," entirely through Git, with no bespoke platform-team intervention required per app.

A platform team that enforces every practice in this chapter but makes each one painful to comply with will find teams working *around* the platform rather than *with* it. A platform team that makes compliance the path of least resistance gets the same security and reliability outcomes without the constant friction — and without needing to police it.

---

## Summary

Platform/cluster-admin best practices, chapter by chapter: least-privilege RBAC with regular `kubectl auth can-i` audits (Ch. 2); policy enforced as code via Gatekeeper/Kyverno rather than relying on PR review (Ch. 3); default-deny NetworkPolicies baked into namespace provisioning (Ch. 4); preferring an existing Operator over custom controllers for well-solved problems (Ch. 5); choosing the lightest isolation and multi-cluster model that matches your actual trust boundary rather than the most fashionable one (Ch. 6, 11); adopting a service mesh only for a concrete, named problem (Ch. 7); full GitOps coverage including infra-adjacent config, not just app manifests (Ch. 8); automated and *tested* backup/DR and a rehearsed, scheduled upgrade cadence (Ch. 9, 10); autoscaling tuned from VPA recommendation-mode data instead of guesses (Ch. 12); audit logs treated as a first-class operational signal with standing dashboards and alerts (Ch. 13); rehearsed incident runbooks for the cluster-wide failure scenarios; and platform self-service investment so that enforcing all of the above doesn't become a bottleneck teams learn to route around.

---

## Knowledge Check

1. Why is `kubectl auth can-i --list` a more reliable audit tool than reading the RBAC manifests themselves?
2. Give a concrete example of a rule that should be enforced by an admission controller rather than left to PR review, and explain why.
3. What determines whether a team should use namespace-based soft multi-tenancy versus hard multi-tenancy versus a separate cluster?
4. Why is a Velero backup schedule with a 100% success rate not sufficient evidence that disaster recovery will actually work?
5. What does running VPA in `Off` mode give you that guessing at resource requests/limits does not?
6. Why can enforcing every practice in this chapter still fail to improve platform security/reliability if self-service isn't also invested in?

---

## Hands-On Exercise

1. Pick a ServiceAccount in a cluster you control (or create one) and run `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<name>` against it. Compare the actual output against what you *expect* it to be able to do, based on its bound Roles — note any surprises.
2. Write a Kyverno or Gatekeeper policy (in `audit` mode first) that enforces one rule from this chapter (no `:latest` tags, mandatory resource limits, or no root containers) and validate it against a few existing manifests before ever switching it to `enforce`.
3. Design a namespace-provisioning template (a Kustomize base or a small Helm chart) that bundles a default-deny NetworkPolicy, a ResourceQuota, and a baseline RoleBinding together, so that applying one template gives a new team namespace all three by default.
4. Set up a VPA in `Off` mode against a real or sample Deployment, generate some load, and after at least a few hours, run `kubectl describe vpa` to see the recommendation — compare it against whatever resource values are currently set.

---

## Further Reading

- [Kubernetes Documentation — Using RBAC Authorization](https://kubernetes.io/docs/reference/access-control/rbac/)
- [Kyverno Policies Library](https://kyverno.io/policies/)
- [OperatorHub.io](https://operatorhub.io)
- [Kubernetes Documentation — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Velero Documentation — Disaster Recovery](https://velero.io/docs/main/disaster-case/)
- [Kubernetes Documentation — Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./13-auditing-and-troubleshooting-at-scale.md">← Previous: Auditing and Troubleshooting at Scale</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./15-common-mistakes.md">Next: Common Mistakes and Pitfalls →</a>
</div>
