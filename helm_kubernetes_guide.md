# 🧠 Helm – Kubernetes Package Manager

## What is Helm?
Helm is a **package manager for Kubernetes**, similar to:
- `apt` for Linux
- `pip` for Python
- `npm` for Node.js

It helps you **define, install, upgrade, and manage Kubernetes applications** easily.

---

## Why Helm is needed

Without Helm:
- Many YAML files to manage
- Manual changes per environment
- Hard rollbacks

With Helm:
- YAML templating
- Versioned releases
- Easy upgrades & rollbacks
- Reusable configurations

---

## Simple Interview Definition

> **Helm is a Kubernetes package manager that uses charts to deploy and manage applications efficiently.**

---

## Key Helm Concepts

### 1️⃣ Chart
A **Chart** is a Helm package that contains all Kubernetes resources for an app.

Example structure:
```
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
```

---

### 2️⃣ Values (`values.yaml`)
Stores configurable values:
```yaml
replicaCount: 2
image:
  repository: genaiapp
  tag: 1.0.72
```

---

### 3️⃣ Templates
Uses Go templating syntax:
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

### 4️⃣ Release
A **Release** is a deployed instance of a chart.

```bash
helm install genaiapp ./myapp
```

---

## What Helm Can Do

| Feature | Description |
|------|------------|
Install | Deploy applications |
Upgrade | Update app versions |
Rollback | Revert to previous version |
Versioning | Track release history |
Reuse | Same chart for multiple envs |

---

## Common Helm Commands

| Command | Purpose |
|------|--------|
| `helm install` | Install a chart |
| `helm upgrade` | Upgrade a release |
| `helm rollback` | Rollback a release |
| `helm uninstall` | Remove a release |
| `helm list` | List releases |
| `helm history` | View release history |

---

## Example: Image Update Using Helm

```bash
helm upgrade genaiapp ./myapp --set image.tag=1.0.73
```

Rollback:
```bash
helm rollback genaiapp 1
```

---

## Helm vs kubectl apply

| kubectl apply | Helm |
|--------------|------|
Manual YAML | Template-based |
No rollback history | Built-in rollback |
Hard to manage envs | Easy env configs |
No versioning | Versioned releases |

---

## When to Use Helm

✅ Production workloads  
✅ Microservices  
✅ CI/CD pipelines  
✅ Multiple environments  

❌ Small one-off deployments

---

## One-line Interview Answer

> **Helm is a package manager for Kubernetes that uses charts to install, upgrade, and manage applications with version control.**

---

## ⭐ Tip
Helm is commonly used in **production Kubernetes clusters** and **DevOps pipelines**.
