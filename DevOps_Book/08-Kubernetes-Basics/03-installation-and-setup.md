# Chapter 3 — Installation and Setup

## Learning Objectives

By the end of this chapter you will be able to:

- Compare the local cluster options (kind, minikube, k3d, Docker Desktop) and managed cloud options (EKS, GKE, AKS), and explain when each is appropriate
- Install `kind` and `kubectl`, and create your first working local cluster
- Verify a cluster is healthy using `kubectl cluster-info` and `kubectl get nodes`
- Explain the structure of a kubeconfig file and switch between cluster contexts
- Read and write a basic Kubernetes YAML manifest, understanding all four required top-level fields
- Explain the difference between imperative and declarative `kubectl` usage, and state why production teams standardize on declarative
- Use the essential `kubectl` commands for inspecting and managing cluster objects

---

## Prerequisites for This Chapter

- **Chapter 1 — Introduction to Kubernetes** — required. You should already know the vocabulary (Cluster, Node, Pod, Control Plane) and the high-level architecture map.
- **Chapter 2 — Architecture and Internals** — required. This chapter leans heavily on the "desired state vs. actual state" concept from Chapter 2 to explain why declarative usage is preferred.
- **Docker (Topic 4)** — required. `kind` runs Kubernetes nodes as Docker containers, so a working Docker installation is a hard prerequisite for the exercises in this chapter.

---

## 3.1 Ways to Run Kubernetes

There is no single "install Kubernetes" step — you choose a cluster based on what you're trying to do. Broadly, the options split into two categories: **local development clusters** (free, fast, disposable, for learning and iteration) and **managed cloud clusters** (production-grade, cost money, backed by a cloud provider's SLA).

### Local Development Clusters

| Tool | How It Works | Strengths | Trade-offs |
|------|--------------|-----------|------------|
| **kind** (Kubernetes IN Docker) | Runs each Kubernetes "node" as a Docker container | Fast startup, easy multi-node clusters, scriptable, no VM overhead, actively maintained by the Kubernetes project itself | Requires Docker; not intended for production |
| **minikube** | Runs Kubernetes inside a single VM (or container) driver | Very mature, many built-in addons (dashboard, ingress, metrics-server) | Slower startup than kind, historically single-node-first |
| **k3d** | Runs **k3s** (a lightweight Kubernetes distribution by Rancher) inside Docker | Extremely lightweight and fast, good for resource-constrained machines | Uses k3s, a stripped-down distribution — a few defaults differ subtly from upstream Kubernetes |
| **Docker Desktop (built-in Kubernetes)** | Enables a single-node cluster inside Docker Desktop itself | Zero extra install if you already use Docker Desktop, simple toggle in settings | Mac/Windows only, single-node only, tied to Docker Desktop's release cycle |

### Managed Cloud Clusters (Production)

| Service | Provider | Notes |
|---------|----------|-------|
| **EKS** (Elastic Kubernetes Service) | AWS | Fully managed control plane; you manage (or use Fargate to avoid managing) worker nodes |
| **GKE** (Google Kubernetes Engine) | Google Cloud | Widely regarded as the most mature managed offering, since Google originated Kubernetes |
| **AKS** (Azure Kubernetes Service) | Microsoft Azure | Free control plane (you pay only for worker nodes), deep Azure AD integration |

**The tradeoff in one sentence:** local clusters (kind, minikube, k3d) give you free, fast iteration with zero cloud cost and zero cloud account requirement, but no real high availability and no production traffic; managed clusters (EKS, GKE, AKS) give you a production-grade, highly available control plane, automatic node repair, and deep cloud integration (load balancers, IAM, storage), but cost real money and take longer to provision.

You do not choose one forever — professional Kubernetes engineers use local clusters daily for development and testing manifests before ever touching a shared staging or production cluster. That is exactly the workflow this course follows.

---

## 3.2 Why This Course Uses `kind`

Every hands-on exercise in this course runs on **kind**. Three reasons:

1. **Free and account-less.** No cloud account, no billing, no credit card — it runs entirely on your own machine using Docker, which you already installed in Topic 4.
2. **Fast and disposable.** A `kind` cluster starts in well under a minute and can be deleted and recreated just as fast — perfect for experimenting without fear of breaking something expensive.
3. **Faithful to upstream Kubernetes.** Unlike k3s-based tools, `kind` runs the real, unmodified Kubernetes control plane and components discussed in Chapter 2 — what you learn transfers directly to a managed cluster with no surprises about missing or renamed features.

### Installing kind and kubectl

```bash
# --- Install kubectl (the Kubernetes CLI) on Linux ---
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client

# --- Install kind on Linux ---
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version
```

```bash
# --- macOS (via Homebrew) ---
brew install kubectl
brew install kind

# --- Windows (via Chocolatey) ---
choco install kubernetes-cli
choco install kind
```

Both tools are single static binaries — there is no daemon to configure for `kubectl` itself, and `kind` only needs a running Docker daemon to talk to.

---

## 3.3 Creating Your First Cluster

```bash
# Create a cluster named "learn-k8s" (kind creates one control-plane container)
kind create cluster --name learn-k8s
```

Expected output looks roughly like this:

```
Creating cluster "learn-k8s" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-learn-k8s"
You can now use your cluster with:

kubectl cluster-info --context kind-learn-k8s
```

Notice `kind` already did two useful things for you: it created a working **CNI** (Container Network Interface plugin, so Pods can talk to each other) and a default **StorageClass** — both things you would have to configure by hand on a truly bare cluster.

**Verify the cluster is healthy:**

```bash
# Confirm the control plane is reachable and shows expected core services
kubectl cluster-info

# Expected output includes:
# Kubernetes control plane is running at https://127.0.0.1:XXXXX
# CoreDNS is running at https://127.0.0.1:XXXXX/api/v1/namespaces/kube-system/...

# List the Node(s) in the cluster
kubectl get nodes

# Expected output:
# NAME                       STATUS   ROLES           AGE   VERSION
# learn-k8s-control-plane    Ready    control-plane   45s   v1.30.0
```

A `STATUS` of `Ready` means the `kubelet` on that Node (recall Chapter 2) has successfully registered with the API server and is reporting healthy. If you ever see `NotReady`, that is your first clue to go check the `kubelet` and networking layer on that Node — exactly the kind of architecture-informed debugging Chapter 2 was preparing you for.

**Creating a multi-node cluster** (useful later in the course, e.g. for scheduling and affinity exercises in Chapter 10):

```yaml
# kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name multi-node-demo --config kind-multi-node.yaml
kubectl get nodes
# Shows one control-plane node and two worker nodes
```

**Cleaning up when you're done experimenting:**

```bash
kind delete cluster --name learn-k8s
```

---

## 3.4 The kubeconfig File

`kubectl` needs to know two things to talk to a cluster: *where* the API server is, and *who you are*. Both live in a file called the **kubeconfig**.

- **Default location:** `~/.kube/config`
- When you ran `kind create cluster`, it automatically wrote a new entry into this file (or created it, if it didn't exist) and pointed `kubectl` at it.

### Structure of a kubeconfig

A kubeconfig has three sections that work together, plus a pointer to which combination is currently active:

```yaml
apiVersion: v1
kind: Config

clusters:
  - name: kind-learn-k8s
    cluster:
      server: https://127.0.0.1:52341     # where the API server lives
      certificate-authority-data: ...      # CA cert to trust the server

users:
  - name: kind-learn-k8s
    user:
      client-certificate-data: ...         # your client cert
      client-key-data: ...                 # your client private key

contexts:
  - name: kind-learn-k8s
    context:
      cluster: kind-learn-k8s              # which cluster entry to use
      user: kind-learn-k8s                 # which user entry to use
      namespace: default                   # optional: default namespace for this context

current-context: kind-learn-k8s            # the context kubectl uses right now
```

Think of it as three independent lists — clusters (network locations), users (identities/credentials), and contexts (named pairings of a cluster + a user, plus an optional default namespace) — with one "current context" switch that decides which pairing `kubectl` uses for every command you run.

### Why This Matters: Juggling Multiple Clusters

The moment you have more than one cluster — your local `kind` cluster, a shared staging cluster, and a production cluster — the kubeconfig's context system is what keeps you from accidentally running a command against the wrong one. A single kubeconfig file can hold entries for all of them simultaneously.

```bash
# List every context currently known to kubectl
kubectl config get-contexts

# Example output:
# CURRENT   NAME              CLUSTER           NAMESPACE
# *         kind-learn-k8s    kind-learn-k8s    default
#           staging-cluster   staging-eks       staging
#           prod-cluster      prod-eks          production

# Switch which cluster/user kubectl targets
kubectl config use-context staging-cluster

# Confirm which context is currently active
kubectl config current-context

# Run a one-off command against a different context without switching permanently
kubectl get pods --context prod-cluster
```

**This is a genuinely dangerous corner of Kubernetes if treated casually.** Running `kubectl delete deployment payments` while your current context silently points at `prod-cluster` instead of your local `kind` cluster is a real, well-documented category of production incident. Always check `kubectl config current-context` (or use a shell prompt plugin that displays it) before running anything destructive.

---

## 3.5 Anatomy of a Kubernetes YAML Manifest

Every object you create in Kubernetes — Pods, Deployments, Services, ConfigMaps, everything — is described using the same four top-level fields.

```yaml
apiVersion: v1          # Which version of the Kubernetes API defines this object's schema
kind: Pod                # What type of object this is
metadata:                # Data that identifies this object (not its behavior)
  name: hello-pod
  labels:
    app: hello
spec:                     # The desired state — what you actually want to happen
  containers:
    - name: hello-container
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Line by line:

- **`apiVersion`** — Kubernetes' API is versioned and grouped (e.g., `v1` for core objects like Pods and Services, `apps/v1` for Deployments and ReplicaSets, `batch/v1` for Jobs). This tells the API server which schema to validate your object against. Get this wrong (e.g., write `apiVersion: v1` for a Deployment, which actually needs `apps/v1`) and the API server will reject the manifest outright.
- **`kind`** — the object type: `Pod`, `Deployment`, `Service`, `ConfigMap`, `Namespace`, and dozens more, each covered in upcoming chapters.
- **`metadata`** — identifying information: `name` (must be unique within its Namespace), optional `labels` (key-value tags used for selection — Chapter 4 covers labels in depth), optional `namespace` (which Namespace this object belongs to; defaults to `default` if omitted), and optional `annotations` (non-identifying metadata, often used by tooling).
- **`spec`** — the actual **desired state**, tying directly back to Chapter 2's reconciliation loop. Everything under `spec` is what you are asking Kubernetes to make true and continuously keep true. For a Pod, this includes which containers to run, which image and ports each uses, resource requests/limits, and more (all covered fully in Chapter 4).

Every object also gets a `status` field once it exists in the cluster — but you never write `status` yourself. It is populated and continuously updated by Kubernetes to reflect **actual** state (e.g., a Pod's `status.phase: Running`), which is precisely the other half of the desired-vs-actual comparison from Chapter 2.

```bash
# Create the Pod described above
kubectl apply -f hello-pod.yaml

# See both spec (what you asked for) and status (what's actually true) together
kubectl get pod hello-pod -o yaml
```

---

## 3.6 Imperative vs. Declarative kubectl Usage

Just like Terraform distinguishes declarative infrastructure from imperative scripts, `kubectl` supports both an imperative style and a declarative style — and which one you use matters a great deal in a professional setting.

### Imperative Commands

Imperative commands tell Kubernetes exactly what action to take, right now, with no file involved:

```bash
# Imperatively create a Pod running nginx
kubectl run hello-pod --image=nginx:1.27 --port=80

# Imperatively create a Deployment
kubectl create deployment hello-deploy --image=nginx:1.27 --replicas=3

# Imperatively expose it as a Service
kubectl expose deployment hello-deploy --port=80 --target-port=80
```

These are fast and great for **exploration** — quickly spinning something up to poke at, or checking a flag's behavior — but they leave no record. There is no file to commit to Git, no history of what was run or why, and no way to `diff` a proposed change before applying it.

### Declarative Usage

Declarative usage means writing the desired state to a YAML file and letting `kubectl apply` reconcile the live cluster to match it:

```bash
kubectl apply -f deployment.yaml
```

Recall from Chapter 2: `kubectl apply` doesn't just blindly recreate objects — it computes a diff between what's already in the cluster and what the file describes, and applies only the necessary changes. Run it once, nothing exists yet, and it creates everything. Change one field in the file and run it again, and only that field is updated. Run it a third time with no changes, and nothing happens at all — this is **idempotency**, the same property Terraform's `apply` relies on.

### Why Production Teams Use Declarative + Version Control Exclusively

| | Imperative (`kubectl run`/`create`) | Declarative (`kubectl apply -f`) |
|---|---|---|
| **Record of changes** | None — lost the moment your shell history rotates | Every change is a Git commit with an author and message |
| **Code review** | Not possible — there's nothing to review | A PR diff shows exactly what will change before it's applied |
| **Reproducibility** | Depends on someone remembering the exact flags used | Re-running `kubectl apply -f` reproduces the exact same state anywhere |
| **Rollback** | Manual, error-prone | `git revert` + `kubectl apply` restores the prior state precisely |
| **Drift detection** | None | `kubectl diff -f file.yaml` shows drift between the file and the live cluster |

This is also the seed of **GitOps** — a practice where the Git repository containing your manifests is treated as the single source of truth, and an automated agent (like Argo CD or Flux) continuously applies whatever is committed to Git, rather than engineers running `kubectl apply` from their laptops at all. GitOps is covered fully in Topic 9, Advanced Kubernetes — for now, the important habit to build is simple: **write YAML files, commit them to Git, and use `kubectl apply -f`, not imperative commands, for anything you intend to keep.**

A very common and perfectly legitimate hybrid workflow: use an imperative command to quickly scaffold a manifest, then save it as a file and commit it, rather than typing YAML from scratch:

```bash
# Generate the YAML without actually creating the object (--dry-run + -o yaml)
kubectl create deployment hello-deploy --image=nginx:1.27 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Now edit deployment.yaml by hand, commit it to Git, and apply it declaratively
kubectl apply -f deployment.yaml
```

---

## 3.7 Essential kubectl Command Reference

These commands cover the majority of day-to-day Kubernetes work. You will use every single one of them repeatedly throughout the rest of this course.

| Command | Purpose | Example |
|---------|---------|---------|
| `get` | List objects and their high-level status | `kubectl get pods` |
| `describe` | Show full details and recent events for one object — the first stop when debugging | `kubectl describe pod hello-pod` |
| `logs` | Print a container's stdout/stderr | `kubectl logs hello-pod` |
| `exec` | Run a command inside a running container | `kubectl exec -it hello-pod -- bash` |
| `apply` | Declaratively create/update objects from a file (idempotent) | `kubectl apply -f deployment.yaml` |
| `delete` | Remove an object | `kubectl delete -f deployment.yaml` |
| `edit` | Open a live object in your editor and apply changes on save | `kubectl edit deployment hello-deploy` |
| `explain` | Show field-level documentation for any object schema, straight from the API server | `kubectl explain pod.spec.containers` |

A few notes worth internalizing early:

- `kubectl describe` is almost always your **first debugging command**. Its `Events` section at the bottom shows exactly what the scheduler, kubelet, and other components have done and observed about this object recently — often the fastest route to a root cause.
- `kubectl logs -f hello-pod` (`-f` for follow) streams logs live, similar to `tail -f`.
- `kubectl explain` is an underused gift: it queries the API server's own schema, so it's always accurate for the exact Kubernetes version you're running — `kubectl explain deployment.spec.strategy` will tell you precisely what fields are valid there, with no need to search documentation.
- `kubectl edit` opens the live object, but remember this is an **imperative** action against a running cluster — any change you make this way is not reflected anywhere in Git, and the next `kubectl apply -f` from your source-controlled YAML will silently overwrite it. Treat `edit` as a debugging tool, not a deployment method.

---

## 3.8 Real-World Scenario: A New Engineer's First Day on Staging

Maria just joined the platform team. Her manager sends her a `staging-kubeconfig.yaml` file and says: "Take a look at what's running in staging before our sync tomorrow." Here is exactly what she types.

```bash
# 1. Point kubectl at the file she was given, without overwriting her existing config
export KUBECONFIG=~/Downloads/staging-kubeconfig.yaml

# 2. Sanity-check she's looking at the right place
kubectl config current-context
# staging-cluster

kubectl cluster-info
# Kubernetes control plane is running at https://staging.k8s.acmecorp.internal:6443

# 3. See what Namespaces exist, to get her bearings
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   400d
# kube-system       Active   400d
# checkout          Active   210d
# payments          Active   180d
# notifications     Active   95d

# 4. Look at what's running in the checkout namespace
kubectl get pods -n checkout
# NAME                                READY   STATUS    RESTARTS   AGE
# checkout-service-7d9f8c9b6c-abcde   1/1     Running   0          3d
# checkout-service-7d9f8c9b6c-fghij   1/1     Running   0          3d
# checkout-service-7d9f8c9b6c-klmno   1/1     Running   2          3d

# 5. Notice one Pod has 2 restarts — investigate
kubectl describe pod checkout-service-7d9f8c9b6c-klmno -n checkout
# Scrolls to the Events section at the bottom to see what happened

# 6. Check its recent logs around the restart
kubectl logs checkout-service-7d9f8c9b6c-klmno -n checkout --previous

# 7. Merge this kubeconfig permanently into her main config for future convenience,
#    rather than exporting KUBECONFIG every session
KUBECONFIG=~/.kube/config:~/Downloads/staging-kubeconfig.yaml \
  kubectl config view --flatten > ~/.kube/config-merged
mv ~/.kube/config-merged ~/.kube/config

kubectl config get-contexts
# Now shows both her local kind cluster and staging-cluster side by side
```

In fifteen minutes, using nothing but `get`, `describe`, `logs`, and basic kubeconfig management, Maria has a clear picture of what's deployed, which Namespace organizes which team's services, and has already spotted and started investigating one Pod's restarts — without needing to ask anyone anything or touch the cluster's actual configuration.

---

## Best Practices

- Use `kind` for all local learning and manifest development; reserve managed clusters for staging/production.
- Always confirm `kubectl config current-context` before running any destructive command, especially once you're managing more than one cluster.
- Write manifests declaratively and commit them to Git; treat imperative `kubectl run`/`create`/`edit` as exploration or debugging tools only, never as your deployment method.
- Use `kubectl create ... --dry-run=client -o yaml` to scaffold manifests quickly instead of writing YAML from memory.
- Make `kubectl describe` your reflexive first debugging step — its Events section usually points straight at the problem.

## Common Mistakes

- Running commands against the wrong cluster because the current context wasn't checked first — a leading cause of real production incidents.
- Using `kubectl edit` or `kubectl create` to make a "quick fix" in production, which then gets silently overwritten the next time the real manifest is applied from Git.
- Getting `apiVersion` wrong for a given `kind` (e.g., using `v1` for a Deployment instead of `apps/v1`) and being confused by the resulting API server rejection.
- Forgetting that `kind` clusters are ephemeral and local — expecting a `kind` cluster's Services of type `LoadBalancer` to behave like they would on a real cloud provider.

---

## Summary

You can run Kubernetes locally (kind, minikube, k3d, Docker Desktop) for free, fast iteration, or on a managed cloud service (EKS, GKE, AKS) for production-grade high availability and cloud integration — this course uses `kind` throughout because it is free, fast, and runs the real upstream Kubernetes components discussed in Chapter 2. The kubeconfig file (`~/.kube/config`) stores clusters, users, and contexts, and the context system is what lets you safely juggle multiple clusters — always verify your current context before running destructive commands. Every Kubernetes object is described with four top-level YAML fields — `apiVersion`, `kind`, `metadata`, `spec` — and `spec` is specifically the desired state that Chapter 2's reconciliation loop continuously enforces. Declarative usage (`kubectl apply -f`) is idempotent, diffable, and reviewable through Git, which is why production teams standardize on it over imperative commands. A small set of `kubectl` commands — `get`, `describe`, `logs`, `exec`, `apply`, `delete`, `edit`, `explain` — covers the overwhelming majority of daily Kubernetes work.

---

## Knowledge Check

1. What are the tradeoffs between using a local cluster like `kind` and a managed cloud cluster like EKS for a production application?
2. What three types of entries make up a kubeconfig file, and how does a "context" tie them together?
3. Why is it dangerous to run `kubectl delete` without first checking `kubectl config current-context`?
4. Name the four required top-level fields in every Kubernetes manifest, and explain what each one is for.
5. Why is `kubectl apply -f deployment.yaml` considered idempotent, and why does that property matter for a production workflow?
6. Give one specific reason `kubectl describe` is usually a better first debugging step than `kubectl get`.

---

## Hands-On Exercise

Complete both parts using your own machine.

### Part A — Create and Inspect a Cluster

1. Install `kubectl` and `kind` using the commands in section 3.2.
2. Create a cluster: `kind create cluster --name learn-k8s`.
3. Run `kubectl cluster-info` and `kubectl get nodes`, and confirm the Node shows `STATUS: Ready`.
4. Run `kubectl config get-contexts` and identify which context is currently active.
5. Run `kubectl get namespaces` and note the four Namespaces that exist by default in a fresh cluster.

### Part B — Write and Apply Your First Manifest

1. Create a file `hello-pod.yaml` using the exact example from section 3.5.
2. Apply it declaratively: `kubectl apply -f hello-pod.yaml`.
3. Run `kubectl apply -f hello-pod.yaml` a second time with no changes to the file, and observe the output says `unchanged` — this is idempotency in action.
4. Run `kubectl get pod hello-pod -o yaml` and find both the `spec` you wrote and the `status` Kubernetes populated for you.
5. Run `kubectl describe pod hello-pod` and locate the `Events` section at the bottom.
6. Change the `image` tag in your YAML file to a different nginx version, re-run `kubectl apply -f hello-pod.yaml`, and observe the word `configured` in the output instead of `unchanged`.
7. Clean up: `kind delete cluster --name learn-k8s`.

---

## Further Reading

- kubernetes.io/docs/tasks/tools/ — official guide to installing `kubectl` on every platform
- kind.sigs.k8s.io/docs/user/quick-start/ — official kind Quick Start guide
- kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/ — official kubeconfig documentation
- kubernetes.io/docs/reference/kubectl/quick-reference/ — official kubectl cheat sheet
- kubernetes.io/docs/concepts/overview/working-with-objects/object-management/ — imperative vs. declarative object management, straight from the source

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./02-architecture-and-internals.md">← Previous: Architecture and Internals</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./04-pods-and-workloads.md">Next: Pods and Workloads →</a>
</div>
