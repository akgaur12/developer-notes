# 📘 Kubernetes kubectl Commands Cheat Sheet

This document contains **commonly used Kubernetes (kubectl) commands**, grouped by category for quick reference.

---

## 📦 Pod Commands

| Command | Description |
|-------|-------------|
| `kubectl get pods` | List all pods |
| `kubectl get pods -o wide` | List pods with node & IP |
| `kubectl describe pod <pod>` | Detailed pod info & events |
| `kubectl logs <pod>` | View pod logs |
| `kubectl logs <pod> --previous` | Logs from crashed container |
| `kubectl exec -it <pod> -- /bin/bash` | Access pod shell |
| `kubectl delete pod <pod>` | Delete a pod |

---

## 🚀 Deployment & ReplicaSet

| Command | Description |
|-------|-------------|
| `kubectl get deploy` | List deployments |
| `kubectl describe deploy <name>` | Deployment details |
| `kubectl get rs` | List ReplicaSets |
| `kubectl rollout status deploy <name>` | Check rollout |
| `kubectl rollout restart deploy <name>` | Restart deployment |
| `kubectl rollout undo deploy <name>` | Rollback deployment |

---

## 🌐 Services & Networking

| Command | Description |
|-------|-------------|
| `kubectl get svc` | List services |
| `kubectl describe svc <svc>` | Service details |
| `kubectl get endpoints <svc>` | Pods behind a service |
| `kubectl port-forward svc/<svc> 8080:80` | Forward service port |
| `kubectl port-forward pod/<pod> 8080:80` | Forward pod port |

---

## ⚙️ ConfigMaps & Secrets

| Command | Description |
|-------|-------------|
| `kubectl get configmap` | List ConfigMaps |
| `kubectl describe configmap <cm>` | ConfigMap details |
| `kubectl get secret` | List Secrets |
| `kubectl describe secret <secret>` | Secret metadata |

---

## 🧠 Nodes & Cluster Info

| Command | Description |
|-------|-------------|
| `kubectl get nodes` | List cluster nodes |
| `kubectl describe node <node>` | Node details |
| `kubectl top nodes` | Node resource usage |
| `kubectl top pods` | Pod resource usage |

---

## 📂 Namespace Management

| Command | Description |
|-------|-------------|
| `kubectl get ns` | List namespaces |
| `kubectl get pods -n <ns>` | Pods in a namespace |
| `kubectl config set-context --current --namespace=<ns>` | Set default namespace |

---

## 📄 Apply & YAML Operations

| Command | Description |
|-------|-------------|
| `kubectl apply -f file.yaml` | Apply configuration |
| `kubectl delete -f file.yaml` | Delete resources |
| `kubectl get deploy -o yaml` | Export YAML |
| `kubectl diff -f file.yaml` | Preview changes |

---

## 🧪 Debugging & Troubleshooting

| Command | Description |
|-------|-------------|
| `kubectl get events` | View cluster events |
| `kubectl get events --sort-by=.metadata.creationTimestamp` | Events sorted |
| `kubectl describe pod <pod>` | Debug pod issues |
| `kubectl get pods -w` | Watch pod changes live |

---

## 🔑 Context & Configuration

| Command | Description |
|-------|-------------|
| `kubectl config get-contexts` | List contexts |
| `kubectl config use-context <ctx>` | Switch cluster |
| `kubectl config view` | View kubeconfig |

---

## ⭐ Most Used Commands (Daily)

```bash
kubectl get pods
kubectl logs <pod>
kubectl describe pod <pod>
kubectl get svc
kubectl rollout restart deploy <name>
```

---

## ✅ Tip
Always combine `kubectl describe` + `kubectl logs` when debugging issues.
