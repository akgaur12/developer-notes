# Chapter 16 — Hands-On Projects

## Learning Objectives

By the end of this chapter you will have:

- Locked down a real namespace with the full soft-multi-tenancy pattern: quotas, default-deny networking, scoped RBAC, and Pod Security Standards
- Built a GitOps pipeline with Argo CD and configured an automated canary rollout with Argo Rollouts
- Installed and operated a real database Operator, configured automated backups, and run a full disaster-recovery drill with Velero
- Designed a multi-cluster, policy-enforced platform suitable as a portfolio piece for a platform engineer or senior DevOps/SRE role

---

## Project 1 — Lock Down a Namespace (Beginner)

**Goal:** Take an existing namespace running a multi-tier app (assume the `shop` application from Topic 8's Project 2 — frontend, backend, Postgres) and apply the full soft-multi-tenancy pattern from Chapter 6: a ResourceQuota, default-deny NetworkPolicies plus explicit allows for real traffic paths, a Role+RoleBinding scoping a team ServiceAccount to just this namespace, and a Pod Security Standard label.

**What you'll build:** A `shop-prod` namespace where the `shop-team` ServiceAccount can manage the app's own resources and nothing else, where Pods can only talk to the Pods they legitimately need to, and where every Pod is required to run as non-root with no privilege escalation.

**Architecture:**

```
Namespace: shop-prod
  ├── PodSecurity label: restricted
  ├── ResourceQuota: caps total CPU/memory/objects
  ├── NetworkPolicy: default-deny-all (ingress + egress)
  ├── NetworkPolicy: allow frontend → backend
  ├── NetworkPolicy: allow backend → postgres
  ├── NetworkPolicy: allow ingress-controller → frontend
  ├── NetworkPolicy: allow DNS egress (kube-system, port 53)
  ├── ServiceAccount: shop-team
  ├── Role: shop-team-role (namespaced, scoped to shop-prod objects)
  └── RoleBinding: shop-team-role → shop-team ServiceAccount
```

**`manifests/namespace.yaml`** (the Pod Security Standard label from Chapter 3 — `restricted` is the strictest tier; use `baseline` if the existing images can't yet run fully non-root):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
```

**`manifests/quota.yaml`** (Chapter 6):

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shop-prod-quota
  namespace: shop-prod
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "30"
    persistentvolumeclaims: "5"
    services.loadbalancers: "0"   # this team never provisions its own LB
```

**`manifests/networkpolicy-default-deny.yaml`** (Chapter 4 — the foundation of zero-trust networking; without this, every "allow" rule below is meaningless because everything is already allowed):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shop-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**`manifests/networkpolicy-allow.yaml`** (explicit allows for the app's real traffic paths — everything else stays blocked):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: shop-prod
spec:
  podSelector:
    matchLabels: { app: shop-frontend }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: ingress-nginx }
      ports:
        - protocol: TCP
          port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: shop-prod
spec:
  podSelector:
    matchLabels: { app: shop-backend }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: shop-frontend }
      ports:
        - protocol: TCP
          port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: shop-prod
spec:
  podSelector:
    matchLabels: { app: postgres }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: shop-backend }
      ports:
        - protocol: TCP
          port: 5432
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: shop-prod
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: kube-system }
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    - to:
        - podSelector:
            matchLabels: { app: shop-backend }
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - podSelector:
            matchLabels: { app: postgres }
      ports:
        - protocol: TCP
          port: 5432
```

**`manifests/rbac.yaml`** (Chapter 2 — a namespaced Role, never a ClusterRole, because this team owns exactly one namespace):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shop-team
  namespace: shop-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shop-team-role
  namespace: shop-prod
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "deployments", "services", "configmaps", "jobs", "cronjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]        # read-only on secrets — no create/delete from CI
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "create"]      # debugging access, not cluster-wide
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shop-team-binding
  namespace: shop-prod
subjects:
  - kind: ServiceAccount
    name: shop-team
    namespace: shop-prod
roleRef:
  kind: Role
  name: shop-team-role
  apiGroup: rbac.authorization.k8s.io
```

**Implementation steps:**

1. Create the namespace with the PSS label applied at creation time, then apply the quota, NetworkPolicies, and RBAC: `kubectl apply -f manifests/`.
2. Move (or redeploy) the `shop` application's Deployments/Services into `shop-prod`, ensuring every container sets `securityContext.runAsNonRoot: true`, `allowPrivilegeEscalation: false`, and drops all capabilities — required for the `restricted` PSS tier to admit the Pods at all.
3. Confirm the quota is being tracked: `kubectl describe resourcequota shop-prod-quota -n shop-prod`.
4. Verify the ServiceAccount's actual permissions match intent:
   ```bash
   kubectl auth can-i create deployments --as=system:serviceaccount:shop-prod:shop-team -n shop-prod   # yes
   kubectl auth can-i delete namespaces --as=system:serviceaccount:shop-prod:shop-team                # no
   kubectl auth can-i create deployments --as=system:serviceaccount:shop-prod:shop-team -n kube-system # no
   kubectl auth can-i create secrets --as=system:serviceaccount:shop-prod:shop-team -n shop-prod       # no
   ```
5. Verify the NetworkPolicy actually blocks what it should, with a disposable test Pod launched *inside* the namespace but not matching any allow rule's source:
   ```bash
   kubectl run netpol-test --rm -it --image=busybox -n shop-prod --command -- \
     wget -qO- --timeout=3 http://postgres.shop-prod.svc.cluster.local:5432
   # should time out — this Pod has no label matching allow-backend-to-postgres's source selector
   ```
   Then confirm the same request succeeds from a Pod carrying the `app: shop-backend` label, proving the policy is scoped correctly rather than accidentally blocking everything.

**Success criteria:** `kubectl auth can-i` confirms the `shop-team` ServiceAccount can manage its own namespace's workloads but cannot touch Secrets writes, other namespaces, or cluster-scoped resources; the default-deny test Pod's connection to Postgres times out while the legitimately-labeled backend Pod's connection succeeds; every Pod in `shop-prod` passes admission under the `restricted` Pod Security Standard with zero warnings.

---

## Project 2 — GitOps Pipeline with Progressive Delivery (Intermediate)

**Goal:** Replace manual `kubectl apply`/`helm upgrade` deployment of the `shop` app with Argo CD watching a Git repository, then layer an Argo Rollouts canary strategy onto the backend service so new versions roll out gradually with an automated or manual promotion gate, per Chapter 8.

**Repo structure** (a separate Git repo from application source — the GitOps principle from Chapter 8 that the deployment repo is its own source of truth):

```
shop-gitops/
├── apps/
│   └── shop/
│       ├── namespace.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── backend-rollout.yaml      # Rollout, not Deployment
│       ├── backend-service.yaml
│       ├── backend-analysistemplate.yaml
│       ├── postgres-statefulset.yaml
│       └── ingress.yaml
└── argocd/
    └── shop-application.yaml
```

**`argocd/shop-application.yaml`** (the Argo CD `Application` object — this is what turns Git into the source of truth):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/shop-gitops.git
    targetRevision: main
    path: apps/shop
  destination:
    server: https://kubernetes.default.svc
    namespace: shop-prod
  syncPolicy:
    automated:
      prune: true        # remove resources deleted from Git
      selfHeal: true      # revert manual kubectl edits back to Git's version
    syncOptions:
      - CreateNamespace=true
```

**`apps/shop/backend-rollout.yaml`** (Argo Rollouts replaces the backend's Deployment with a `Rollout` — same Pod template, but with a canary strategy):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: shop-backend
  namespace: shop-prod
spec:
  replicas: 5
  selector:
    matchLabels: { app: shop-backend }
  template:
    metadata:
      labels: { app: shop-backend }
    spec:
      containers:
        - name: backend
          image: myorg/shop-backend:1.6.0
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits: { cpu: 500m, memory: 256Mi }
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
  strategy:
    canary:
      stableService: shop-backend-stable
      canaryService: shop-backend-canary
      steps:
        - setWeight: 20
        - pause: { duration: 2m }
        - analysis:
            templates:
              - templateName: backend-error-rate
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

**`apps/shop/backend-analysistemplate.yaml`** (automated promotion gate — if Prometheus/Chapter-10-style metrics aren't available yet, delete the `analysis` step above and rely on the two `pause` steps as manual gates, promoted with `kubectl argo rollouts promote shop-backend`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: backend-error-rate
  namespace: shop-prod
spec:
  metrics:
    - name: error-rate
      interval: 30s
      count: 4
      successCondition: result < 0.05
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus-server.monitoring.svc.cluster.local
          query: |
            sum(rate(http_requests_total{job="shop-backend",status=~"5.."}[2m]))
            /
            sum(rate(http_requests_total{job="shop-backend"}[2m]))
```

**Implementation steps:**

1. Install Argo CD and Argo Rollouts into the cluster (both ship as standard manifests — `kubectl create namespace argocd && kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`, and similarly for `argo-rollouts`), plus the `kubectl argo rollouts` CLI plugin for visibility.
2. Push the `shop-gitops` repo structure above to a real Git remote.
3. Apply the Argo CD `Application` object: `kubectl apply -f argocd/shop-application.yaml -n argocd`. Confirm Argo CD picks it up: `argocd app get shop` or `kubectl get application shop -n argocd`.
4. Confirm the app reaches `Synced`/`Healthy` and every manifest in Git now exists live in `shop-prod`.
5. Make a real change: bump `backend-rollout.yaml`'s image tag to `1.6.1` and push to `main`.
6. Watch Argo CD detect the drift and sync automatically (within its default 3-minute poll, or trigger instantly with `argocd app sync shop`), and watch Argo Rollouts execute the canary steps:
   ```bash
   kubectl argo rollouts get rollout shop-backend -n shop-prod --watch
   ```
7. Prove `selfHeal` works: manually `kubectl scale` the backend Rollout or `kubectl edit` a label, and watch Argo CD revert it back to what Git says within seconds — this is the core GitOps guarantee from Chapter 8, that the cluster can never permanently drift from Git.

**Success criteria:** A Git commit changing the backend's image tag triggers an automatic canary rollout visible in the Argo Rollouts CLI/dashboard, progressing through 20% → pause → 50% → pause → 100% either via the automated analysis step or manual promotion; a manual `kubectl` edit to any Argo-CD-managed resource is reverted automatically; `argocd app history shop` shows a clean audit trail of every deployed revision tied to a Git commit SHA.

---

## Project 3 — Install and Operate a Database Operator (Advanced)

**Goal:** Replace the hand-rolled Postgres StatefulSet+PVC from Topic 8's Project 2 with a real Operator — CloudNativePG — configured for automated backups to object storage, per Chapter 5's worked example. Separately, install Velero (Chapter 10) to back up the rest of the `shop-prod` namespace, then run a full restore drill for both a database-level failure and a whole-namespace failure.

**Why this matters:** A hand-rolled StatefulSet gives you replicas and storage, but *nothing* about failover, point-in-time recovery, or safe minor-version upgrades — that operational knowledge is exactly what an Operator encodes into a controller, per Chapter 5.

**Step 1 — Install the CloudNativePG Operator:**

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.23/releases/cnpg-1.23.1.yaml
kubectl get pods -n cnpg-system   # confirm the operator controller is Running
```

**`manifests/postgres-cluster.yaml`** (the Custom Resource — this single object replaces the StatefulSet, PVC, Service, and backup CronJob from Topic 8 entirely):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: shop-db
  namespace: shop-prod
spec:
  instances: 3                     # 1 primary + 2 replicas, automatic failover
  imageName: ghcr.io/cloudnative-pg/postgresql:16.2

  storage:
    size: 10Gi
    storageClass: standard

  bootstrap:
    initdb:
      database: shop
      owner: shopapp
      secret:
        name: shop-db-credentials

  backup:
    barmanObjectStore:
      destinationPath: "s3://shop-db-backups/cnpg"
      s3Credentials:
        accessKeyId:
          name: backup-s3-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-s3-credentials
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
    retentionPolicy: "30d"
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: shop-db-nightly
  namespace: shop-prod
spec:
  schedule: "0 2 * * *"
  cluster:
    name: shop-db
```

**Step 2 — Migrate the backend to point at the Operator's Service** (CNPG creates `shop-db-rw` for read/write and `shop-db-ro` for read replicas automatically — no more hand-written headless Service):

```yaml
# backend-configmap.yaml — update DB_HOST only
data:
  DB_HOST: "shop-db-rw.shop-prod.svc.cluster.local"
```

**Step 3 — Install Velero for namespace-level backup** (Chapter 10 — covers everything the database Operator doesn't: Deployments, ConfigMaps, RBAC objects, the NetworkPolicies from Project 1):

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.2 \
  --bucket shop-namespace-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./velero-credentials
```

**Implementation steps:**

1. Install the Operator and apply the `Cluster` CR; watch it provision the primary and replicas: `kubectl get cluster shop-db -n shop-prod -w` and `kubectl get pods -n shop-prod -l cnpg.io/cluster=shop-db`.
2. Confirm automated backups are actually landing in object storage: `kubectl get scheduledbackup,backup -n shop-prod`.
3. **Drill 1 — simulated data loss inside the database:** connect via `kubectl cnpg psql shop-db -n shop-prod` and drop a table on purpose. Restore into a **new** Cluster from the most recent backup rather than overwriting the live one:
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: shop-db-restored
     namespace: shop-prod
   spec:
     instances: 1
     storage: { size: 10Gi, storageClass: standard }
     bootstrap:
       recovery:
         source: shop-db
     externalClusters:
       - name: shop-db
         barmanObjectStore:
           destinationPath: "s3://shop-db-backups/cnpg"
           s3Credentials:
             accessKeyId: { name: backup-s3-credentials, key: ACCESS_KEY_ID }
             secretAccessKey: { name: backup-s3-credentials, key: SECRET_ACCESS_KEY }
   ```
   Confirm the dropped table exists again in `shop-db-restored`, then point a scratch copy of the backend at it to verify application-level correctness, not just "the SQL file restored."
4. **Drill 2 — simulated whole-namespace loss:** take a Velero backup, then delete the entire `shop-prod` namespace, then restore into a fresh namespace:
   ```bash
   velero backup create shop-prod-full --include-namespaces shop-prod --wait
   kubectl delete namespace shop-prod
   velero restore create --from-backup shop-prod-full --namespace-mappings shop-prod:shop-prod-restored --wait
   kubectl get all -n shop-prod-restored
   ```
5. Document, in a runbook committed alongside the manifests, the exact recovery time observed for each drill — this becomes real RTO data instead of a guess (Chapter 10).

**Success criteria:** The dropped-table drill is fully recovered with correct data via the Operator's own backup/recovery mechanism, without any manual `pg_restore` scripting; the full-namespace-deletion drill is fully recovered via Velero into a working namespace with the application reachable again; both drills have a documented, measured recovery time.

---

## Project 4 — Multi-Cluster, Policy-Enforced Platform (Production-Grade Capstone)

**Goal:** Design and stand up a portfolio-quality platform spanning two clusters, managed via GitOps fleet patterns, with cluster-wide policy enforcement, per-team RBAC, default-deny networking everywhere, audit logging, and a written incident runbook. This project does not require real multi-region cloud infrastructure — two local `kind` clusters simulate the fleet affordably — but every manifest and config is real and reusable against real clusters later.

**Architecture:**

```
                    ┌────────────────────────┐
                    │   Git repo (fleet)      │
                    │  clusters/prod-east/    │
                    │  clusters/prod-west/    │
                    │  base/ (shared policy)  │
                    └───────────┬─────────────┘
                                │ ApplicationSet
                    ┌───────────┴─────────────┐
                    ▼                          ▼
          kind cluster: prod-east    kind cluster: prod-west
          ├── Kyverno policies       ├── Kyverno policies
          ├── RBAC per team          ├── RBAC per team
          ├── default-deny NetPol    ├── default-deny NetPol
          ├── audit logging          ├── audit logging
          └── shop app (Argo CD)     └── shop app (Argo CD)
```

**Step 1 — Two local clusters simulating the fleet:**

```bash
kind create cluster --name prod-east
kind create cluster --name prod-west
kubectl config get-contexts    # confirm both are reachable
```

**Step 2 — Fleet-wide GitOps via an Argo CD `ApplicationSet`** (Chapter 8/11 — one manifest deploys the app to every cluster registered with Argo CD, rather than hand-maintaining N `Application` objects):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: shop-fleet
  namespace: argocd
spec:
  generators:
    - clusters: {}          # auto-discovers every cluster registered via `argocd cluster add`
  template:
    metadata:
      name: "shop-{{name}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/yourorg/shop-gitops.git
        targetRevision: main
        path: apps/shop
      destination:
        server: "{{server}}"
        namespace: shop-prod
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: ["CreateNamespace=true"]
```

Register both clusters with the Argo CD instance running in `prod-east` (treating it as the hub cluster): `argocd cluster add kind-prod-east` and `argocd cluster add kind-prod-west`.

**Step 3 — Kyverno cluster-wide policies** (Chapter 3 — enforced on *every* namespace in *every* cluster via the same GitOps repo, not opt-in per team):

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-resources
      match:
        any:
          - resources: { kinds: ["Pod"] }
      validate:
        message: "CPU and memory limits are required on every container."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-latest
      match:
        any:
          - resources: { kinds: ["Pod"] }
      validate:
        message: "Images must be pinned to a specific tag, not :latest."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-non-root
      match:
        any:
          - resources: { kinds: ["Pod"] }
      validate:
        message: "Containers must not run as root."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
```

**Step 4 — Per-team RBAC and default-deny networking**, applied identically on both clusters via the same `base/` directory in the fleet repo — reuse the `Role`/`RoleBinding` pattern and the default-deny `NetworkPolicy` from Project 1 verbatim, parametrized per team namespace.

**Step 5 — Audit logging** (Chapter 13), enabled on each cluster's API server with a policy that captures at least all writes to RBAC and Secrets objects at the `RequestResponse` level and everything else at `Metadata`:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
      - group: ""
        resources: ["secrets"]
  - level: Metadata
    omitStages: ["RequestReceived"]
```

**Step 6 — Incident runbook** (a Markdown file committed to the fleet repo, covering at least two failure scenarios from Chapter 13, e.g.):

```
## Runbook: RBAC denies a deploy that should be allowed
1. Reproduce with: kubectl auth can-i <verb> <resource> --as=<sa> -n <ns>
2. List all bindings touching the SA: kubectl get rolebindings,clusterrolebindings -A \
   -o json | jq '.items[] | select(.subjects[]?.name=="<sa>")'
3. Diff the Role's rules against the attempted action; patch the Role in Git,
   never with `kubectl edit` directly against the cluster
4. Open a PR, let Argo CD sync it, re-run step 1 to confirm

## Runbook: A cluster fails to receive fleet updates (ApplicationSet drift)
1. Check cluster registration: argocd cluster list
2. Check ApplicationSet controller logs: kubectl logs -n argocd \
   deploy/argocd-applicationset-controller
3. Confirm network reachability from the hub cluster's Argo CD to the
   target cluster's API server (a common failure mode with kind clusters
   on the same Docker network — check for port/DNS conflicts)
4. Force a manual reconcile: kubectl delete application shop-<cluster> -n argocd
   (ApplicationSet regenerates it on its next reconcile loop)
```

**Implementation steps:**

1. Stand up both `kind` clusters and install Argo CD on `prod-east` as the hub.
2. Register `prod-west` as a remote cluster in Argo CD.
3. Install Kyverno on both clusters (via the `ApplicationSet` itself, targeting a `platform` path that includes the ClusterPolicies, RBAC, and NetworkPolicies — "policy as GitOps," not a one-off `kubectl apply`).
4. Deploy the `shop` app fleet-wide via the `ApplicationSet` from Step 2.
5. Prove policy enforcement works: attempt to deploy a Pod with `image: myorg/shop-backend:latest` and confirm Kyverno rejects it on both clusters.
6. Prove RBAC and NetworkPolicy isolation with the same `kubectl auth can-i` and blocked-traffic tests from Project 1, run against both clusters.
7. Write the runbook and commit it to the fleet repo alongside the manifests.
8. Push the entire repo (fleet config, policies, RBAC, NetworkPolicies, runbook) to a public GitHub repository with a README explaining the architecture — this is the artifact you link from a resume or portfolio.

**Success criteria:** The same application, RBAC model, and policy set are live and enforced identically on both clusters, driven entirely from one Git repo; a policy-violating manifest (missing limits, `:latest` tag, or root user) is rejected by Kyverno on both clusters; `kubectl auth can-i` and NetworkPolicy tests pass identically on both clusters; the repo is public, documented, and includes a runbook covering at least two real incident scenarios — this is explicitly a "I ran a real platform" project demonstrating readiness for a platform engineer or senior DevOps/SRE role.

---

## Summary

These four projects build on each other in a deliberate progression: Project 1 proves you can lock down a single namespace the way a responsible tenant should, Project 2 automates delivery into that namespace the way a platform team does, Project 3 replaces hand-rolled operational toil with a real Operator and proves you can actually recover from disaster, and Project 4 stitches every earlier chapter's skill into one coherent, policy-enforced, multi-cluster platform worth putting on a resume.

| Project | Level | Approx Time | Key Skills |
|---------|-------|-------------|------------|
| 1 — Lock Down a Namespace | Beginner | 2–3 hours | ResourceQuota, default-deny NetworkPolicy, scoped RBAC, Pod Security Standards |
| 2 — GitOps + Progressive Delivery | Intermediate | 6–8 hours | Argo CD Application, Argo Rollouts canary, automated/manual promotion gates |
| 3 — Database Operator + DR Drill | Advanced | 8–10 hours | CloudNativePG Operator, automated backups, Velero namespace restore |
| 4 — Multi-Cluster Policy Platform | Advanced/Capstone | 12–16 hours | ApplicationSet fleet GitOps, Kyverno, cross-cluster RBAC/NetworkPolicy, audit logging, runbooks |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-common-mistakes.md">← Previous: Common Mistakes and Pitfalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./17-interview-preparation.md">Next: Interview Preparation →</a>
</div>
