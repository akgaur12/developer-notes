# Chapter 17 — Course Summary

## You've Completed Docker!

This chapter reviews everything covered, provides a completion checklist, and points you toward the next topic.

---

## What You Learned

### Foundations (Chapters 01–03)
- **Containers vs VMs**: containers share the host kernel, use namespaces and cgroups for isolation; VMs run their own OS
- **Docker Architecture**: Client → Daemon (dockerd) → containerd → runc; image layers via OverlayFS
- **Images**: read-only, layered, content-addressable; variants (alpine, slim, distroless, scratch)
- **Containers**: running image instances; lifecycle (created → running → stopped → removed)

### Core Skills (Chapters 04–06)
- **Dockerfile**: all 14 instructions, ENTRYPOINT vs CMD, layer caching, .dockerignore
- **Multi-stage builds**: separate build/runtime, BuildKit cache mounts and secret mounts, size reductions 75–99%
- **Volumes**: named volumes (recommended) vs bind mounts (dev) vs tmpfs (ephemeral); backup/restore patterns

### Infrastructure (Chapters 07–09)
- **Networking**: bridge/host/overlay drivers; user-defined networks for DNS-based service discovery; port publishing; network isolation
- **Docker Compose**: multi-service YAML, healthchecks, depends_on, .env secrets, override files, log rotation
- **Registry**: Docker Hub, GHCR, ECR; tagging strategy (semantic + SHA); Trivy scanning; Cosign signing; lifecycle policies

### Advanced (Chapters 10–12)
- **Security**: non-root USER, read-only filesystems, capability dropping, seccomp, secret management, distroless images, Docker socket risk
- **Intermediate Concepts**: resource limits, health checks in depth, restart policies, logging drivers, debugging strategies
- **Advanced Concepts**: BuildKit multi-platform, OCI specs, Swarm overview, networking internals (veth pairs, iptables NAT), container introspection

---

## Completion Checklist

### Beginner
```
□ Run containers from Docker Hub images (nginx, postgres, python)
□ Map ports and inspect running containers
□ Write a basic Dockerfile with FROM, WORKDIR, COPY, RUN, CMD
□ Build and tag an image
□ Understand what .dockerignore does and why it matters
□ Use docker logs, docker exec, docker inspect
```

### Intermediate
```
□ Write a production Dockerfile with non-root user and healthcheck
□ Build a multi-stage Dockerfile and measure the size difference
□ Run a multi-container stack with Docker Compose
□ Use named volumes and verify data persists across container removal
□ Configure a user-defined network and verify DNS resolution by name
□ Use .env file for secrets in Compose
□ Scan an image with Trivy and interpret the output
```

### Advanced
```
□ Implement BuildKit cache mounts for fast repeated builds
□ Build a multi-platform image with docker buildx
□ Drop all capabilities and add back only what's needed
□ Set memory and CPU limits and observe OOM kill
□ Configure log rotation with max-size and max-file
□ Push to a private registry (GHCR or ECR) from a CI pipeline
□ Design a Compose stack with proper network segmentation
```

---

## Key Commands Reference

```bash
# ─── Images ────────────────────────────────────────────────────────
docker build -t myapp:1.0 .                   # build from Dockerfile
docker build --target builder -t myapp:test . # build specific stage
docker image ls                               # list images
docker image prune                            # remove dangling images
docker pull nginx:alpine                      # pull from registry
docker push myregistry/myapp:1.0             # push to registry
docker history myapp:1.0                      # show layers

# ─── Containers ────────────────────────────────────────────────────
docker run -d --name myapp -p 3000:3000 myapp:1.0
docker run -it --rm alpine sh                # interactive, auto-remove
docker run --memory=512m --cpus=1.0 myapp:1.0
docker ps                                     # running containers
docker ps -a                                  # all containers
docker logs -f myapp                          # follow logs
docker exec -it myapp sh                      # exec into container
docker stop myapp && docker rm myapp
docker stats                                  # live resource usage

# ─── Volumes ───────────────────────────────────────────────────────
docker volume create mydata
docker volume ls && docker volume inspect mydata
docker run -v mydata:/data myapp
docker run -v $(pwd):/app:ro myapp           # bind mount, read-only

# ─── Networks ──────────────────────────────────────────────────────
docker network create mynet
docker network ls && docker network inspect mynet
docker run --network mynet --name svc1 myapp
docker exec svc1 ping svc2                   # DNS by name

# ─── Compose ───────────────────────────────────────────────────────
docker compose up -d                          # start all
docker compose up -d --build                  # rebuild and start
docker compose logs -f api                    # follow service logs
docker compose exec db psql -U app -d appdb  # exec into service
docker compose down                           # stop and remove
docker compose down -v                        # also remove volumes

# ─── Registry ──────────────────────────────────────────────────────
docker tag myapp:1.0 ghcr.io/myorg/myapp:1.0
docker push ghcr.io/myorg/myapp:1.0
trivy image myapp:1.0                         # vulnerability scan
docker scout cves myapp:1.0                   # Docker built-in scan

# ─── Cleanup ───────────────────────────────────────────────────────
docker system df                              # disk usage
docker system prune                           # remove unused
docker system prune -a                        # remove ALL unused images
```

---

## Mental Models to Keep

**"Containers are processes, not machines"**
> A container is just a Linux process with restricted visibility (namespaces) and limited resources (cgroups). The process runs directly on the host kernel — there's no hypervisor. This is why containers start instantly and why kernel exploits in a container can affect the host.

**"Images are immutable; containers are ephemeral"**
> Never rely on data written inside a container. Everything in the container filesystem beyond the image layers is lost on `docker rm`. Persist anything important to a named volume.

**"Order matters in Dockerfile for caching"**
> Think of the Dockerfile as a cache hierarchy. Docker re-runs every instruction from the first changed instruction onward. Put stable things (OS packages, dependencies) near the top; volatile things (source code) near the bottom.

**"User-defined networks = automatic DNS"**
> On the default bridge, containers can only reach each other by IP. On a user-defined network, Docker provides automatic DNS resolution by container name. Always create custom networks in Compose.

---

## What's Next

You've completed Topic 4: Docker. The next topic in the DevOps roadmap is:

**Topic 5: CI/CD Pipelines**
- GitHub Actions: workflows, triggers, jobs, steps
- Building and testing in CI
- Docker build and push in CI
- Deployment strategies: rolling, blue/green, canary
- Secrets management in CI/CD
- GitLab CI and Jenkins overview
- Production deployment pipelines

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="16-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../05-CI-CD-Pipelines/00-index.md">Next: CI/CD Pipelines →</a>
</div>
