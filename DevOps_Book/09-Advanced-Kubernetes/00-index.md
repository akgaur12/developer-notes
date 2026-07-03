# Advanced Kubernetes — Complete Course Index

> **DevOps Learning Path — Topic 9 of 11**

---

## Course Overview

Kubernetes Basics (Topic 8) taught you how to deploy, configure, expose, and scale applications on a cluster someone else already built and secured for you. This course is about being that someone: the platform/cluster-admin skills needed to run Kubernetes safely for real teams and real production traffic.

You'll learn how to control *who* can do *what* on a cluster (RBAC), enforce security and compliance rules automatically (admission control, Pod Security Standards), lock down Pod-to-Pod traffic (NetworkPolicies), extend Kubernetes itself with your own APIs (CRDs and Operators), safely share one cluster across many teams (multi-tenancy), manage service-to-service traffic and security at scale (service mesh), and deploy changes the way modern platform teams do it — via Git, not via someone's laptop (GitOps). The back half of the course covers the unglamorous but essential operational skills: upgrading a live cluster, backing it up, running it across regions, tuning autoscaling, and troubleshooting it at 3 AM when something big breaks.

By the end, you'll have the skills expected of a platform engineer or senior DevOps/SRE responsible for Kubernetes as shared infrastructure — not just a team deploying an app to it.

---

## What You'll Be Able to Do

- Design and implement least-privilege RBAC for users, teams, and workloads
- Enforce security policy automatically using admission controllers, Pod Security Standards, and policy engines (OPA/Gatekeeper, Kyverno)
- Lock down cluster networking with NetworkPolicies and understand zero-trust networking in Kubernetes
- Extend the Kubernetes API with Custom Resource Definitions and understand how Operators automate operational knowledge
- Design multi-tenant clusters that safely isolate teams sharing the same infrastructure
- Explain what a service mesh solves and when you actually need one
- Implement GitOps-based deployment pipelines with Argo CD or Flux, including progressive delivery
- Perform safe cluster upgrades, node maintenance, and etcd backup/restore
- Design backup and disaster recovery strategies for stateful, cluster-based workloads
- Reason about multi-cluster architectures and when to use them
- Tune autoscaling (HPA/VPA/Cluster Autoscaler/Karpenter) and diagnose performance problems at cluster scale
- Troubleshoot cluster-wide failures using audit logs, `kubectl debug`, and node-level tooling
- Apply production-grade best practices and avoid the most common advanced Kubernetes mistakes
- Answer platform-engineering-level Kubernetes interview questions, including system design and incident scenarios

---

## Prerequisites

- **Kubernetes Basics (Topic 8)** — required. This course assumes fluency with Pods, Deployments, Services, ConfigMaps/Secrets, storage, namespaces, health checks, Ingress, StatefulSets/DaemonSets/Jobs, Helm, and HPA. Nothing from Topic 8 is re-taught here.
- **Infrastructure as Code — Terraform (Topic 7)** — recommended. Several chapters reference provisioning cluster infrastructure declaratively.
- **CI/CD Pipelines (Topic 5)** — recommended. GitOps (Chapter 8) builds directly on CI/CD concepts, contrasting push-based pipelines with pull-based deployment.

---

## Estimated Learning Timeline

**4–5 weeks** at 1–2 hours/day.

---

## Learning Milestones

| Milestone | Chapters | Skills Unlocked |
|-----------|----------|-----------------|
| Security & Isolation | 01–04 | RBAC, admission control, Pod Security Standards, NetworkPolicies |
| Extending & Sharing the Cluster | 05–08 | CRDs/Operators, multi-tenancy, service mesh, GitOps |
| Cluster Operations | 09–13 | Upgrades, backup/DR, multi-cluster, autoscaling, troubleshooting at scale |
| Professional | 14–18 | Best practices, avoiding common mistakes, capstone projects, interview-ready |

---

## Full Chapter Index

| # | Chapter | Topics |
|---|---------|--------|
| 01 | [Introduction to Advanced Kubernetes](./01-introduction.md) | Who this course is for, platform-engineer mindset, recap of what Topic 8 already covered, CKA/CKAD overview |
| 02 | [RBAC and Authentication](./02-rbac-and-authentication.md) | Authentication (users, ServiceAccounts, certs, OIDC), Roles, ClusterRoles, RoleBindings, least privilege |
| 03 | [Admission Control and Pod Security](./03-admission-control-and-pod-security.md) | Admission webhooks, Pod Security Standards/Admission, OPA/Gatekeeper, Kyverno |
| 04 | [Network Policies](./04-network-policies.md) | Default-allow networking, NetworkPolicy ingress/egress rules, zero-trust networking, CNI requirements |
| 05 | [Custom Resources and Operators](./05-custom-resources-and-operators.md) | CRDs, the Operator pattern, controller-runtime/kubebuilder, real-world Operator examples |
| 06 | [Multi-Tenancy](./06-multi-tenancy.md) | Soft vs hard multi-tenancy, namespace isolation, virtual clusters, hierarchical namespaces |
| 07 | [Service Mesh](./07-service-mesh.md) | The sidecar proxy pattern, mTLS, traffic management, Istio/Linkerd, when you actually need a mesh |
| 08 | [GitOps and Progressive Delivery](./08-gitops-and-progressive-delivery.md) | GitOps principles, Argo CD vs Flux, pull-based deployment, canary/blue-green with Argo Rollouts |
| 09 | [Cluster Administration and Upgrades](./09-cluster-administration-and-upgrades.md) | Cluster lifecycle, kubeadm, upgrading control plane/nodes, draining, etcd backup/restore |
| 10 | [Backup and Disaster Recovery](./10-backup-and-disaster-recovery.md) | Velero, stateful application backup strategies, DR planning, RTO/RPO |
| 11 | [Multi-Cluster Architectures](./11-multi-cluster-architectures.md) | Why multi-cluster, Cluster API, fleet management, multi-cluster service discovery |
| 12 | [Autoscaling and Performance Tuning](./12-autoscaling-and-performance-tuning.md) | Advanced HPA/VPA, Karpenter vs Cluster Autoscaler, resource tuning, performance diagnosis |
| 13 | [Auditing and Troubleshooting at Scale](./13-auditing-and-troubleshooting-at-scale.md) | Audit logging, `kubectl debug`, node-level tooling (crictl/etcdctl), cluster-wide failure scenarios |
| 14 | [Best Practices](./14-best-practices.md) | Production-grade cluster administration patterns |
| 15 | [Common Mistakes and Pitfalls](./15-common-mistakes.md) | The most frequent advanced Kubernetes/platform mistakes, why they happen, how to fix them |
| 16 | [Hands-On Projects](./16-projects.md) | Beginner → intermediate → advanced → production-grade capstone projects |
| 17 | [Interview Preparation](./17-interview-preparation.md) | Platform-engineering questions, scenario-based incidents, system design |
| 18 | [Course Summary](./18-course-summary.md) | Recap, checklist, quick reference, what's next |

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
| 8 | [Kubernetes Basics](../08-Kubernetes-Basics/00-index.md) | ✅ Complete |
| 9 | **Advanced Kubernetes** | 📍 You are here |
| 10 | Monitoring & Logging | Coming soon |
| 11 | Security (DevSecOps) | Coming soon |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction.md">Next: Introduction to Advanced Kubernetes →</a>
</div>
