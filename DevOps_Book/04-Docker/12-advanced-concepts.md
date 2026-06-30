# Chapter 12 — Advanced Concepts

## Learning Objectives

By the end of this chapter you will be able to:
- Use BuildKit cache mounts, SSH mounts, and inline caching to accelerate CI builds
- Build and push multi-platform images with `docker buildx`
- Explain OCI specifications and how they relate to Docker, containerd, and Kubernetes
- Deploy services in Docker Swarm and perform rolling updates and rollbacks
- Understand the Linux networking primitives Docker uses (veth pairs, bridges, iptables NAT)
- Design isolated multi-network topologies for service-to-service access control
- Manage multiple Docker endpoints with contexts
- Inspect image layers with `dive` and reference images by digest for immutable pinning

---

## 12.1 BuildKit Advanced Features

BuildKit is Docker's next-generation build engine, enabled by default since Docker 23. It unlocks parallel layer building, better caching, and secret/SSH mount types.

**Inline cache — speed up CI builds**

```bash
# Build with inline cache metadata embedded in the image
docker build \
  --cache-from myregistry/myapp:cache \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t myregistry/myapp:latest \
  -t myregistry/myapp:cache \
  .

# Push both tags so next run can use the cache
docker push myregistry/myapp:latest
docker push myregistry/myapp:cache
```

In CI, pull the cache tag before building. BuildKit checks each layer's checksum against the pulled image and skips layers that have not changed.

**SSH mounts — access private repositories during build**

```bash
# Add your key to the agent
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

# Pass the agent socket to docker build
docker build --ssh default .
```

```dockerfile
# In the Dockerfile — SSH credentials are NEVER stored in any layer
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git /app/private
```

**Cache mounts — persist package manager caches across builds**

```dockerfile
# Keep pip's download cache between builds (not stored in the image)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Keep apt's package cache between builds
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

This can reduce a `pip install` or `apt-get` step from 60 seconds to under 5 on a warm cache.

---

## 12.2 Docker Buildx and Multi-Platform Images

`docker buildx` is the CLI plugin for BuildKit's extended build capabilities. Its most important feature is building images for multiple CPU architectures from a single machine.

**Why this matters:** Apple Silicon Macs run `linux/arm64`. Most cloud production environments run `linux/amd64`. If your image only contains one architecture's layers, it will either fail to start or run slowly under emulation on the other.

**Setup**

```bash
# List available builders
docker buildx ls

# Create a new builder that supports cross-compilation
docker buildx create --name cross-builder --driver docker-container --use

# Bootstrap it (pulls the BuildKit image)
docker buildx inspect --bootstrap
```

**Building and pushing a multi-platform image**

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t myregistry/myapp:1.0 \
  --push \
  .
```

The `--push` flag is required because a multi-platform manifest list cannot be loaded into the local Docker image store — it must live in a registry.

**Inspect the resulting manifest list**

```bash
docker buildx imagetools inspect myregistry/myapp:1.0
```

Output shows a manifest for each platform with its digest. When a user or orchestrator pulls `myapp:1.0`, the registry automatically serves the correct architecture.

**Load a single-platform build locally (for testing)**

```bash
docker buildx build \
  --platform linux/amd64 \
  -t myapp:test \
  --load \
  .
```

---

## 12.3 OCI Specs and the Container Ecosystem

The **Open Container Initiative (OCI)** defines three specifications that make container tooling interoperable:

| Spec | What it defines |
|------|----------------|
| Image spec | Layer format, image manifest, image configuration |
| Runtime spec | How a container is started from a bundle (implemented by `runc`) |
| Distribution spec | How images are pushed to and pulled from registries |

Because everything speaks OCI, a Docker-built image runs on Kubernetes, Podman, or any other OCI-compliant runtime without conversion.

**The runtime stack on a modern Linux host:**

```
Docker CLI
    │  (API calls)
    ▼
Docker Daemon (dockerd)
    │  (delegates to)
    ▼
containerd              ← also used directly by Kubernetes
    │  (calls)
    ▼
runc                    ← OCI runtime reference implementation (written in Go)
    │  (fork/exec)
    ▼
Container process (your app)
```

**Alternative runtimes:**

| Runtime | Description |
|---------|-------------|
| `runc` | Reference implementation; default in Docker and containerd |
| `crun` | Written in C; faster startup; lower memory footprint |
| `gVisor` (runsc) | Intercepts syscalls in user space; stronger isolation |
| `Kata Containers` | Runs each container in a lightweight VM; VM-level isolation |

Kubernetes uses **containerd** directly (the Docker daemon is no longer required as of Kubernetes 1.24). Docker images work unchanged because they conform to the OCI image spec.

---

## 12.4 Docker Swarm Overview

Docker Swarm is Docker's built-in container orchestration system. It turns a pool of Docker hosts into a single logical cluster.

**Swarm concepts:**

| Concept | Meaning |
|---------|---------|
| Node | A Docker host participating in the Swarm |
| Manager node | Stores cluster state, schedules tasks, exposes the API |
| Worker node | Runs containers; takes instructions from managers |
| Service | Desired state declaration: which image, how many replicas |
| Task | One running container — one instance of a service |
| Stack | A group of services defined in a Compose file |

**Initialising a Swarm**

```bash
# On the first manager node
docker swarm init --advertise-addr 192.168.1.10

# Output includes a join token; run this on worker nodes
docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

# View the cluster
docker node ls
```

**Deploying a stack**

```bash
docker stack deploy -c docker-compose.yml myapp
docker stack ls
docker stack services myapp
docker stack ps myapp
```

**Scaling and updating services**

```bash
# Scale horizontally
docker service scale myapp_api=5

# Rolling update (replaces tasks one at a time)
docker service update --image myapp:2.0 myapp_api

# Control rollout behaviour
docker service update \
  --image myapp:2.0 \
  --update-parallelism 2 \       # update 2 tasks at a time
  --update-delay 10s \           # wait 10s between batches
  --update-failure-action rollback \
  myapp_api

# Rollback to previous version
docker service rollback myapp_api
```

> Swarm is production-capable and requires no additional tooling. It is well-suited for teams with 1–20 nodes. For hundreds of nodes or complex deployment pipelines, Kubernetes is the more common choice.

---

## 12.5 Container Networking Internals

Understanding what Docker actually does to the host's network stack demystifies firewall rules, port conflicts, and multi-host networking.

**The default bridge (`docker0`)**

```bash
# Docker creates a virtual bridge interface on the host
ip addr show docker0
# inet 172.17.0.1/16 — this is the gateway for containers on the default bridge

ip route | grep docker
# 172.17.0.0/16 dev docker0 — traffic to this subnet goes to docker0
```

**veth pairs — the cable between container and bridge**

When Docker starts a container it creates a **veth (virtual Ethernet) pair**: two virtual interfaces that act like the two ends of a network cable. One end stays in the host network namespace (attached to the bridge), the other end is moved into the container's namespace (visible as `eth0`).

```bash
# See veth interfaces on the host
ip link | grep veth

# Inside the container
docker exec mycontainer ip link       # shows eth0
docker exec mycontainer ip route      # default via 172.17.0.1 (the bridge)
docker exec mycontainer ip addr       # shows its IP
```

**Packet flow for outbound traffic:**

```
Container (eth0)
    → veth pair
    → docker0 bridge
    → iptables MASQUERADE rule (NAT: rewrites source IP to host IP)
    → host NIC
    → internet
```

**Inspecting the NAT rules:**

```bash
sudo iptables -t nat -L DOCKER -n --line-numbers
sudo iptables -t nat -L POSTROUTING -n --line-numbers
```

**Port publishing:**

When you `docker run -p 8080:80 nginx`, Docker adds an iptables DNAT rule: traffic arriving on host port 8080 is redirected to the container's port 80.

---

## 12.6 Custom Bridge Networks and Inter-Service Communication

User-defined bridge networks provide automatic DNS resolution (containers resolve each other by name) and network-level isolation.

**Isolating services with multiple networks**

A useful pattern: frontend services can talk to the API, the API can talk to the database, but the frontend cannot reach the database directly.

```bash
docker network create frontend-net
docker network create backend-net

# API container is on BOTH networks (bridge between them)
docker run -d --name api \
  --network backend-net \
  myapi

docker network connect frontend-net api

# DB is only on the backend network
docker run -d --name db \
  --network backend-net \
  postgres

# Web is only on the frontend network
docker run -d --name web \
  --network frontend-net \
  mywebapp
```

**Access matrix:**

| From → To | web → api | web → db | api → db |
|-----------|-----------|----------|----------|
| Reachable? | Yes | No | Yes |

In Compose:

```yaml
services:
  web:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
```

**Inter-container DNS:** on a user-defined network, containers resolve each other by service name (`api`, `db`). On the default bridge they do not — only by IP. This is a major reason to always use user-defined networks.

---

## 12.7 Image Manifest Lists and Digests

**Image tags are mutable.** `nginx:latest` today is not the same image as `nginx:latest` tomorrow. For reproducible builds and deployments, use **digests** — the SHA256 hash of the image manifest. Digests are immutable.

```bash
# Pull by digest — will always pull the exact same image
docker pull nginx@sha256:a5127df1c6b5bf21e8...

# Get the digest of an image you already have
docker inspect --format='{{index .RepoDigests 0}}' nginx

# Or look it up in the registry without pulling
docker buildx imagetools inspect nginx:1.25
```

**In Dockerfile — pin base images by digest for reproducible builds:**

```dockerfile
FROM nginx@sha256:a5127df1c6b5bf21e8bb40d5b6ab7c90b41ef...
```

This is especially important for production images where you want to guarantee the exact base image used regardless of what `latest` or `1.25` resolves to at build time.

---

## 12.8 Docker Context

Docker contexts let you manage multiple Docker endpoints — local daemon, remote server, or Docker Desktop — from a single CLI.

```bash
# List available contexts
docker context ls

# Create a context for a remote server (using SSH)
docker context create remote-prod \
  --docker "host=ssh://deploy@prod.example.com"

# Switch to the remote context
docker context use remote-prod

# Now all docker commands operate on the remote host
docker ps
docker pull myapp:2.0
docker stack deploy -c docker-compose.yml myapp

# Return to local Docker
docker context use default

# Run a single command against a specific context without switching
docker --context remote-prod ps
```

Contexts are stored in `~/.docker/contexts/`. They are useful for:
- Deploying from your laptop to a remote server
- Managing multiple clusters or environments
- CI pipelines that target different registries or daemons

---

## 12.9 Container Image Introspection

**Dive — layer-by-layer image explorer**

```bash
# Install (macOS)
brew install dive

# Inspect an image
dive myapp:1.0
```

Dive shows:
- Every layer with its size and the files it added/modified/deleted
- An efficiency score (wasted space from files overwritten in later layers)
- Which files are new, modified, or removed per layer

Common findings: a `RUN apt-get install` layer that's hundreds of MB, followed by a layer that deletes the apt cache — the cache is hidden but still in the image. Fix: chain the `rm -rf /var/lib/apt/lists/*` in the same `RUN` command.

**Image manifest inspection**

```bash
# Raw manifest (shows layers as digests)
docker manifest inspect nginx

# With buildx (richer output, handles multi-platform manifests)
docker buildx imagetools inspect nginx:1.25
```

**Extracting a layer manually**

```bash
docker save nginx | tar -xf - -C /tmp/nginx-layers
ls /tmp/nginx-layers           # manifest.json, config JSON, layer tarballs
tar -tf /tmp/nginx-layers/<layer-hash>/layer.tar | head -30
```

**Scanning images for embedded secrets**

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  trufflesecurity/trufflehog:latest \
  docker --image myapp:1.0
```

TruffleHog scans every layer in the image for patterns that look like API keys, passwords, private keys, and other credentials.

---

## 12.10 Performance Tuning

**Measuring container startup time**

```bash
time docker run --rm alpine echo "started"
# real 0m0.312s — baseline for a trivial container
```

**Reduce build context size**

The build context is sent to the Docker daemon before building. A large context (accidentally including `node_modules` or `.git`) slows every build.

```
# .dockerignore — start by ignoring everything, then allow what you need
**
!src/
!package.json
!package-lock.json
!tsconfig.json
```

Check context size before building:

```bash
du -sh .     # total directory size
# Compare with what's actually sent after .dockerignore
docker build --no-cache . 2>&1 | head -3
# "Sending build context to Docker daemon  1.234MB"
```

**Pre-pull images in production to reduce rollout latency**

```bash
# Pull new version on all nodes before switching traffic
docker pull myapp:2.0

# Then update the service
docker service update --image myapp:2.0 myapp_api
```

**Enable BuildKit explicitly (pre-23 Docker)**

```bash
DOCKER_BUILDKIT=1 docker build .
```

**Parallel Compose startup**

Compose already starts services in parallel where `depends_on` allows. To further reduce startup time, ensure services that do not depend on each other have no unnecessary `depends_on` entries.

---

## Summary

BuildKit's cache and secret mounts dramatically reduce CI build times while keeping credentials out of image layers. Multi-platform builds with `docker buildx` solve the ARM/x86 gap in heterogeneous environments. OCI specifications are the glue that makes images portable across Docker, containerd, and Kubernetes. Swarm provides a built-in path to orchestration without Kubernetes complexity. Understanding veth pairs, bridges, and iptables NAT rules demystifies Docker networking. Contexts make it straightforward to manage multiple Docker endpoints from one workstation.

---

## Knowledge Check

1. What is the difference between `--cache-from` and a BuildKit cache mount (`--mount=type=cache`)? When would you use each?
2. Why does `docker buildx build --push` require `--push` when building for multiple platforms?
3. Explain what the OCI runtime spec defines and name the reference implementation.
4. In Docker Swarm, what is the difference between a service and a task?
5. If `docker logs` shows nothing and you suspect an iptables issue with port publishing, what command would you run to inspect the NAT rules?

---

## Hands-on Exercise

**Part A — Multi-platform build**

1. Create a new builder: `docker buildx create --name multibuilder --use`
2. Write a simple Go "hello world" that prints its architecture: `runtime.GOARCH`
3. Build for both `linux/amd64` and `linux/arm64` and push to Docker Hub or a local registry
4. Run `docker buildx imagetools inspect` and verify both platform manifests are present

**Part B — Layer efficiency with dive**

1. Install `dive`
2. Inspect `python:3.12` (an official image) and find the largest layer
3. Build an intentionally wasteful image: install a package in one `RUN` layer, then try to delete it in a second `RUN` layer
4. Run `dive` and confirm the deleted files still contribute to image size
5. Fix it by merging both operations into a single `RUN` layer and compare sizes

**Part C — Network isolation**

Set up three services with the following access rules:
- `frontend` can reach `api`
- `api` can reach `db`
- `frontend` cannot reach `db`

Use `docker network create` and manually verify with `docker exec frontend ping db` (should fail) and `docker exec frontend ping api` (should succeed).

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="11-intermediate-concepts.md">← Previous: Intermediate Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="13-best-practices.md">Next: Best Practices →</a>
</div>
