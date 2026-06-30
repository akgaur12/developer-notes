# Chapter 2 — Docker Architecture & Internals

## Learning Objectives

By the end of this chapter you will be able to:

- Describe Docker's client-server architecture and how each component interacts
- Explain how Linux namespaces provide container isolation
- Explain how cgroups enforce resource limits
- Describe how OverlayFS enables Docker's image layer system
- Walk through what happens internally when you run `docker run nginx`
- Understand the container lifecycle states and the commands that transition between them

---

## 2.1 Docker Architecture Overview

Docker uses a **client-server architecture**. When you type `docker run nginx`, you're using the Docker **client** (CLI), which talks to the Docker **daemon** over a REST API.

### Components

| Component | Role |
|-----------|------|
| **Docker CLI** (`docker`) | The command-line interface you interact with |
| **Docker Daemon** (`dockerd`) | The server process; manages images, containers, networks, volumes |
| **containerd** | Mid-level runtime; manages the container lifecycle |
| **runc** | Low-level OCI runtime; actually creates and starts the container process |

### Architecture Diagram

```
[docker CLI] ──REST API──► [Docker Daemon (dockerd)]
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼             ▼
                    [containerd]  [Images]      [Networks]
                         │        [Volumes]
                         ▼
                      [runc]
                         │
                         ▼
                   [Container Process]
```

The CLI and daemon can run on the same machine (the default) or on different machines (remote Docker hosts).

### What Happens When You Run `docker run nginx`

1. You type `docker run nginx` in your terminal
2. The Docker CLI sends a `POST /containers/create` request to `dockerd` via the Unix socket
3. `dockerd` checks if the `nginx` image exists locally — it doesn't, so it pulls it from Docker Hub
4. `dockerd` asks `containerd` to create a container from the image
5. `containerd` calls `runc` to set up Linux namespaces, cgroups, and mount the filesystem
6. `runc` starts the nginx process inside the isolated environment, then exits (its job is done)
7. `containerd` continues to monitor the container; the nginx process is now PID 1 in its namespace

---

## 2.2 Linux Kernel Features Powering Docker

Docker is not magic — it's a user-friendly interface on top of three key Linux kernel features: **namespaces**, **cgroups**, and **union filesystems**.

### Namespaces — Isolation

Namespaces make each container believe it has its own private version of system resources. The kernel maintains separate namespace views for different groups of processes.

| Namespace | What It Isolates | Effect in a Container |
|-----------|-----------------|----------------------|
| **pid** | Process IDs | Container has its own PID 1; can't see host processes |
| **net** | Network stack | Own network interfaces, IP address, routing table, iptables rules |
| **mnt** | Filesystem mounts | Own root filesystem; can't see host mounts |
| **uts** | Hostname and domain name | Container can have its own hostname |
| **user** | User and group IDs | UID 0 inside container can map to unprivileged UID on host |
| **ipc** | IPC (semaphores, message queues) | Containers can't interfere with each other's IPC |

You can verify this yourself: start a container and check that `ps aux` inside shows only a handful of processes, while the host has hundreds.

### cgroups (Control Groups) — Resource Limits

Namespaces provide **isolation** (what a container can see); cgroups provide **limits** (how much a container can use). Without cgroups, one container could consume all CPU/memory and starve everything else on the host.

cgroups let Docker enforce:

- **CPU** — max CPU shares or hard CPU limits
- **Memory** — maximum RAM; OOM killer kills the container, not the host process
- **Block I/O** — limits on disk read/write speed
- **Network** — bandwidth throttling (via tc, not cgroups directly)

```bash
# Limit a container to 512 MB RAM and 1 CPU core
docker run --memory=512m --cpus=1.0 nginx

# Limit to 50% of one CPU core
docker run --cpus=0.5 nginx

# Soft memory limit (container can exceed during low-pressure periods)
docker run --memory=512m --memory-reservation=256m nginx
```

When a container exceeds its memory limit, the kernel's OOM (Out of Memory) killer terminates a process **inside the container** rather than on the host. This is the container-level equivalent of a process crash.

### Union Filesystems (OverlayFS) — Layered Images

Docker images are built from **layers**. Each instruction in a Dockerfile creates a new layer. OverlayFS is a Linux union filesystem that stacks these read-only layers on top of each other and presents them as a single unified filesystem.

```
┌────────────────────────────────────────┐
│     Writable Container Layer           │  ← Specific to this running container
│     (copy-on-write; lost when removed) │
├────────────────────────────────────────┤
│     Layer 3: COPY app /app             │  ← Read-only; from Dockerfile
├────────────────────────────────────────┤
│     Layer 2: RUN apt install python3   │  ← Read-only; shared across images
├────────────────────────────────────────┤
│     Layer 1: Base image (ubuntu:22.04) │  ← Read-only; shared across images
└────────────────────────────────────────┘
```

**Key properties:**

- Layers are **content-addressed** by SHA256 hash — if the content is identical, the layer is shared
- Read-only layers can be **shared** across many images and containers (saving disk space)
- When a container modifies a file from a lower layer, OverlayFS copies it up to the writable layer (**copy-on-write**)
- When a container is removed, only its writable layer is deleted; the shared layers remain

---

## 2.3 The Image Layer System

Understanding layers is critical for writing efficient Dockerfiles and diagnosing slow builds.

```bash
# See the layers and their sizes for an image
docker history nginx

# Expected output (newest at top):
# IMAGE         CREATED       CREATED BY                      SIZE
# <missing>     2 weeks ago   CMD ["nginx" "-g" "daemon off"]  0B
# <missing>     2 weeks ago   EXPOSE 80                        0B
# <missing>     2 weeks ago   COPY docker-entrypoint.sh ...    4.62kB
# <missing>     2 weeks ago   RUN apt-get install -y nginx     55.2MB
# ...

# Inspect full image metadata (includes layer SHA256 digests)
docker image inspect nginx

# Check disk usage
docker system df
docker system df -v   # verbose: per-image breakdown
```

### Layer Caching

Docker caches layers. When rebuilding an image, Docker compares the **instruction + context** for each layer. If nothing has changed since the last build, Docker reuses the cached layer — even if it's from a previous build of a different image.

**Cache invalidation rules:**
- `FROM` — cache miss if base image digest changed
- `RUN` — cache miss if the command text changed
- `COPY`/`ADD` — cache miss if the file content changed (based on checksum)
- Any cache miss invalidates **all subsequent layers**

This last rule is why order matters in Dockerfiles. Dependencies that change rarely (e.g., `package.json` installs) should come before code that changes often (e.g., `COPY . .`).

---

## 2.4 The Container Lifecycle

A container moves through defined states. Understanding these states helps you write correct orchestration scripts and debug stuck containers.

```
          ┌──────────────────────────────────────────────────┐
          │                                                    │
   docker create       docker start          docker pause
          │                  │                    │
          ▼                  ▼                    ▼
       [created] ──────► [running] ──────► [paused]
                             │                    │
                        docker stop          docker unpause
                             │                    │
                             ▼                    ▼
                          [stopped] ◄─────── [running]
                             │
                          docker rm
                             │
                             ▼
                          [deleted]
```

```bash
# Create a container without starting it
docker create --name my-nginx nginx

# Start a created (or stopped) container
docker start my-nginx

# Pause — sends SIGSTOP to all processes (freezes them in memory)
docker pause my-nginx

# Unpause — sends SIGCONT, resumes from exact state
docker unpause my-nginx

# Stop — sends SIGTERM, waits 10s, then sends SIGKILL
docker stop my-nginx

# Stop with a custom timeout
docker stop --time 30 my-nginx

# Kill — sends SIGKILL immediately (no graceful shutdown)
docker kill my-nginx

# Remove a stopped container
docker rm my-nginx

# Force-remove a running container (stop + rm in one step)
docker rm -f my-nginx

# Auto-remove when container exits (great for one-off tasks)
docker run --rm alpine echo "hello"
```

**Tip:** Always prefer `docker stop` over `docker kill` for stateful applications (databases, queues). `SIGTERM` gives the process a chance to flush buffers and clean up.

---

## 2.5 How `docker run` Works Internally

Let's trace exactly what happens at each layer:

```
You:        docker run -d -p 8080:80 --name web nginx

Step 1:     Docker CLI parses the command and builds a JSON payload.
            Sends POST /containers/create to dockerd via /var/run/docker.sock

Step 2:     dockerd checks if nginx:latest exists in the local image store.
            → Not found. Sends GET /v2/library/nginx/manifests/latest to Docker Hub.
            → Downloads image layers (OverlayFS layers in /var/lib/docker/overlay2/)

Step 3:     dockerd creates a container spec:
            - Namespace config (pid, net, mnt, uts, ipc)
            - cgroup config (no limits specified → inherits host defaults)
            - Network: create a veth pair; attach one end to docker0 bridge
            - Port mapping: iptables DNAT rule: host:8080 → container:80
            - Root filesystem: mount OverlayFS with nginx layers + writable layer

Step 4:     dockerd hands the spec to containerd.
            containerd prepares the bundle (rootfs + config.json per OCI spec).

Step 5:     containerd calls runc with the bundle.
            runc:
              - Creates Linux namespaces (unshare syscalls)
              - Sets up cgroup membership
              - Mounts the OverlayFS rootfs
              - Pivots root into the container filesystem
              - Executes the image's CMD: nginx -g "daemon off;"
              - runc exits; the nginx process is now PID 1 in its namespace

Step 6:     containerd monitors the container.
            dockerd records container state.
            CLI prints the container ID and returns you to the prompt.
```

---

## 2.6 The Docker Daemon API

The Docker daemon exposes a REST API over a Unix socket. The Docker CLI is just a client that speaks this API — you can call it directly with `curl`.

```bash
# Query the Docker API directly (requires access to the socket)
curl --unix-socket /var/run/docker.sock http://localhost/version

# List running containers
curl --unix-socket /var/run/docker.sock http://localhost/containers/json | python3 -m json.tool

# Get info about a specific container
curl --unix-socket /var/run/docker.sock \
  http://localhost/containers/$(docker ps -q -f name=my-nginx)/json | \
  python3 -m json.tool
```

### Security Implication

Access to `/var/run/docker.sock` is equivalent to **root access on the host**. Anyone who can write to that socket can start containers that mount the host filesystem, and escape to root.

This is why:
- Never expose the Docker socket to untrusted containers (e.g., `docker run -v /var/run/docker.sock:/var/run/docker.sock`)
- The `docker` group is privileged — only add trusted users
- In CI/CD, prefer rootless Docker or Podman
- In production, use Docker socket proxies (like Traefik's socket proxy) that limit API access

---

## Summary

- Docker uses a client-server architecture: `docker CLI` → `dockerd` → `containerd` → `runc`
- **Namespaces** provide isolation: each container gets its own view of PIDs, network, filesystem, hostname, and users
- **cgroups** enforce limits: CPU, memory, and I/O constraints prevent one container from starving others
- **OverlayFS** enables the layered image system: read-only shared layers + a per-container writable layer
- Container states: created → running → paused → stopped → deleted; use `stop` for graceful shutdown
- The Docker daemon API is a REST API on `/var/run/docker.sock`; socket access = root access

---

## Knowledge Check

1. What are the three Linux kernel features that power Docker's core functionality?
2. Which namespace isolates a container's network interfaces and IP address from the host?
3. What happens when a container exceeds its memory limit (set with `--memory`)?
4. Why does the order of instructions in a Dockerfile matter for build performance?
5. What is the difference between `docker stop` and `docker kill`?

---

## Hands-On Exercise

### Part A — Explore Namespace Isolation

```bash
# 1. Start a container in the background
docker run -d --name ns-test nginx

# 2. Find its PID on the host
docker inspect --format '{{.State.Pid}}' ns-test

# 3. Look at the container's namespaces from the host
ls -la /proc/<PID>/ns/

# 4. Compare with your shell's namespaces
ls -la /proc/$$/ns/

# 5. Enter the container and see its isolated process list
docker exec -it ns-test bash
ps aux   # should show only nginx processes
exit

# Cleanup
docker rm -f ns-test
```

### Part B — Explore cgroups

```bash
# 1. Run a memory-limited container
docker run -d --name mem-test --memory=128m nginx

# 2. Find the cgroup path
docker inspect --format '{{.HostConfig.Memory}}' mem-test
# Should print: 134217728 (128 * 1024 * 1024)

# 3. Check the cgroup filesystem
cat /sys/fs/cgroup/memory/docker/$(docker inspect --format '{{.Id}}' mem-test)/memory.limit_in_bytes

# Cleanup
docker rm -f mem-test
```

### Part C — Inspect Image Layers

```bash
# View layers for three images of different complexity
docker pull alpine
docker pull python:3.11-slim
docker pull ubuntu

docker history alpine
docker history python:3.11-slim
docker history ubuntu

# Compare sizes
docker images alpine python:3.11-slim ubuntu
```

**Questions to answer:** How many layers does alpine have? Which has the most layers? Which is largest?

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./01-introduction.md">← Previous: Introduction to Docker</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./03-images-and-containers.md">Next: Images & Containers →</a>
</div>
