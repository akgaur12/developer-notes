# Chapter 16 — Interview Preparation

**Learning Objectives**

---

## 16.1 Foundational Questions

**Q: What is the difference between a Docker image and a container?**
> An image is a read-only template containing application code, runtime, libraries, and configuration. A container is a running (or stopped) instance of an image. The relationship is like a class (image) vs an object (container). You can run multiple containers from one image.

**Q: How is a Docker container different from a virtual machine?**
> Containers share the host OS kernel and use Linux namespaces/cgroups for isolation. VMs emulate entire hardware and run their own OS kernel. Containers boot in milliseconds and are MBs in size; VMs take minutes and are GBs. VMs are more strongly isolated; containers are more efficient but depend on the host kernel.

**Q: Explain Docker image layers.**
> Images are composed of read-only layers, each corresponding to a Dockerfile instruction. Docker uses OverlayFS (a union filesystem) to stack layers. When a container starts, a thin writable layer is added on top. Layers are content-addressable and shared between images that have the same base — an `ubuntu:22.04` layer is stored only once even if 100 images use it. This makes pulls efficient and reduces disk usage.

**Q: What is the difference between CMD and ENTRYPOINT?**
> ENTRYPOINT defines the fixed executable; CMD provides default arguments. With both set, CMD becomes the arguments to ENTRYPOINT. CMD alone: the whole command, overridable by `docker run <image> <command>`. ENTRYPOINT alone: the command can't be overridden without `--entrypoint`. Best practice: use exec form (JSON array) for both to avoid shell wrapping and ensure signals reach PID 1.

**Q: What happens when a container exits?**
> The container's writable filesystem layer is preserved but the process is stopped. The container state becomes "Exited". Data in volumes persists; data only in the container filesystem is gone when `docker rm` is run. The exit code is recorded and visible via `docker inspect`.

**Q: What is a Docker volume vs a bind mount?**
> A volume is managed by Docker (`/var/lib/docker/volumes/`), portable, and the recommended way to persist data. A bind mount maps a specific host path into the container. Bind mounts are useful for development (live code reload) but create host-path dependency. Use volumes in production.

---

## 16.2 Architecture and Internals Questions

**Q: Explain the Docker architecture.**
> Client-server: the Docker CLI sends REST API calls to the Docker daemon (dockerd). The daemon manages images, containers, networks, and volumes. Underneath dockerd is containerd (manages container lifecycle) and runc (the OCI runtime that actually runs containers using Linux kernel features).

**Q: What Linux kernel features does Docker use?**
> Three main features:
> - **Namespaces**: isolation. pid (process tree), net (network stack), mnt (filesystem), uts (hostname), user, ipc. Each container gets its own namespace instances.
> - **cgroups (control groups)**: resource limits. CPU, memory, I/O throttling.
> - **Union filesystems (OverlayFS)**: layered image filesystem. Read-only base layers + writable container layer.

**Q: What is the Docker socket and why is it a security risk?**
> `/var/run/docker.sock` is the Unix socket the Docker CLI uses to communicate with the daemon. Any process with access to this socket can issue any Docker API call — including running privileged containers, mounting the host filesystem, and escalating to root. Mounting the socket into a container gives that container full host control. Only trusted tooling (CI runners, monitoring agents) should have socket access, and only when necessary.

---

## 16.3 Scenario-Based Questions

**Scenario 1: "Your containerized app works in dev but fails in production"**
```
Debugging approach:
1. Image consistency: same image? docker inspect for tags/digests
2. Environment: compare env vars between dev and prod
3. Network: different network topology? Service discovery different?
4. Volumes: is data missing that dev had (bind mount vs prod volume)?
5. Resources: are there memory/CPU limits in prod that aren't in dev?
6. Secrets: are secrets properly injected in prod?
7. Logs: docker logs <container>, look for startup errors
```

**Scenario 2: "Container keeps OOM-killing every hour"**
```
1. Confirm: docker inspect <container> | jq '.[0].State.OOMKilled'  → true
2. Check memory usage trend: docker stats --no-stream
3. Look at memory limit: docker inspect | grep Memory
4. Options:
   a. Increase the limit: --memory=1g
   b. Fix memory leak in application
   c. Add --memory-swap to allow swap as buffer
   d. Check for log accumulation eating memory (json-file driver)
```

**Scenario 3: "Image takes 10 minutes to build in CI"**
```
1. Check cache: is --cache-from being used?
2. Check order: are dependencies copied before source code?
3. Check build context: is .dockerignore excluding large dirs?
4. Check if BuildKit is enabled: DOCKER_BUILDKIT=1
5. Parallel stages: are independent stages running concurrently?
6. Registry cache: push cache layer to registry with BUILDKIT_INLINE_CACHE=1
```

**Scenario 4: "Container can't connect to another container"**
```
1. Same network? docker inspect both containers | grep NetworkMode
2. User-defined network? Default bridge doesn't support DNS names
3. Test: docker exec app1 ping app2  (by name)
4. Test: docker exec app1 nc -zv app2 5432  (by name + port)
5. Check firewall/security groups on the host
6. Check the target container is actually listening: docker exec app2 ss -tlnp
```

---

## 16.4 System Design Questions

**"Design the deployment strategy for a 3-tier web app using Docker"**

Key points to cover:
```
1. Separate images per tier (nginx, app, db)
2. Docker Compose for local dev and single-server production
3. Kubernetes for multi-server / high availability production
4. Image registry strategy: private registry, tagged with git SHA
5. Secret management: Vault or cloud secrets manager
6. Network isolation: frontend/backend networks
7. Persistent storage: named volumes for DB, cloud block storage in prod
8. Health checks on all services
9. Rolling deploys: docker service update (Swarm) or kubectl rollout (K8s)
10. Observability: centralized logging, Prometheus metrics
```

**"How would you handle database migrations in a containerized environment?"**
```
Common pattern: init container / migration container
- Run migration as a separate container that exits 0 on success
- depends_on with condition: service_completed_successfully
- Or: app runs migrations on startup (with idempotent migration scripts)
- Never run migrations inside the same container as the app (harder to track)
- Always test rollback: can you migrate down if the release fails?
```

---

## 16.5 Quick-Fire Questions

| Question | Answer |
|----------|--------|
| Default network driver? | bridge |
| How to run a container and auto-remove it? | `--rm` flag |
| Exec form vs shell form? | Exec form uses JSON array, no shell wrapping, signals reach PID 1 |
| What does `--init` do? | Runs tini as PID 1 for proper signal handling |
| What is layer caching? | Docker reuses unchanged layers; put volatile instructions last |
| What is a dangling image? | Untagged image (`<none>:<none>`) — intermediate build artifact |
| What port does Docker registry use? | 5000 (default) |
| What is `docker system prune`? | Removes stopped containers, dangling images, unused networks, build cache |
| Difference between `stop` and `kill`? | stop sends SIGTERM then SIGKILL after timeout; kill sends SIGKILL immediately |
| What is distroless? | Minimal images with no shell/package manager — smaller attack surface |

---

## 16.6 "Walk Me Through Your Docker Workflow"

STAR format example:
```
Situation: We had a monolithic app with inconsistent deployments — "works on my machine" problems
across 6 developers and 3 environments.

Task: Containerize the app and standardize the deployment pipeline.

Action:
1. Wrote a multi-stage Dockerfile (build → test → production stages)
2. Created docker-compose.yml for local dev with all dependencies
3. Added Trivy scanning to CI — caught 3 CRITICAL CVEs in base image before prod
4. Set up private ECR registry with lifecycle policies
5. Tagged images with git SHA for immutable traceability
6. Added health checks and restart policies
7. Wrote runbooks for common issues (OOM, connection failures)

Result: Zero "works on my machine" issues in 8 months. Deploy time dropped from 25 minutes
to 4 minutes. Image sizes reduced 85% with multi-stage builds.
```

**Knowledge Check** (5 questions), no exercise (interview prep IS the exercise)

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="15-projects.md">← Previous: Hands-On Projects</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="17-course-summary.md">Next: Course Summary →</a>
</div>
