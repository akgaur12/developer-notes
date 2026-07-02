# Kubernetes Basics — Complete Course Index

> **DevOps Learning Path — Topic 8 of 11**

---

## Course Overview

You can already containerize an application (Docker, Topic 4), provision cloud infrastructure as code (Terraform, Topic 7), and build a CI/CD pipeline (Topic 5). Kubernetes is where these skills converge: it is the industry-standard system for running containers reliably, at scale, across a fleet of machines.

A single Docker host works fine until you need more than one machine, zero-downtime deployments, self-healing on crash, and automatic scaling under load. Kubernetes solves exactly that problem. It is now the default target for production container workloads — used by companies from three-person startups to the largest tech companies in the world, and it is the deployment layer behind most managed cloud container services (EKS, GKE, AKS).

This course takes you from "what is a Pod?" to running a multi-tier application in a real cluster with health checks, autoscaling, Ingress routing, and Helm packaging. It focuses on **using** Kubernetes as an application operator. Cluster administration topics — RBAC, network policies, multi-tenancy, custom operators, and GitOps — are covered in Topic 9, Advanced Kubernetes.

---

## What You'll Be Able to Do

- Explain what Kubernetes is, the problem it solves, and how its architecture works internally
- Install and use `kubectl` against a local cluster (kind/minikube) and a managed cluster
- Deploy, update, and roll back applications using Deployments and ReplicaSets
- Expose applications internally and externally using Services and Ingress
- Externalize configuration and secrets using ConfigMaps and Secrets
- Attach persistent storage to stateful workloads using PV, PVC, and StorageClasses
- Control resource usage and enforce quotas with Namespaces, requests/limits, and QuotaRanges
- Configure liveness, readiness, and startup probes, and schedule pods with affinity and taints/tolerations
- Run stateful applications, background agents, and batch workloads with StatefulSets, DaemonSets, Jobs, and CronJobs
- Package and version applications with Helm charts
- Scale workloads automatically with the Horizontal Pod Autoscaler
- Apply production-grade best practices and avoid the most common Kubernetes mistakes
- Answer Kubernetes interview questions with confidence, including scenario-based troubleshooting

---

## Prerequisites

- **Docker (Topic 4)** — required. You must understand images, containers, and container networking before orchestrating them.
- **Networking Basics (Topic 2)** — required. DNS, ports, and load balancing concepts are used throughout.
- **Linux Fundamentals (Topic 1)** — assumed. Comfortable with the command line and YAML-adjacent config files.
- **Infrastructure as Code — Terraform (Topic 7)** — recommended, not required. Later chapters reference provisioning a cluster with Terraform, but all exercises run on a free local cluster (kind).

---

## Estimated Learning Timeline

**5–6 weeks** at 1–2 hours/day. This is the largest course in the roadmap so far — Kubernetes has more moving parts than any single previous topic.

---

## Learning Milestones

| Milestone | Chapters | Skills Unlocked |
|-----------|----------|-----------------|
| Foundations | 01–03 | Explain Kubernetes architecture; run a local cluster; use `kubectl` |
| Core Workloads | 04–07 | Deploy, expose, and configure applications |
| Stateful & Scheduled | 08–12 | Persistent storage, resource governance, health checks, batch workloads |
| Production Operations | 13–16 | Helm, autoscaling, best practices, avoiding common mistakes |
| Professional | 17–19 | Capstone projects, interview-ready, course mastery |

---

## Full Chapter Index

| # | Chapter | Topics |
|---|---------|--------|
| 01 | [Introduction to Kubernetes](./01-introduction.md) | The orchestration problem, Kubernetes history, architecture at a glance, K8s vs alternatives |
| 02 | [Architecture and Internals](./02-architecture-and-internals.md) | Control plane components, node components, etcd, the reconciliation loop |
| 03 | [Installation and Setup](./03-installation-and-setup.md) | kubectl, kind/minikube, kubeconfig, YAML manifests, imperative vs declarative |
| 04 | [Pods and Workloads](./04-pods-and-workloads.md) | Pod fundamentals, multi-container patterns, pod lifecycle, labels |
| 05 | [Deployments and ReplicaSets](./05-deployments-and-replicasets.md) | ReplicaSets, rolling updates, rollbacks, deployment strategies |
| 06 | [Services and Networking](./06-services-and-networking.md) | Service types, kube-proxy, DNS, the Kubernetes networking model |
| 07 | [ConfigMaps and Secrets](./07-configmaps-and-secrets.md) | Externalizing configuration, env vars vs volume mounts, Secret types |
| 08 | [Storage and Persistent Volumes](./08-storage-and-persistent-volumes.md) | Volumes, PV/PVC, StorageClasses, dynamic provisioning, access modes |
| 09 | [Namespaces and Resource Management](./09-namespaces-and-resource-management.md) | Namespaces, requests/limits, QoS classes, LimitRange, ResourceQuota |
| 10 | [Health Checks and Scheduling](./10-health-checks-and-scheduling.md) | Liveness/readiness/startup probes, node affinity, taints and tolerations |
| 11 | [Ingress and Load Balancing](./11-ingress-and-load-balancing.md) | Ingress resources and controllers, path/host routing, TLS termination |
| 12 | [StatefulSets, DaemonSets and Jobs](./12-statefulsets-daemonsets-and-jobs.md) | Stateful workloads, per-node agents, batch Jobs, CronJobs |
| 13 | [Helm and Package Management](./13-helm-and-package-management.md) | Charts, releases, values, templating, Artifact Hub |
| 14 | [Scaling and Autoscaling](./14-scaling-and-autoscaling.md) | Horizontal Pod Autoscaler, Vertical Pod Autoscaler, Cluster Autoscaler |
| 15 | [Best Practices](./15-best-practices.md) | Production-grade patterns, resource governance, security basics, 12-factor alignment |
| 16 | [Common Mistakes and Pitfalls](./16-common-mistakes.md) | The most frequent Kubernetes mistakes, why they happen, how to fix them |
| 17 | [Hands-On Projects](./17-projects.md) | Beginner → intermediate → advanced → production-grade capstone projects |
| 18 | [Interview Preparation](./18-interview-preparation.md) | Foundational, architecture, scenario-based, and system design questions |
| 19 | [Course Summary](./19-course-summary.md) | Recap, checklist, quick reference, what's next |

---

## DevOps Roadmap Series

| # | Topic | Status |
|---|-------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | ✅ Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | ✅ Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | ✅ Complete |
| 4 | [Docker](../04-Docker/00-index.md) | ✅ Complete |
| 5 | [CI/CD Pipelines](../05-CI-CD-Pipelines/00-index.md) | ✅ Complete |
| 6 | [Cloud Fundamentals AWS](../06-Cloud-Fundamentals-AWS/00-index.md) | ✅ Complete |
| 7 | [Infrastructure as Code (Terraform)](../07-Infrastructure-as-Code-Terraform/00-index.md) | ✅ Complete |
| 8 | **Kubernetes Basics** | 📍 You are here |
| 9 | Advanced Kubernetes | Coming soon |
| 10 | Monitoring & Logging | Coming soon |
| 11 | Security (DevSecOps) | Coming soon |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction.md">Next: Introduction to Kubernetes →</a>
</div>
