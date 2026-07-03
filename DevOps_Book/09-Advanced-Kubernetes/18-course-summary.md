# Chapter 18 — Course Summary & Next Steps

## 18.1 What You've Learned

Congratulations on completing **Advanced Kubernetes** — Topic 9 of the DevOps Learning Path. You started this course able to deploy, expose, configure, and scale applications on a cluster someone else built and secured. You finish it able to be that someone — the platform/cluster-admin responsible for running Kubernetes safely for real teams and real production traffic.

| Chapter | What You Can Do Now |
|---------|----------------------|
| 01 Introduction | Explain the platform-engineer mindset and how it differs from an application deployer's, and place CKA/CKAD in context |
| 02 RBAC and Authentication | Design least-privilege access for users, teams, and workloads using Roles, ClusterRoles, and bindings |
| 03 Admission Control and Pod Security | Enforce security policy automatically via admission webhooks, Pod Security Standards, and OPA/Kyverno |
| 04 Network Policies | Implement zero-trust, default-deny Pod-to-Pod networking with explicit allow rules |
| 05 Custom Resources and Operators | Extend the Kubernetes API with CRDs and understand how Operators automate operational knowledge |
| 06 Multi-Tenancy | Design soft multi-tenant clusters that safely isolate multiple teams sharing one cluster |
| 07 Service Mesh | Explain what a service mesh solves (mTLS, traffic management, observability) and when you actually need one |
| 08 GitOps and Progressive Delivery | Build Argo CD pipelines with canary/blue-green rollouts via Argo Rollouts |
| 09 Cluster Administration and Upgrades | Safely upgrade a live cluster, drain nodes, and back up/restore etcd |
| 10 Backup and Disaster Recovery | Design and test Velero-based backup/DR strategies with real RTO/RPO numbers |
| 11 Multi-Cluster Architectures | Reason about fleet management and multi-cluster GitOps patterns |
| 12 Autoscaling and Performance Tuning | Tune HPA/VPA/Karpenter and diagnose cluster-scale performance problems |
| 13 Auditing and Troubleshooting at Scale | Use audit logs, `kubectl debug`, and node-level tooling to diagnose cluster-wide failures |
| 14 Best Practices | Apply production-grade cluster administration patterns |
| 15 Common Mistakes | Recognize and avoid the most frequent advanced Kubernetes/platform mistakes |
| 16 Projects | Locked down a namespace, built a GitOps canary pipeline, operated a database Operator with DR drills, and designed a multi-cluster policy-enforced platform |
| 17 Interview Preparation | Answer platform-engineering-level foundational, architectural, scenario, and system-design questions with confidence |

---

## 18.1.1 The Mental Model, in One Paragraph

Kubernetes Basics taught you that Kubernetes is a declarative, reconciling system. This course adds the other half of that idea: everything a platform team does is *also* reconciliation, just at a different layer. RBAC reconciles "who is allowed to do what" against every incoming request. Admission controllers and Kyverno/OPA reconcile "does this object meet policy" before it's ever persisted. NetworkPolicies reconcile "which packets are allowed to flow" continuously as Pods come and go. An Operator reconciles a whole domain of operational knowledge — replica health, failover, backup schedules — that used to live in a human's runbook. Argo CD reconciles the entire cluster against Git. Once you see platform engineering as "more reconciliation loops, at higher-stakes layers," the chapters in this course stop being a list of unrelated tools and become one consistent discipline: decide the desired state, write it down declaratively, and let a controller enforce it forever instead of trusting a human to remember.

---

## 18.2 Completion Checklist

```
Access Control & Policy:
  [ ] Can write a least-privilege Role/ClusterRole from scratch for a real workload
  [ ] Can debug an RBAC denial using kubectl auth can-i and binding inspection
  [ ] Can explain the full admission control pipeline in order (mutating → PSA → validating)
  [ ] Can enforce Pod Security Standards on a namespace and know when to use baseline vs restricted
  [ ] Can write a Kyverno/OPA policy enforcing a custom organizational rule

Networking & Isolation:
  [ ] Can implement default-deny NetworkPolicy for both ingress and egress
  [ ] Can write explicit allow rules scoped to real traffic paths, not overly broad selectors
  [ ] Can diagnose "NetworkPolicy applied but traffic not blocked" (CNI support check first)
  [ ] Can explain what problem a service mesh solves that Ingress + Services do not

Extensibility & Multi-Tenancy:
  [ ] Can explain the relationship between a CRD, a Custom Resource, and an Operator
  [ ] Can install and operate a real Operator (e.g., CloudNativePG) end-to-end
  [ ] Can design a soft multi-tenancy model for multiple teams on one cluster
  [ ] Can articulate when soft multi-tenancy is insufficient and hard multi-tenancy is needed

Delivery & Operations:
  [ ] Can set up an Argo CD Application syncing a Git repo to a cluster
  [ ] Can configure an Argo Rollouts canary strategy with pause/analysis steps
  [ ] Can explain GitOps drift, selfHeal, and why kubectl edit against a GitOps resource is dangerous
  [ ] Can tune HPA/VPA without letting them fight each other
  [ ] Can explain Karpenter vs Cluster Autoscaler and when each is the right choice

Cluster Lifecycle:
  [ ] Can safely cordon/drain/upgrade a node while respecting PodDisruptionBudgets
  [ ] Can take and restore an etcd snapshot
  [ ] Can design a backup/DR strategy with explicit RTO/RPO numbers
  [ ] Can run a full Velero namespace restore drill
  [ ] Can read an audit log to reconstruct what happened during an incident

Projects Completed:
  [ ] Project 1: Namespace locked down with quota, default-deny NetworkPolicy, scoped RBAC, and PSS
  [ ] Project 2: GitOps pipeline with an Argo Rollouts canary triggered by a Git commit
  [ ] Project 3: Database Operator installed with automated backups, plus a full Velero restore drill
  [ ] Project 4: Multi-cluster, policy-enforced platform with a documented incident runbook
```

If any row above still feels shaky, that's a signal, not a failure — go back to the relevant chapter and redo the hands-on exercise before moving to Topic 10. Platform engineering skills are trusted in an interview (and in production) only once they're muscle memory, not just something you've read about once.

---

## 18.3 kubectl / CLI Quick Reference

```bash
# ── RBAC ─────────────────────────────────────────────────────────────────
kubectl auth can-i <verb> <resource> --as=<user>                  # check permission
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa> -n <ns>
kubectl auth can-i --list --as=<user> -n <ns>                      # list everything allowed
kubectl create role <name> --verb=get,list,watch --resource=pods -n <ns>
kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa> -n <ns>
kubectl create clusterrolebinding <name> --clusterrole=<cr> --user=<user>

# ── Admission / Policy ───────────────────────────────────────────────────
kubectl label namespace <ns> pod-security.kubernetes.io/enforce=restricted
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get clusterpolicy                                          # Kyverno policies
kubectl get constrainttemplate,constraint                          # OPA/Gatekeeper

# ── NetworkPolicy ────────────────────────────────────────────────────────
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <ns>
kubectl run probe --rm -it --image=busybox -n <ns> --command -- wget -qO- <target>

# ── Debugging ─────────────────────────────────────────────────────────────
kubectl debug node/<node> -it --image=busybox                      # node-level debug shell
kubectl debug <pod> -it --image=busybox --target=<container>       # ephemeral debug container
kubectl get events --sort-by=.lastTimestamp -A
crictl ps                                                           # container runtime state (on-node)
crictl logs <container-id>
etcdctl snapshot save /backup/etcd-snapshot.db                     # etcd backup
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
etcdctl endpoint health --cluster                                   # quorum/health check

# ── Cluster Ops ───────────────────────────────────────────────────────────
kubectl cordon <node>                                               # stop new scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data     # evict, respecting PDBs
kubectl uncordon <node>                                              # allow scheduling again
kubectl get pdb -A                                                   # check disruption budgets
kubeadm upgrade plan / kubeadm upgrade apply <version>              # control-plane upgrade

# ── Autoscaling ───────────────────────────────────────────────────────────
kubectl get hpa,vpa -A
kubectl top nodes
kubectl top pods -A

# ── GitOps / Tools ────────────────────────────────────────────────────────
argocd app sync <app>                                                # force immediate sync
argocd app diff <app>                                                # show drift vs Git
argocd app history <app>                                             # deployment audit trail
kubectl argo rollouts get rollout <name> --watch                    # watch a canary progress
kubectl argo rollouts promote <name>                                 # manual promotion gate
kubectl argo rollouts abort <name>                                   # abort and roll back
velero backup create <name> --include-namespaces <ns> --wait
velero restore create --from-backup <name> --namespace-mappings <old>:<new>
helm install / helm upgrade / kustomize build .
```

---

## 18.4 Kubernetes Advanced YAML Quick Reference

```yaml
# Role + RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: example-role, namespace: example-ns }
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: example-binding, namespace: example-ns }
subjects:
  - kind: ServiceAccount
    name: example-sa
    namespace: example-ns
roleRef: { kind: Role, name: example-role, apiGroup: rbac.authorization.k8s.io }

---
# NetworkPolicy (default-deny + explicit allow)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: example-ns }
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend-to-backend, namespace: example-ns }
spec:
  podSelector: { matchLabels: { app: backend } }
  policyTypes: ["Ingress"]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: frontend } } }]
      ports: [{ protocol: TCP, port: 8080 }]

---
# CustomResourceDefinition + Custom Resource
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: widgets.example.com }
spec:
  group: example.com
  scope: Namespaced
  names: { plural: widgets, singular: widget, kind: Widget }
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                size: { type: integer }
---
apiVersion: example.com/v1
kind: Widget
metadata: { name: my-widget }
spec:
  size: 3

---
# Argo CD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: example-app, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/example-gitops.git
    targetRevision: main
    path: apps/example
  destination:
    server: https://kubernetes.default.svc
    namespace: example-ns
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: ["CreateNamespace=true"]

---
# Argo Rollouts canary steps block
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: example-rollout }
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 2m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

---

## 18.5 What's Next: Topic 10 — Monitoring & Logging

You now know how to secure, extend, deploy to, and operate a Kubernetes cluster. What this course deliberately did not cover in depth is *observability* — actually seeing what your cluster and applications are doing in real time, alerting on problems before users report them, and defining what "healthy" even means for a service. That's Topic 10.

Topic 10 covers:

- **Prometheus** — the de facto metrics collection and time-series database for Kubernetes, and the query language (PromQL) to ask questions of that data
- **Grafana** — building dashboards on top of Prometheus (and other data sources) to visualize cluster and application health
- **ELK / Loki** — centralized log aggregation, so `kubectl logs` on one Pod at a time stops being your only option once you have dozens of replicas across multiple clusters
- **Alerting** — turning metrics thresholds into pages, and doing it without drowning the on-call engineer in noise
- **SLOs (Service Level Objectives)** — defining measurable reliability targets and error budgets, the discipline that turns "the app seems fine" into a number you can actually manage against

A useful way to frame the jump: many of Topic 9's chapters *assumed* observability tooling would eventually exist — Chapter 8's Argo Rollouts analysis step queries Prometheus for error rate, Chapter 12's autoscaling tuning is only as good as the metrics feeding it, and Chapter 13's troubleshooting-at-scale is far easier with centralized logs than with `kubectl logs` one Pod at a time. Topic 10 is where those assumptions become real.

| What you learned in this course | How it maps to Monitoring & Logging |
|---|---|
| Installing and operating an Operator (Chapter 5) | Prometheus itself is typically installed via the Prometheus Operator, using the exact same CRD + controller pattern you already know |
| GitOps with Argo CD (Chapter 8) | Grafana dashboards and Prometheus rules are managed as code in the same GitOps repo, versioned and reviewed like any other manifest |
| Autoscaling tuning (Chapter 12) | HPA custom metrics (request rate, queue depth) are sourced directly from Prometheus via the metrics adapter |
| Audit logging and troubleshooting (Chapter 13) | Centralized logging (Loki/ELK) is the natural extension of the same "where do I look during an incident" instinct, just at cluster-wide scale |
| RBAC (Chapter 2) | Grafana/Prometheus access itself needs the same least-privilege thinking — not every team should see every other team's dashboards |

---

## 18.6 The Full Picture

```
Topics 1–4:   Core foundations
  Linux, Networking, Git, Docker

Topics 5–7:   Cloud and automation
  CI/CD Pipelines, AWS, Terraform

Topics 8–9:   Container orchestration          (YOU ARE HERE — finishing 9)
  Kubernetes Basics, Advanced Kubernetes

Topics 10–11: Production operations
  Monitoring & Logging, Security (DevSecOps)
```

You are now 9 of 11 courses complete. Everything from here builds directly on this course: Prometheus and Grafana will be deployed onto the RBAC-, NetworkPolicy-, and admission-control-governed clusters you now know how to build; the GitOps pipelines from Chapter 8 are how their configuration gets deployed and versioned; and Topic 11's security hardening assumes you're already comfortable with the policy-enforcement tooling (Kyverno/OPA, Pod Security Standards) this course introduced. If Chapters 1–17 felt solid, Topic 10 will feel like adding eyes to a platform you already know how to run.

---

## 18.7 Recommended Resources

- **Kubernetes official documentation — RBAC, NetworkPolicy, and CRDs** (kubernetes.io/docs/reference/access-authn-authz, kubernetes.io/docs/concepts/services-networking/network-policies, kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources) — the authoritative reference for every primitive this course builds on
- **"Kubernetes Patterns"** by Bilgin Ibryam and Roland Huß — the best available catalog of the design patterns behind Operators, multi-container Pods, and controller behavior; directly deepens Chapter 5
- **CNCF Cloud Native Landscape** (landscape.cncf.io) — now that you've used real projects from it (Argo, Kyverno, Velero), it's worth revisiting to see how they fit into the broader ecosystem, including what Topic 10 and 11 will introduce next
- **Argo Project documentation** (argo-cd.readthedocs.io, argoproj.github.io/argo-rollouts) — the canonical reference for everything in Chapter 8 and Project 2, including sync options and analysis providers this course didn't have room to cover exhaustively
- **Open Policy Agent / Gatekeeper docs** (open-policy-agent.github.io/gatekeeper) **and Kyverno docs** (kyverno.io/docs) — go deeper on policy authoring than Chapter 3 could, including testing policies before enforcing them
- **CKA (Certified Kubernetes Administrator) certification page** (cncf.io/certification/cka) — the natural certification target after this course; its curriculum overlaps heavily with Chapters 2, 4, 9, 10, and 13

None of these replace hands-on repetition. Keep the `kind` clusters from this course's projects around, break things in them deliberately, and use the CKA exam objectives as a checklist of what to practice next — the fastest way to retain everything in this course is to keep operating a real (even if small) cluster the way a platform team would.

---

## 18.8 Progress Tracker

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | Complete |
| 4 | [Docker](../04-Docker/00-index.md) | Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | Complete |
| 6 | [Cloud Fundamentals (AWS)](../06-Cloud-Fundamentals-AWS/00-index.md) | Complete |
| 7 | [Infrastructure as Code (Terraform)](../07-Infrastructure-as-Code-Terraform/00-index.md) | Complete |
| 8 | [Kubernetes Basics](../08-Kubernetes-Basics/00-index.md) | Complete |
| 9 | Advanced Kubernetes | Complete — you are here |
| 10 | Monitoring & Logging | Coming soon |
| 11 | Security (DevSecOps) | Coming soon |

Continue to: **[Monitoring & Logging](../10-Monitoring-and-Logging/00-index.md)**

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./17-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../10-Monitoring-and-Logging/00-index.md">Next: Monitoring & Logging →</a>
</div>
