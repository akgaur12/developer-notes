# Chapter 10 — Docker Security

## Learning Objectives

By the end of this chapter you will be able to:
- Explain the Docker threat model and why containers are not fully isolated VMs
- Run containers as non-root users and enforce read-only filesystems
- Drop Linux capabilities and apply seccomp profiles
- Manage secrets safely, never baking them into images or environment variables
- Scan images for vulnerabilities and use minimal base images

---

## 10.1 The Docker Security Model

Docker containers share the host kernel — they are not fully isolated like virtual machines. A VM has its own kernel; a container borrows the host's. This makes containers lightweight, but it also means a compromised container is closer to the host than a compromised VM.

**The threat model**

| Threat | Description |
|--------|-------------|
| Container escape | Exploiting a kernel or runtime vulnerability to break out of isolation |
| Privilege escalation | A root process inside a container gaining root on the host |
| Image vulnerabilities | CVEs in base images or application dependencies |
| Secrets leakage | Passwords and tokens baked into images or visible in environment variables |
| Resource abuse | A container consuming all CPU/memory, causing a denial of service for other workloads |

Security is **defence-in-depth** — multiple overlapping layers, not one magic fix.

**Security hierarchy**

```
Host OS
  └─ Linux Kernel (namespaces, cgroups, seccomp, capabilities)
       └─ Container Runtime (containerd / runc)
            └─ Container (filesystem, network, PID namespace)
                 └─ Application (your code)
```

Each layer adds a control. If one fails, the others still limit the blast radius.

---

## 10.2 Running as Non-Root

By default, processes inside a Docker container run as **root (UID 0)**. If that container is ever escaped, the attacker has root on the host.

```dockerfile
# BAD: running as root (default)
FROM node:20
COPY . .
CMD ["node", "server.js"]

# GOOD: create and use a non-root user
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

Check which user a running container is using:

```bash
docker inspect --format '{{.Config.User}}' my-container
```

Many official images already include a non-root user:
- `node` image → user `node`
- `nginx` image → user `nginx`
- `postgres` image → user `postgres`

Use them:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node . .
USER node
CMD ["node", "server.js"]
```

---

## 10.3 Read-Only Filesystems

Mounting the container filesystem as read-only prevents an attacker from writing malware, dropping scripts, or modifying application files after the container starts.

```bash
# Run container with read-only root filesystem
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  nginx
```

Applications that need to write use `tmpfs` (ephemeral, in-memory) or named volumes (persistent):

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp
      - /run
    volumes:
      - app-data:/var/lib/myapp   # persistent writes go here

volumes:
  app-data:
```

**Why this helps:** even if an attacker exploits the application, they cannot persist a backdoor inside the container.

---

## 10.4 Dropping Linux Capabilities

Linux capabilities break up the monolithic `root` privilege into ~40 distinct permissions. By default, Docker grants containers about 14 of them. Most applications need fewer.

```bash
# Drop all capabilities, then add back only what is needed
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx
```

```yaml
# docker-compose.yml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

**Common capabilities and their meanings:**

| Capability | What it allows | Needed? |
|------------|---------------|---------|
| `NET_BIND_SERVICE` | Bind ports below 1024 | Only for web servers using port 80/443 |
| `CHOWN` | Change file ownership | Only if app manages file ownership |
| `SETUID` / `SETGID` | Change process UID/GID | Avoid if possible |
| `SYS_ADMIN` | Wide range of admin ops | Almost never |
| `NET_ADMIN` | Configure network interfaces | Almost never |
| `SYS_PTRACE` | Attach to processes with ptrace | Debuggers only |

The goal: `--cap-drop ALL` then add back the minimum your app actually requires.

---

## 10.5 Seccomp Profiles

Seccomp (secure computing mode) filters which Linux **system calls** a container may make. The Docker default profile blocks about 44 of the 300+ available syscalls, covering the most dangerous ones (`keyctl`, `ptrace`, `reboot`, etc.).

```bash
# Use the default seccomp profile (already applied automatically)
docker run nginx

# Apply a custom profile
docker run --security-opt seccomp=./my-seccomp.json myapp

# Disable seccomp entirely — strongly NOT recommended
docker run --security-opt seccomp=unconfined myapp
```

Custom profiles are JSON files listing allowed syscalls. The Docker documentation provides the full default profile as a starting point for tightening further.

---

## 10.6 AppArmor and SELinux

These are **mandatory access control (MAC)** systems that enforce policy beyond what DAC (file permissions) can express.

- **AppArmor** — used on Ubuntu/Debian; Docker applies the `docker-default` profile automatically
- **SELinux** — used on RHEL/CentOS; Docker applies the `svirt_sandbox_file_t` label automatically

```bash
# Explicitly specify AppArmor profile
docker run --security-opt apparmor=docker-default nginx

# Load a custom AppArmor profile
sudo apparmor_parser -r -W ./my-docker-profile
docker run --security-opt apparmor=my-docker-profile nginx

# Disable (risky — only for debugging)
docker run --security-opt apparmor=unconfined nginx
```

On most modern Docker installations these are active without any configuration. Verify with:

```bash
docker info | grep -i security
```

---

## 10.7 Secrets Management

Secrets are one of the most commonly mishandled aspects of Docker. The rule: **a secret must never be visible in an image layer, an environment variable listing, or a process table.**

```bash
# NEVER: environment variable (visible in docker inspect and ps aux)
docker run -e DATABASE_PASSWORD=mysecret myapp

# BETTER: .env file (still stored on disk, but not in the image)
docker run --env-file .env myapp

# GOOD: Docker Swarm secrets (for Swarm deployments)
echo "mysecret" | docker secret create db_password -
docker service create --secret db_password myapp
# Secret is available inside the container at /run/secrets/db_password
# It never appears in environment variables or image layers

# PRODUCTION: external secret stores (Vault, AWS Secrets Manager, etc.)
# The application fetches its own secrets at startup using its instance role
```

**In Dockerfiles — never bake secrets into image layers:**

```dockerfile
# BAD: TOKEN is baked into the layer and visible forever in docker history
RUN curl -H "Authorization: Bearer $TOKEN" https://internal-api/config

# GOOD: BuildKit secret mounts — secret is never stored in any layer
RUN --mount=type=secret,id=token \
    curl -H "Authorization: Bearer $(cat /run/secrets/token)" https://internal-api/config
```

Build with the secret:

```bash
docker build --secret id=token,src=./token.txt .
```

**In Compose, reference secrets from environment, not hardcoded values:**

```yaml
services:
  app:
    environment:
      DATABASE_URL: ${DATABASE_URL}   # read from shell or .env file at deploy time
```

---

## 10.8 Image Security

**Scanning for vulnerabilities**

```bash
# Docker Scout (built into Docker Desktop and Docker Hub)
docker scout cves myapp:1.0

# Trivy (open source, widely used in CI)
trivy image myapp:1.0

# Fail CI if high or critical CVEs are found
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:1.0
```

**Using minimal base images**

| Base image | Size | Shell | Package manager | CVE surface |
|------------|------|-------|-----------------|-------------|
| `ubuntu:22.04` | ~70 MB | bash | apt | High |
| `debian:bookworm-slim` | ~75 MB | bash | apt | Medium |
| `alpine:3.19` | ~7 MB | sh | apk | Low |
| `gcr.io/distroless/nodejs:20` | ~30 MB | none | none | Very low |
| `scratch` | 0 MB | none | none | Minimal |

```dockerfile
# Distroless: no shell, no package manager — minimal attack surface
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["dist/server.js"]
```

Distroless images make `docker exec` difficult (no shell), but that is a feature in production — attackers cannot get an interactive shell even if they compromise the process.

---

## 10.9 The Docker Socket and Privilege Escalation

The Docker socket (`/var/run/docker.sock`) is the API endpoint of the Docker daemon. Any process with access to it can issue arbitrary Docker commands — including launching a privileged container that mounts the host filesystem.

```bash
# NEVER do this in production
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
# This is equivalent to giving the container root on the host
```

**Legitimate uses** (CI runners, monitoring agents like Portainer) exist, but require careful consideration:

- Use **Docker-in-Docker (dind)** for CI builds instead of socket mounting
- Use **rootless Docker** to reduce the privilege level of the daemon itself
- Apply **socket proxy** (e.g., `tecnativa/docker-socket-proxy`) to expose only the API endpoints actually needed

---

## 10.10 Rootless Docker

Rootless Docker runs the entire Docker daemon as a non-root user. Container root no longer maps to host root.

```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Set the socket path
export DOCKER_HOST=unix:///run/user/1000/docker.sock

# Verify
docker run hello-world
docker info | grep rootless
```

Benefits:
- A container escape gives the attacker only the privileges of your regular user account
- No SUID binaries required
- Works without modifying system-wide Docker configuration

Limitations: some features (e.g., binding privileged ports) require additional setup.

---

## 10.11 Security Checklist

```
□ Base image      Use minimal (alpine, distroless, slim variants)
□ USER            Always run as non-root; never leave USER unset
□ Read-only       --read-only with tmpfs for writable paths
□ Capabilities    --cap-drop ALL, add only what is needed
□ Secrets         Never in ENV vars, never in image layers
□ Scanning        trivy or docker scout in every CI pipeline run
□ Signing         Use cosign to sign images before pushing to registry
□ Socket          Never mount /var/run/docker.sock in production containers
□ Network         Use user-defined networks; disable inter-container comms on default bridge
□ Updates         Rebuild images regularly to pick up patched base images
```

---

## Summary

Docker containers share the host kernel, so security requires defence in depth rather than relying on any single control. The most impactful steps are: run as a non-root user, use minimal base images, drop unnecessary capabilities, never put secrets in environment variables or image layers, and scan images for CVEs in CI. Rootless Docker and read-only filesystems provide additional hardening with low operational cost.

---

## Knowledge Check

1. Why is running a container as root dangerous even though the container is "isolated"?
2. What is the difference between `--cap-drop ALL --cap-add NET_BIND_SERVICE` and just running without any capability flags?
3. Name two ways a secret can unintentionally leak when using Docker, and how to prevent each.
4. What does mounting `/var/run/docker.sock` into a container effectively grant that container?
5. What is the key advantage of a distroless base image from a security perspective?

---

## Hands-on Exercise

**Part A — Harden a Dockerfile**

Start with this insecure Dockerfile:

```dockerfile
FROM ubuntu:22.04
COPY . /app
WORKDIR /app
RUN apt-get update && apt-get install -y nodejs npm
RUN npm install
CMD ["node", "server.js"]
```

Convert it to:
- Use `node:20-alpine` as the base
- Run as a non-root user
- Support `--read-only` at runtime (identify which paths need `tmpfs`)
- Drop all capabilities (verify the app still starts)

**Part B — Vulnerability Scanning**

1. Install Trivy: `brew install trivy` (macOS) or follow the Linux install guide
2. Scan the original `ubuntu:22.04`-based image: `trivy image myapp:insecure`
3. Scan your hardened alpine-based image: `trivy image myapp:hardened`
4. Compare the CVE counts and severities
5. Add `trivy image --exit-code 1 --severity CRITICAL myapp:hardened` to a `Makefile` target called `security-scan`

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="09-registry-and-image-management.md">← Previous: Registry & Image Management</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="11-intermediate-concepts.md">Next: Intermediate Concepts →</a>
</div>
