# Docker — Complete Course Index

> **DevOps Learning Path — Topic 4 of 11**

Docker is the industry standard for packaging and running applications in containers. It solves the perennial "it works on my machine" problem by bundling an application together with everything it needs to run — code, runtime, libraries, environment variables, and configuration — into a single portable unit. Today, over 96% of modern production deployments use containers in some form, and Docker is the tool that made containers mainstream.

## What You'll Be Able to Do After This Course

- Build Docker images from scratch using Dockerfiles
- Write efficient, production-ready Dockerfiles with proper layer caching
- Run, inspect, and manage containers confidently
- Use Docker Compose to orchestrate multi-container applications
- Understand Docker's networking model and connect services together
- Mount volumes for persistent storage and bind mounts for development
- Push and pull images from public and private registries
- Apply Docker security best practices for production workloads
- Diagnose and avoid the most common Docker mistakes

## Prerequisites

Before starting this course you should have completed:

- **Linux Fundamentals** (Topic 1) — filesystem, permissions, processes, package management
- **Git & Version Control** (Topic 3) — you'll version-control Dockerfiles and Compose files

You should also be comfortable with:

- Working in a terminal / command line
- Basic networking concepts (IP addresses, ports, TCP/UDP)
- Text editors such as `vim`, `nano`, or VS Code

## Estimated Learning Timeline: 3–4 Weeks

| Milestone | Chapters | Skills Unlocked |
|-----------|----------|-----------------|
| **Beginner** | 01–04 | Run containers, pull images, write basic Dockerfiles, understand core concepts |
| **Intermediate** | 05–08 | Multi-stage builds, volumes and persistent storage, container networking, Docker Compose |
| **Advanced** | 09–12 | Registry management, Docker security, internals deep-dive, advanced patterns |
| **Professional** | 13–17 | Production best practices, common mistakes to avoid, hands-on projects, interview ready |

---

## Full Chapter Index

| # | Chapter | Topics |
|---|---------|--------|
| 00 | [Course Index](./00-index.md) | Overview, prerequisites, learning path |
| 01 | [Introduction to Docker](./01-introduction.md) | What is Docker, containers vs VMs, installation, first container |
| 02 | [Docker Architecture & Internals](./02-architecture-and-internals.md) | Daemon, namespaces, cgroups, OverlayFS, container lifecycle |
| 03 | [Images & Containers](./03-images-and-containers.md) | Pulling images, running containers, managing lifecycle, cleanup |
| 04 | [Writing Dockerfiles](./04-dockerfile.md) | Dockerfile instructions, best practices, layer caching |
| 05 | [Multi-stage Builds](./05-multi-stage-builds.md) | Build stages, minimizing image size, builder pattern |
| 06 | [Volumes & Storage](./06-volumes-and-storage.md) | Named volumes, bind mounts, tmpfs, storage drivers |
| 07 | [Docker Networking](./07-networking.md) | Bridge, host, overlay networks, DNS, port publishing |
| 08 | [Docker Compose](./08-docker-compose.md) | Compose file syntax, services, networks, volumes, profiles |
| 09 | [Registry & Image Management](./09-registry-and-image-management.md) | Docker Hub, private registries, ECR/GCR, tagging strategy |
| 10 | [Docker Security](./10-security.md) | Non-root users, capabilities, secrets, image scanning, AppArmor |
| 11 | [Intermediate Concepts](./11-intermediate-concepts.md) | Health checks, logging drivers, resource constraints, init processes |
| 12 | [Advanced Concepts](./12-advanced-concepts.md) | BuildKit, buildx, multi-platform images, Docker plugins |
| 13 | [Best Practices](./13-best-practices.md) | Production patterns, image hygiene, CI/CD integration |
| 14 | [Common Mistakes](./14-common-mistakes.md) | Anti-patterns, debugging, pitfalls to avoid |
| 15 | [Hands-On Projects](./15-projects.md) | Containerize a Node app, Python API, multi-service stack |
| 16 | [Interview Preparation](./16-interview-preparation.md) | Common questions, scenario-based problems, answers |
| 17 | [Course Summary](./17-course-summary.md) | Key takeaways, cheat sheet, next steps |

---

## DevOps Roadmap Series

| # | Course | Status |
|---|--------|--------|
| 1 | [Linux Fundamentals](../01-Linux-Fundamentals/00-index.md) | ✅ Complete |
| 2 | [Networking Basics](../02-Networking-Basics/00-index.md) | ✅ Complete |
| 3 | [Git & Version Control](../03-Git-Version-Control/00-index.md) | ✅ Complete |
| 4 | **Docker** | 📍 You are here |
| 5 | CI/CD Pipelines | 🔜 Coming soon |
| 6 | Kubernetes | 🔜 Coming soon |
| 7 | Infrastructure as Code (Terraform) | 🔜 Coming soon |
| 8 | Cloud Platforms (AWS/GCP/Azure) | 🔜 Coming soon |
| 9 | Monitoring & Observability | 🔜 Coming soon |
| 10 | Security & Compliance (DevSecOps) | 🔜 Coming soon |
| 11 | Site Reliability Engineering (SRE) | 🔜 Coming soon |

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <span></span>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./01-introduction.md">Next: Introduction to Docker →</a>
</div>
