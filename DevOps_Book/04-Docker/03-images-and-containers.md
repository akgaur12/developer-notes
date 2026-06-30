# Chapter 3 — Images & Containers

## Learning Objectives

By the end of this chapter you will be able to:

- Explain what a Docker image is and how it is identified
- Search for, pull, and inspect images from Docker Hub
- Run containers with the most important flags
- Manage container lifecycle: view, inspect, exec, log, stop, remove
- Use cleanup commands to reclaim disk space
- Understand image tags and why `latest` is dangerous in production

---

## 3.1 What Is a Docker Image?

A Docker image is a **read-only, self-contained template** that includes everything needed to run an application:

- Application code or binary
- Language runtime (Python interpreter, JRE, Node.js, etc.)
- System libraries and dependencies
- Environment variable defaults
- Filesystem layout and configuration files
- Instructions for what command to run at startup

### Image Identification

Images are identified by a **name:tag** pair:

| Example | Meaning |
|---------|---------|
| `nginx` | Official nginx image, implicit `:latest` tag |
| `nginx:1.25` | Specific version of nginx |
| `nginx:1.25-alpine` | Version 1.25 built on Alpine Linux |
| `python:3.11-slim` | Python 3.11, minimal Debian variant |
| `myuser/myapp:v2.1.0` | User-published image on Docker Hub |
| `ghcr.io/myorg/myapp:sha-abc123` | Image on GitHub Container Registry |

### The `latest` Tag

`latest` is just a conventional tag name — it means "the last image the maintainer tagged as latest." It is **not** automatically the newest version and is **not** updated atomically. Using `latest` in production means:

- You cannot reproduce a deployment later
- A `docker pull` might silently get a different version
- Debugging becomes harder ("which nginx was running last Tuesday?")

**Always use specific version tags in production.**

### Image ID

Every image is identified internally by a **SHA256 content hash** of its manifest:

```
sha256:61395b4c586da2b9b3b7ca903ea6a448e6719e9b2b0e95c76461e89edc3b0f2e
```

This hash changes whenever any layer changes. It ensures images are immutable and tamper-evident.

---

## 3.2 Finding and Pulling Images

### Searching for Images

```bash
# Search Docker Hub
docker search nginx

# Filter by star count (community quality signal)
docker search --filter stars=100 python

# Filter for official images only
docker search --filter is-official=true node
```

Official images are maintained by Docker or the upstream project and live at the root of Docker Hub (e.g., `nginx`, `postgres`, `python`). Community images are prefixed with a username: `bitnami/nginx`, `grafana/grafana`.

For production, prefer official images or images from trusted organizations (bitnami, chainguard, etc.).

### Pulling Images

```bash
# Pull the latest tag (not recommended for production)
docker pull nginx

# Pull a specific version
docker pull nginx:1.25

# Pull with a specific variant
docker pull nginx:1.25-alpine

# Pull from a non-Docker Hub registry
docker pull ghcr.io/myorg/myapp:v1.2.3
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
```

### Listing and Inspecting Local Images

```bash
# List all local images
docker images
docker image ls

# Formatted output
docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}"

# Filter by name
docker images nginx

# Show image digests (SHA256)
docker images --digests nginx

# Detailed JSON metadata
docker image inspect nginx

# Show layers (name, size, command used to create each)
docker history nginx
docker history --no-trunc nginx   # don't truncate the command column
```

### Image Name Anatomy

```
[registry/][namespace/]name[:tag][@digest]

nginx                           → Docker Hub official image
bitnami/nginx                   → Docker Hub user/org image
ghcr.io/myorg/myapp:v1.0        → GitHub Container Registry
123.dkr.ecr.us-east-1.amazonaws.com/app:latest  → AWS ECR
```

### Image Variants — Choosing the Right Base

| Variant | Base | Size | Use Case |
|---------|------|------|----------|
| `ubuntu` / `debian` | Full Debian/Ubuntu | ~70–120 MB | Familiar tools, easy debugging |
| `-slim` | Minimal Debian | ~20–40 MB | Production: smaller attack surface |
| `-alpine` | Alpine Linux (musl libc) | ~5–15 MB | Smallest size; note: musl ≠ glibc |
| `-bullseye` / `-bookworm` | Specific Debian release | ~70 MB | Pin to a specific Debian version |
| `-scratch` | Empty filesystem | ~0 B | Statically compiled binaries (Go, Rust) |

**Alpine caveat:** Alpine uses musl libc instead of glibc. Some compiled software (especially Python C extensions) behaves differently or fails to compile on Alpine. Test thoroughly before using Alpine for anything non-trivial.

---

## 3.3 Running Containers

The `docker run` command combines `docker create` + `docker start`. It has many options — here are the ones you'll use most.

### Basic Patterns

```bash
# Simplest: run and exit
docker run alpine echo "hello world"

# Interactive shell (removes container after exit)
docker run -it --rm ubuntu bash
docker run -it --rm python:3.11 python3

# Detached (background), named
docker run -d --name my-nginx nginx

# Detached with port mapping
docker run -d -p 8080:80 --name web nginx
```

### Port Mapping

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx

# Bind to localhost only (not exposed to the network)
docker run -d -p 127.0.0.1:8080:80 nginx

# Map multiple ports
docker run -d -p 80:80 -p 443:443 nginx

# Let Docker assign a random host port
docker run -d -p 80 nginx
docker port my-nginx              # find which host port was assigned
```

### Environment Variables

```bash
# Pass individual variables
docker run -d -e POSTGRES_PASSWORD=secret postgres
docker run -d \
  -e DB_HOST=db \
  -e DB_PORT=5432 \
  -e APP_ENV=production \
  my-app:latest

# Pass from a file (one VAR=value per line)
docker run -d --env-file .env my-app:latest
```

### Resource Limits

```bash
# Memory limit (container gets OOM-killed if exceeded)
docker run -d --memory=256m nginx

# CPU limit (1.0 = one full core; 0.5 = half a core)
docker run -d --cpus=1.0 nginx
docker run -d --cpus=0.5 nginx

# Memory + CPU together
docker run -d --memory=512m --cpus=2.0 my-app:latest

# CPU shares (relative weight, default 1024)
docker run -d --cpu-shares=512 low-priority-job
```

### Restart Policies

```bash
# Never restart (default)
docker run --restart=no nginx

# Always restart (even after docker stop + docker start daemon)
docker run --restart=always nginx

# Restart unless explicitly stopped
docker run --restart=unless-stopped nginx

# Restart on failure only (useful for batch jobs)
docker run --restart=on-failure:3 my-job   # max 3 retries
```

In production, `--restart=unless-stopped` is the most common policy for long-running services.

---

## 3.4 Managing Containers

### Viewing Containers

```bash
# Running containers only
docker ps

# All containers (including stopped)
docker ps -a

# Custom formatted output
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Filter by status
docker ps -f status=exited
docker ps -f name=nginx

# Show only container IDs (useful in scripts)
docker ps -q              # running
docker ps -aq             # all
```

### Viewing Logs

```bash
# Print all logs so far
docker logs my-nginx

# Follow logs in real time (like tail -f)
docker logs -f my-nginx

# Show last 50 lines
docker logs --tail 50 my-nginx

# Include timestamps
docker logs -t my-nginx

# Follow from last 10 lines
docker logs -f --tail 10 my-nginx

# Filter by time
docker logs --since 10m my-nginx      # last 10 minutes
docker logs --since 2024-01-01 my-nginx
```

### Executing Commands in a Running Container

```bash
# Get an interactive shell in a running container
docker exec -it my-nginx bash
docker exec -it my-nginx sh        # use sh if bash isn't available (alpine)

# Run a one-off command
docker exec my-nginx nginx -t      # test nginx config
docker exec my-nginx cat /etc/nginx/nginx.conf

# Run as a specific user
docker exec -u root my-nginx bash
docker exec -u 1000 my-app bash

# Set environment variables for the exec session
docker exec -e DEBUG=true my-app python debug_script.py
```

### Inspecting Containers

```bash
# Full JSON metadata
docker inspect my-nginx

# Extract specific fields with Go template syntax
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-nginx
docker inspect --format '{{.State.Status}}' my-nginx
docker inspect --format '{{.HostConfig.Memory}}' my-nginx
docker inspect --format '{{json .NetworkSettings.Ports}}' my-nginx

# Process list inside container (from host)
docker top my-nginx

# Live resource usage (CPU, memory, I/O, network)
docker stats
docker stats my-nginx             # single container
docker stats --no-stream          # snapshot (don't keep refreshing)
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### Copying Files

```bash
# Copy from container to host
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf

# Copy from host to container
docker cp ./app.conf my-nginx:/etc/nginx/conf.d/

# Copy a directory
docker cp my-nginx:/var/log/nginx/ ./nginx-logs/
```

Note: `docker cp` works even with stopped containers.

---

## 3.5 Container Lifecycle Management

### Starting and Stopping

```bash
# Stop gracefully (SIGTERM, then SIGKILL after 10s timeout)
docker stop my-nginx

# Stop with custom grace period
docker stop --time 30 my-nginx

# Start a stopped container
docker start my-nginx

# Restart (stop + start)
docker restart my-nginx
docker restart --time 5 my-nginx

# Pause / unpause (sends SIGSTOP/SIGCONT)
docker pause my-nginx
docker unpause my-nginx
```

### Removing Containers

```bash
# Remove a stopped container
docker rm my-nginx

# Force remove (running or stopped)
docker rm -f my-nginx

# Remove multiple containers
docker rm container1 container2 container3

# Remove all stopped containers
docker container prune

# Remove all stopped containers without confirmation prompt
docker container prune -f
```

### Disk Cleanup

Docker disk usage grows silently. Regular cleanup is important, especially on CI/CD runners.

```bash
# Show disk usage summary
docker system df

# Verbose breakdown (per image, container, volume)
docker system df -v

# Remove dangling images (untagged layers from old builds)
docker image prune

# Remove all unused images (not just dangling)
docker image prune -a

# Remove stopped containers
docker container prune

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Nuclear option: remove everything unused (containers, networks, images)
docker system prune

# Include unused images (not just dangling)
docker system prune -a

# Skip confirmation prompt (useful in scripts)
docker system prune -af
```

---

## 3.6 Committing Container Changes (and Why Not To)

Docker lets you snapshot a running container's filesystem state into a new image:

```bash
# Make some changes in a container
docker run -it --name temp-ubuntu ubuntu bash
# apt update && apt install -y curl vim
# exit

# Commit the changes to a new image
docker commit temp-ubuntu my-ubuntu-with-tools:v1

# Run from the committed image
docker run -it my-ubuntu-with-tools:v1 bash
```

### Why This Is an Anti-Pattern

- **Not reproducible** — no one can see what commands you ran to produce this image
- **No audit trail** — the image is a black box; can't diff it or review it
- **Hard to maintain** — updating means running a container, making changes, committing again
- **Security risk** — easy to accidentally commit credentials, cache, or sensitive files
- **Can't be automated** — doesn't fit into CI/CD pipelines

**Always use Dockerfiles to build images.** Commits are useful for quick debugging snapshots, but should never be used for production images. Chapter 4 covers Dockerfiles in depth.

---

## 3.7 Image Tags and Versioning

### Tagging Images

```bash
# Tag an existing image with a new name/tag
docker tag nginx:latest my-registry.io/myteam/nginx:1.25

# Tag your application image
docker tag my-app:latest my-app:v2.1.0
docker tag my-app:v2.1.0 ghcr.io/myorg/my-app:v2.1.0

# Multiple tags for the same image (same SHA, different aliases)
docker tag my-app:v2.1.0 my-app:v2.1
docker tag my-app:v2.1.0 my-app:v2
docker tag my-app:v2.1.0 my-app:latest   # also update latest

# Verify all tags point to the same image ID
docker images my-app
```

### Semantic Versioning Strategy for Images

A common tagging strategy for production images:

| Tag | Meaning | Stability |
|-----|---------|-----------|
| `my-app:v2.1.0` | Exact version | Immutable — never changes |
| `my-app:v2.1` | Latest patch in 2.1.x | Updates on patch releases |
| `my-app:v2` | Latest minor in 2.x | Updates on minor/patch releases |
| `my-app:latest` | Latest stable release | Updates on every release |
| `my-app:main-abc1234` | Git branch + short SHA | Per-commit CI builds |

In Kubernetes or Docker Compose, **always pin to an immutable tag** (`v2.1.0` or a digest) in production manifests. Use `latest` only in development.

### Removing Images

```bash
# Remove a specific image
docker rmi nginx:1.25
docker image rm nginx:1.25

# Remove by image ID (removes all tags pointing to that ID)
docker rmi <image_id>

# Force remove (even if containers are using it)
docker rmi -f nginx

# Remove all local images (use with caution)
docker rmi $(docker images -q)
```

---

## Summary

- A Docker image is an immutable, layered template identified by `name:tag` or SHA256 digest
- Image variants: full `-slim` for smaller size; `-alpine` for minimal size (musl libc caveat)
- `docker run` is the central command; key flags: `-it`, `-d`, `-p`, `-e`, `--rm`, `--name`, `--memory`, `--cpus`, `--restart`
- Managing containers: `docker ps`, `docker logs`, `docker exec`, `docker inspect`, `docker stats`
- Cleanup: `docker system prune`, `docker image prune`, `docker container prune`
- Always pin image versions in production; never use `latest` in deployment manifests
- Never use `docker commit` for production images — use Dockerfiles

---

## Knowledge Check

1. What is the difference between `docker pull` and `docker run` with respect to images?
2. You want to run nginx in the background and map host port 9090 to container port 80. What is the exact command?
3. How do you view the live output from a container that's already running in detached mode?
4. What does `docker system prune -a` do, and when would you want to use it?
5. Why is using the `latest` tag dangerous in production deployments?

---

## Hands-On Exercise

Complete the following steps, referencing the commands in this chapter:

1. **Pull three images** — `nginx:alpine`, `postgres:15`, `python:3.11-slim`

2. **Inspect them:**
   ```bash
   docker history nginx:alpine        # how many layers? how small?
   docker history postgres:15         # how many layers?
   docker image ls                    # compare sizes
   ```

3. **Run each:**
   ```bash
   # nginx: background, port 8080
   docker run -d --name demo-nginx -p 8080:80 nginx:alpine
   curl http://localhost:8080

   # postgres: background with required env var
   docker run -d --name demo-postgres \
     -e POSTGRES_PASSWORD=mysecret \
     -e POSTGRES_DB=testdb \
     postgres:15

   # python: interactive, auto-remove
   docker run -it --rm python:3.11-slim python3
   # Try: import sys; print(sys.version)
   # Exit with Ctrl+D
   ```

4. **Explore running containers:**
   ```bash
   docker ps
   docker stats --no-stream
   docker inspect --format '{{.NetworkSettings.IPAddress}}' demo-nginx
   docker logs demo-postgres
   docker exec -it demo-postgres psql -U postgres
   # \l    (list databases)
   # \q    (quit)
   ```

5. **Clean up:**
   ```bash
   docker stop demo-nginx demo-postgres
   docker rm demo-nginx demo-postgres
   docker system df       # see current disk usage
   docker image prune     # remove dangling images
   ```

**Goal:** You should be comfortable starting and stopping containers, reading logs, and exec-ing into a running container.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./02-architecture-and-internals.md">← Previous: Docker Architecture & Internals</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./04-dockerfile.md">Next: Writing Dockerfiles →</a>
</div>
