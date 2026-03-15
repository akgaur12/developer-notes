# 🚀 DevOps Learning Path

A **step-by-step DevOps roadmap** explaining **what to study first and what comes next**, from beginner to advanced level.

---

## 🧱 Phase 0: Core Foundations (Must Have)

These are **non-negotiable basics** for DevOps.

### 1️⃣ Linux Fundamentals

Learn how systems actually work.

**Topics:**

* Linux filesystem (`/etc`, `/var`, `/usr`, `/opt`)
* File & directory commands: `ls`, `cd`, `cp`, `mv`, `rm`
* Text processing: `grep`, `awk`, `sed`
* Permissions: `chmod`, `chown`
* Process management: `ps`, `top`, `kill`
* Networking tools: `curl`, `wget`, `netstat`, `ss`

**Goal:** Work confidently using only the terminal

---

### 2️⃣ Networking Basics

Understand how applications communicate.

**Topics:**

* TCP/IP, UDP
* DNS
* HTTP vs HTTPS
* Ports & firewalls
* Load balancer & reverse proxy concepts

**Tools:**

* `curl`, `ping`, `traceroute`
* Nginx (basic setup)

---

### 3️⃣ Git & Version Control

Essential for collaboration and CI/CD.

**Topics:**

* `clone`, `add`, `commit`, `push`, `pull`
* Branching strategies
* `merge` vs `rebase`
* `stash`, `cherry-pick`
* GitHub / GitLab workflows

---

## 🐳 Phase 1: Containers & CI/CD

This is where DevOps truly begins.

### 4️⃣ Docker (Very Important)

Learn how applications are packaged.

**Topics:**

* Images vs containers
* Dockerfile & multi-stage builds
* Volumes & networks
* Docker Compose

**Hands-on:**

* Containerize a backend app
* Connect app + database using Docker Compose

---

### 5️⃣ CI/CD Pipelines

Automate build, test, and deployment.

**Topics:**

* CI vs CD
* Pipeline stages
* Build automation

**Tools:**

* GitHub Actions (recommended)
* GitLab CI

**Hands-on:**

* Build Docker image on every push
* Run automated tests
* Push image to container registry

---

## ☁️ Phase 2: Cloud & Infrastructure

Move from local to production mindset.

### 6️⃣ Cloud Fundamentals (AWS Preferred)

Learn the cloud building blocks.

**Topics:**

* EC2
* IAM
* S3
* VPC (basic understanding)
* Security Groups
* Load Balancers

---

### 7️⃣ Infrastructure as Code (IaC)

Provision infrastructure using code.

**Tool:** Terraform

**Topics:**

* Providers & resources
* State management
* Variables & outputs
* Modules

**Hands-on:**

* Create EC2 using Terraform
* Deploy Dockerized app via Terraform

---

## ☸️ Phase 3: Kubernetes (Core DevOps Skill)

Mandatory for modern DevOps roles.

### 8️⃣ Kubernetes Basics

**Topics:**

* Pods
* Deployments
* Services
* ConfigMaps & Secrets
* Namespaces
* Ingress

**Hands-on:**

* Deploy application to Kubernetes
* Expose it using Ingress

---

### 9️⃣ Advanced Kubernetes

**Topics:**

* Helm charts
* Rolling updates
* Auto-scaling (HPA)
* Resource limits & requests
* Liveness & readiness probes

---

## 📊 Phase 4: Observability & Operations

### 🔟 Monitoring & Logging

**Concepts:**

* Metrics vs logs vs traces

**Tools:**

* Prometheus
* Grafana
* ELK Stack / Loki

**Hands-on:**

* Monitor CPU & memory
* Visualize logs in Grafana

---

### 1️⃣1️⃣ Security (DevSecOps Basics)

**Topics:**

* Secrets management
* IAM best practices
* Container security
* Image vulnerability scanning

**Tools:**

* Trivy
* Vault (basic)

---

## ⚙️ Phase 5: Advanced & Optional Skills

Learn after core DevOps is strong.

* Ansible (configuration management)
* ArgoCD (GitOps)
* Service Mesh (Istio)
* SRE principles
* Chaos Engineering

---

## 🧠 Learning Order Summary

```
Linux → Networking → Git → Docker → CI/CD → Cloud → Terraform → Kubernetes → Monitoring → Security
```

---

## 📆 Suggested Timeline

| Phase                 | Duration  |
| --------------------- | --------- |
| Foundations           | 2–3 weeks |
| Docker & CI/CD        | 3–4 weeks |
| Cloud & Terraform     | 3 weeks   |
| Kubernetes            | 4–5 weeks |
| Monitoring & Security | 2 weeks   |

**⏱ Total: ~3–4 months (job-ready DevOps skills)**

---

## 🎯 Final Advice

* Focus on **hands-on projects**
* Learn by **building & breaking things**
* Don’t memorize tools — understand *why* they exist

Happy Learning 🚀
