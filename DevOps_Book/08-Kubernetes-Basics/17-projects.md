# Chapter 17 — Hands-On Projects

## Learning Objectives

By the end of this chapter you will have:

- Deployed a stateless application to a local `kind` cluster with self-healing verified
- Built a complete multi-tier application (frontend, backend API, database) with Ingress routing
- Packaged that application as a reusable Helm chart with autoscaling and scheduled backups
- Deployed a production-grade version of the stack to a real managed cluster with TLS and quotas

---

## Project 1 — Deploy Your First App (Beginner)

**Goal:** Get a stateless web application running on a local Kubernetes cluster, verify it's reachable, and prove self-healing works.

**What you'll build:** An nginx Deployment with 3 replicas behind a Service, running on a local `kind` cluster.

```bash
# Create a local cluster (from Chapter 3)
kind create cluster --name project1
kubectl cluster-info --context kind-project1
```

**`manifests/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
  labels:
    app: hello-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
        - name: nginx
          image: nginxdemos/hello:0.5
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 3
          livenessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 10
```

**`manifests/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web
spec:
  type: NodePort
  selector:
    app: hello-web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

**Implementation steps:**

1. Apply both manifests: `kubectl apply -f manifests/`
2. Confirm all three Pods are ready: `kubectl get pods -l app=hello-web`
3. View it in a browser via port-forward (works regardless of `kind`'s networking setup):
   ```bash
   kubectl port-forward svc/hello-web 8080:80
   # open http://localhost:8080
   ```
4. Prove self-healing: delete a Pod and watch Kubernetes replace it automatically.
   ```bash
   kubectl delete pod -l app=hello-web --field-selector=status.phase=Running -o name | head -1 | xargs kubectl delete
   kubectl get pods -l app=hello-web -w
   ```

**Success criteria:**

- `kubectl get pods` shows `3/3` Pods in `Running` and `Ready` state
- The nginx welcome/demo page loads in a browser via port-forward or NodePort
- Deleting any one Pod results in a new Pod automatically appearing within seconds, and the Service never fully drops (verify with a `while true; do curl -s localhost:8080 > /dev/null && echo ok; sleep 0.5; done` loop running in a second terminal during the delete)

---

## Project 2 — Multi-Tier Application (Intermediate)

**Goal:** Deploy a realistic 3-tier application — frontend, backend API, and a Postgres database — with configuration externalized, secrets handled properly, persistent storage for the database, and a single Ingress exposing everything under one hostname.

**Architecture:**

```
Internet
   │
   ▼
Ingress (host: shop.local)
   │             │
   │ /           │ /api
   ▼             ▼
frontend Svc   backend Svc  ──►  db Svc (ClusterIP, internal only)
(ClusterIP)    (ClusterIP)         │
                                   ▼
                              Postgres + PVC
```

**Project structure:**

```
manifests/
├── frontend/
│   ├── deployment.yaml
│   └── service.yaml
├── backend/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── db/
│   ├── secret.yaml
│   ├── pvc.yaml
│   ├── statefulset.yaml
│   └── service.yaml
└── ingress.yaml
```

**`manifests/db/secret.yaml`:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_USER: shopapp
  POSTGRES_PASSWORD: change-me-in-real-life
  POSTGRES_DB: shop
```

**`manifests/db/pvc.yaml`** (uses a StorageClass from Chapter 8 — `standard` on `kind`, `gp3`/`ebs-sc` on EKS):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
```

**`manifests/db/statefulset.yaml`** (a StatefulSet is the right choice here, not a Deployment — see Chapter 12 — because the database needs a stable identity and its own dedicated volume):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
        - name: postgres
          image: postgres:16.2
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef: { name: db-credentials }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata     # avoids the "lost+found" mount-root issue
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits: { cpu: 500m, memory: 512Mi }
          readinessProbe:
            exec: { command: ["pg_isready", "-U", "shopapp"] }
            initialDelaySeconds: 5
          livenessProbe:
            exec: { command: ["pg_isready", "-U", "shopapp"] }
            initialDelaySeconds: 15
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
```

**`manifests/db/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None      # headless — StatefulSet Pods get stable DNS names
  selector: { app: postgres }
  ports:
    - port: 5432
```

**`manifests/backend/configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  LOG_LEVEL: "info"
  DB_HOST: "postgres.default.svc.cluster.local"
  DB_PORT: "5432"
```

**`manifests/backend/deployment.yaml`** must: run 2+ replicas, load `backend-config` via `envFrom.configMapRef`, load `db-credentials` via `envFrom.secretRef`, and set resource requests/limits plus liveness/readiness probes on `/healthz`. **`manifests/backend/service.yaml`** exposes it as a ClusterIP on port 8080. The **frontend** files follow the same pattern (Deployment with 2+ replicas serving a static/SPA build, ClusterIP Service on port 80) and read the API's base URL from its own ConfigMap so it can call `/api` through the Ingress rather than hardcoding the backend's internal address.

**`manifests/ingress.yaml`** (near-complete):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: shop.local
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: backend
                port:
                  number: 8080
          - path: /()(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

**Implementation steps:**

1. Install an Ingress controller in the `kind` cluster (Chapter 11): `ingress-nginx` via its official manifest.
2. Apply manifests in dependency order: `db/` → `backend/` → `frontend/` → `ingress.yaml`.
3. Add `127.0.0.1 shop.local` to `/etc/hosts` and port-forward the ingress controller's Service, or use `kind`'s `extraPortMappings` to bind 80/443 on the host.
4. Verify: `curl http://shop.local/api/health` returns the backend's health payload, and `curl http://shop.local/` returns the frontend HTML.
5. Kill the `postgres-0` Pod and confirm the PVC re-attaches to the recreated Pod with data intact (`\dt` in `psql` still shows your tables).

**Success criteria:** All three tiers are `Running`/`Ready`; the Ingress correctly routes `/` to the frontend and `/api` to the backend; the database survives a Pod restart with data intact; secrets never appear in a `ConfigMap` or in plaintext YAML checked into Git.

---

## Project 3 — Package and Autoscale (Advanced)

**Goal:** Take Project 2's application and turn it into a proper, reusable Helm chart (Chapter 13), add a Horizontal Pod Autoscaler to the backend (Chapter 14), and add a nightly CronJob (Chapter 12) that backs up the Postgres database.

**Helm chart structure:**

```
shop-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── backend-configmap.yaml
│   ├── db-statefulset.yaml
│   ├── db-service.yaml
│   ├── db-pvc.yaml
│   ├── db-secret.yaml
│   ├── db-backup-cronjob.yaml
│   └── ingress.yaml
```

**Key `values.yaml` fields** (parametrizing exactly what Chapter 13 teaches — replica counts, image tags, and resources should never be hardcoded in templates):

```yaml
frontend:
  image:
    repository: myorg/shop-frontend
    tag: "1.2.0"
  replicaCount: 2
  resources:
    requests: { cpu: 50m, memory: 64Mi }
    limits: { cpu: 200m, memory: 128Mi }

backend:
  image:
    repository: myorg/shop-backend
    tag: "1.5.0"
  replicaCount: 2
  resources:
    requests: { cpu: 100m, memory: 128Mi }
    limits: { cpu: 500m, memory: 256Mi }
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

database:
  image:
    repository: postgres
    tag: "16.2"
  storage: 5Gi
  storageClassName: standard
  backup:
    enabled: true
    schedule: "0 2 * * *"     # nightly at 02:00
    retentionDays: 7

ingress:
  enabled: true
  host: shop.local
  className: nginx
```

**`templates/backend-hpa.yaml`** (templated so it only renders when `autoscaling.enabled` is true):

```yaml
{{- if .Values.backend.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}-backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}-backend
  minReplicas: {{ .Values.backend.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.backend.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.backend.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

**`templates/db-backup-cronjob.yaml`:**

```yaml
{{- if .Values.database.backup.enabled }}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ .Release.Name }}-db-backup
spec:
  schedule: {{ .Values.database.backup.schedule | quote }}
  successfulJobsHistoryLimit: {{ .Values.database.backup.retentionDays }}
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: pg-dump
              image: postgres:16.2
              command:
                - sh
                - -c
                - >
                  pg_dump -h {{ .Release.Name }}-postgres -U $POSTGRES_USER $POSTGRES_DB
                  | gzip > /backup/shop-$(date +%Y%m%d-%H%M%S).sql.gz
              envFrom:
                - secretRef: { name: {{ .Release.Name }}-db-credentials }
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: {{ .Release.Name }}-db-backup-pvc
{{- end }}
```

Every workload template (frontend, backend, db) must include `resources.requests`/`resources.limits` and liveness/readiness probes sourced from `values.yaml`, per Chapter 15's best practices — this is the point where "best practice" stops being theoretical and becomes a checklist you apply to your own chart.

**Implementation steps:**

1. Scaffold with `helm create shop-chart`, then strip the generated boilerplate down to the templates above.
2. Lint and dry-run before ever touching a cluster: `helm lint shop-chart && helm template shop-chart --values shop-chart/values.yaml`.
3. Install: `helm install shop shop-chart/`.
4. Load-test the backend (e.g., with `hey` or `k6`) and watch the HPA react: `kubectl get hpa -w`.
5. Manually trigger the CronJob once to verify the backup logic before waiting for 2 AM: `kubectl create job --from=cronjob/shop-db-backup manual-test-1`.
6. Upgrade the release with a changed `backend.image.tag` and confirm Helm performs a clean rolling update: `helm upgrade shop shop-chart/ --set backend.image.tag=1.5.1`.

**Success criteria:** `helm install`/`helm upgrade` work cleanly with no manual `kubectl` steps; the HPA scales the backend up under load and back down after load stops; a manually triggered backup Job produces a valid `.sql.gz` dump; every Pod in `kubectl get pods` shows defined resource requests via `kubectl describe pod`.

---

## Project 4 — Production-Grade Capstone

**Goal:** Take the Helm chart from Project 3 and deploy it to a real managed Kubernetes cluster with production networking, TLS, and governance controls — the kind of deployment that belongs in a portfolio.

**What's different from local `kind`:**

- **Real cluster:** Provision an EKS cluster using the Terraform skills from Topic 7 (VPC, node groups, IAM roles for service accounts), or use any free-tier managed Kubernetes offering if AWS costs are a concern. This is the payoff moment for having learned Terraform first — the cluster itself becomes code you can `terraform apply`.
- **Real Ingress Controller + TLS:** Install `ingress-nginx` (or the AWS Load Balancer Controller) plus `cert-manager`, and request a real certificate via a `ClusterIssuer` (Let's Encrypt):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
# Add to the chart's ingress.yaml template:
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: ["shop.example.com"]
      secretName: shop-tls
```

- **Namespace ResourceQuota** (Chapter 9), so this application cannot starve other tenants of the cluster:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shop-quota
  namespace: shop-prod
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "30"
```

- **PodDisruptionBudgets** for frontend and backend, so a node drain during a cluster upgrade never takes the whole tier down at once (see Chapter 16):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: shop-backend }
```

- **The natural next step, not built here:** centralized logging and metrics. A DaemonSet-based log shipper (Fluent Bit) and a Prometheus/Grafana stack are exactly what Topic 10 (Monitoring & Logging) covers — note it in your project README as "future work" so reviewers see you understand the full production picture, even though it's out of scope for this course.

**Implementation steps:**

1. Provision the cluster with Terraform (or use a managed free-tier cluster) and point `kubectl` at it.
2. Create a dedicated namespace (`shop-prod`) and apply the `ResourceQuota`.
3. Install `ingress-nginx` and `cert-manager` via their Helm charts.
4. `helm install shop shop-chart/ -f values-prod.yaml -n shop-prod` — a production values override with real image tags, a real hostname, and TLS enabled.
5. Confirm the certificate issues successfully: `kubectl describe certificate shop-tls -n shop-prod`.
6. Add the PDBs and confirm with `kubectl get pdb -n shop-prod`.
7. Write a README documenting the architecture, how to deploy it, and what monitoring/logging you'd add next (Topic 10) — this is what makes it portfolio-quality rather than just "a working cluster."

**Success criteria:** The application is reachable over HTTPS with a browser-trusted certificate; `kubectl get resourcequota -n shop-prod` shows usage tracked against real limits; a simulated node drain (`kubectl drain <node> --ignore-daemonsets`) never drops the frontend or backend below their PDB's `minAvailable`; the whole thing is defined as code (Terraform + Helm) and pushed to a public GitHub repository suitable for linking from a resume.

---

## Summary

These four projects build on each other in a deliberate progression: Project 1 proves you can operate the basic building blocks, Project 2 assembles them into something realistic, Project 3 makes it operable and self-scaling, and Project 4 makes it production-grade and portfolio-ready. Each one reuses manifests, patterns, and lessons from the one before it.

| Project | Level | Approx Time | Key Skills |
|---------|-------|-------------|------------|
| 1 — First App | Beginner | 1 hour | Deployment, Service, self-healing, `kubectl port-forward` |
| 2 — Multi-Tier App | Intermediate | 4–6 hours | ConfigMap/Secret, PVC, StatefulSet, Ingress path routing |
| 3 — Package and Autoscale | Advanced | 6–8 hours | Helm charts, HPA, CronJob, resource governance |
| 4 — Production Capstone | Advanced | 8–12 hours | Managed cluster, TLS/cert-manager, ResourceQuota, PDB |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./16-common-mistakes.md">← Previous: Common Mistakes and Pitfalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./18-interview-preparation.md">Next: Interview Preparation →</a>
</div>
