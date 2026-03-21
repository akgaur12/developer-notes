# 🧠 Kubernetes Interview Q&A

## 🔹 Basic Kubernetes Questions

### 1. What is Kubernetes?
Kubernetes (K8s) is an **open-source container orchestration platform** used to automate deployment, scaling, and management of containerized applications.

---

### 2. What problems does Kubernetes solve?
- Container scheduling
- Auto-scaling
- Self-healing
- Load balancing
- Rolling updates & rollbacks

---

### 3. What is a Pod?
A Pod is the **smallest deployable unit** in Kubernetes.
It can contain **one or more containers** sharing network and storage.

---

### 4. Difference between Pod and Container?
| Pod | Container |
|----|----|
K8s abstraction | Runtime unit |
Can contain multiple containers | Single process |
Gets IP | Shares pod IP |

---

### 5. What is a Node?
A Node is a **worker machine** (VM or physical) where Pods run.

---

## 🔹 Core Components

### 6. What are the main Kubernetes components?
**Control Plane**
- API Server
- Scheduler
- Controller Manager
- etcd

**Worker Node**
- kubelet
- kube-proxy
- container runtime

---

### 7. What is etcd?
etcd is a **distributed key-value store** that stores cluster state and configuration.

---

### 8. What is kube-apiserver?
It is the **entry point** to the Kubernetes cluster.

---

## 🔹 Deployment & Scaling

### 9. What is a Deployment?
A Deployment manages Pods, ReplicaSets, rolling updates, and rollbacks.

---

### 10. What is a ReplicaSet?
A ReplicaSet ensures a specified number of Pods are always running.

---

### 11. Deployment vs StatefulSet
| Deployment | StatefulSet |
|----|----|
Stateless apps | Stateful apps |
Random pod names | Fixed pod names |
No ordering | Ordered lifecycle |

---

### 12. How does rolling update work?
Kubernetes gradually replaces old Pods with new ones using:
- maxSurge
- maxUnavailable

---

## 🔹 Services & Networking

### 13. What is a Service?
A Service provides a **stable network endpoint** to access Pods.

---

### 14. Types of Services
| Type | Use |
|----|----|
ClusterIP | Internal |
NodePort | External (Node IP) |
LoadBalancer | Cloud LB |
ExternalName | External DNS |

---

### 15. What is kube-proxy?
kube-proxy manages network rules for Service-to-Pod communication.

---

## 🔹 Configuration & Secrets

### 16. What is a ConfigMap?
Stores **non-sensitive configuration** like env vars and config files.

---

### 17. What is a Secret?
Stores **sensitive data** like passwords and tokens (base64-encoded).

---

## 🔹 Storage

### 18. What is a Volume?
Provides storage beyond container lifecycle.

---

### 19. emptyDir vs PVC
| emptyDir | PVC |
|----|----|
Temporary | Persistent |
Deleted with pod | Survives pod |

---

## 🔹 Troubleshooting & Debugging

### 20. What is ImagePullBackOff?
Kubernetes cannot pull the container image due to tag, auth, or registry issues.

---

### 21. How to debug a crashing pod?
```bash
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

---

### 22. What is CrashLoopBackOff?
Pod keeps restarting due to application or config errors.

---

## 🔹 Advanced / Real-World

### 23. How does Kubernetes do self-healing?
- Restarts containers
- Recreates Pods
- Reschedules Pods

---

### 24. How to do zero-downtime deployments?
Using rolling updates, readiness probes, and multiple replicas.

---

### 25. Readiness vs Liveness probe
| Readiness | Liveness |
|----|----|
Controls traffic | Controls restart |

---

### 26. How to rollback a deployment?
```bash
kubectl rollout undo deployment <name>
```

---

### 27. App running but not accessible – what to check?
- Pod status
- Service
- Endpoints
- Ports
- Network policies

---

### 28. Pod running but Service has no endpoints?
Service selector does not match Pod labels.

---

## ⭐ Interview Tip
Always explain **what**, **why**, and **how** with real examples.
