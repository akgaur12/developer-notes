# DevOps Engineering — Complete Learning Path

A structured, self-paced course series taking you from zero to production-grade DevOps engineer. Each topic builds on the previous one — follow the order for the best learning experience.

---

## What This Is

A complete DevOps curriculum written as Markdown course files. Each course contains 16–18 chapters covering fundamentals through advanced concepts, with real-world examples, hands-on projects, and interview preparation.

**Who this is for:** Developers who want to move into DevOps, sysadmins learning modern tooling, or engineers filling gaps in their infrastructure knowledge.

**What you'll be able to do:** Design, deploy, and operate production systems — from managing Linux servers and configuring networks to orchestrating containers and building CI/CD pipelines.

---

## Prerequisites

- Basic comfort with the command line (typing commands, navigating directories)
- Any programming experience (Python, JavaScript, etc.)
- A Linux/macOS machine or a Linux VM to practice on

No prior DevOps experience required. Start from Topic 1.

---

## Course Roadmap

| # | Course | Key Topics | Chapters | Est. Time | Status |
|---|--------|-----------|----------|-----------|--------|
| 1 | [Linux Fundamentals](./01-Linux-Fundamentals/00-index.md) | Filesystem, permissions, processes, shell scripting, SSH | 17 | 3–4 weeks | ✅ Complete |
| 2 | [Networking Basics](./02-Networking-Basics/00-index.md) | OSI model, TCP/IP, DNS, HTTP/TLS, nginx, firewalls, load balancing | 16 | 3–4 weeks | ✅ Complete |
| 3 | [Git & Version Control](./03-Git-Version-Control/00-index.md) | Git internals, branching strategies, rebase, hooks, GitHub/GitLab workflows | 17 | 2–3 weeks | ✅ Complete |
| 4 | Docker | Images, containers, Dockerfile, Compose, volumes, networking | — | 3–4 weeks | Coming soon |
| 5 | CI/CD Pipelines | GitHub Actions, Jenkins, GitLab CI, pipeline design, testing | — | 3–4 weeks | Coming soon |
| 6 | Cloud Fundamentals (AWS) | EC2, S3, VPC, IAM, RDS, CloudFront, cost management | — | 4–5 weeks | Coming soon |
| 7 | Infrastructure as Code (Terraform) | HCL, state, modules, workspaces, remote backends | — | 3–4 weeks | Coming soon |
| 8 | Kubernetes Basics | Pods, deployments, services, ConfigMaps, ingress, Helm | — | 4–5 weeks | Coming soon |
| 9 | Advanced Kubernetes | RBAC, network policies, operators, multi-tenancy, GitOps | — | 3–4 weeks | Coming soon |
| 10 | Monitoring & Logging | Prometheus, Grafana, ELK stack, alerting, SLOs | — | 3–4 weeks | Coming soon |
| 11 | Security (DevSecOps) | Secrets management, SAST/DAST, supply chain, compliance | — | 3–4 weeks | Coming soon |

**Total estimated time:** 10–14 months at a comfortable pace (1–2 hours/day).

---

## How to Use This

1. **Start at the index.** Each course has a `00-index.md` with a learning timeline, chapter map, and milestones. Read it first.
2. **Go in order.** Chapters are numbered and each one builds on the previous.
3. **Do the exercises.** Each chapter ends with hands-on exercises. Do them — reading alone won't build the muscle memory.
4. **Use the projects.** Each course has dedicated project chapters (beginner → intermediate → advanced → capstone). These are the most important part.
5. **Check the checklist.** Each course summary has a completion checklist. Don't move on until you can tick every box.

---

## Directory Structure

```
DevOps_Book/
├── README.md                        ← you are here
├── 01-Linux-Fundamentals/
│   ├── 00-index.md
│   ├── 01-introduction.md
│   └── ...17 chapters
├── 02-Networking-Basics/
│   ├── 00-index.md
│   └── ...16 chapters
└── 03-Git-Version-Control/
    ├── 00-index.md
    └── ...17 chapters
```

Each course directory follows the same pattern:
- `00-index.md` — course overview, timeline, full chapter list
- `01-` through `NN-` — numbered chapters in sequence
- Final two chapters are always Projects and Interview Preparation
- Last chapter is always Course Summary with a checklist and command reference

---

## Learning Milestones

| Milestone | After Completing | You Can... |
|-----------|-----------------|------------|
| Junior SysAdmin | Topics 1–2 | Manage Linux servers, configure networking, debug connectivity issues |
| DevOps Engineer I | Topics 1–4 | Build and ship containerized applications |
| DevOps Engineer II | Topics 1–6 | Deploy to cloud, manage infrastructure, set up CI/CD |
| Senior DevOps / SRE | Topics 1–9 | Design and operate large-scale Kubernetes platforms |
| Platform Engineer | All 11 topics | Full-stack infrastructure: security, observability, compliance |
