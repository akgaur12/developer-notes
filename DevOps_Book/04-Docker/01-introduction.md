# Chapter 1 — Introduction to Docker

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the problem Docker solves and why containers exist
- Distinguish between a container and a virtual machine
- Identify the key components of the Docker ecosystem
- Install Docker on a Linux system
- Run your first containers using `docker run`
- Understand what flags like `-it`, `-d`, and `-p` do

---

## 1.1 The Problem Docker Solves

### "It Works on My Machine"

You've finished building a feature. It runs perfectly on your laptop. You push to staging — it crashes. The classic exchange:

> **Developer:** "It works on my machine."
> **Ops engineer:** "Then ship your machine."

Docker lets you ship (a snapshot of) your machine.

### Why Environments Diverge

Several factors cause the dev→staging→production inconsistency problem:

- **Operating system differences** — Ubuntu 20.04 on your laptop vs CentOS 7 in production
- **Library versions** — `libssl 1.1` vs `libssl 3.0`; `python 3.9` vs `python 3.11`
- **Environment variables** — `DATABASE_URL` set locally but not in CI
- **System packages** — `imagemagick` installed manually on dev, never documented
- **Runtime configuration** — `ulimit` settings, timezone, locale

### Before Docker: The Old Way

Teams dealt with this through:

- **Virtual machines** — heavyweight, slow to spin up, expensive, hard to reproduce exactly
- **Manual dependency docs** — README files that grew stale; "run `apt install X Y Z` before starting"
- **Snowflake servers** — every server configured slightly differently by hand; no one knows exactly what's on each one
- **Configuration management tools** (Ansible, Chef, Puppet) — better, but still complex and drift-prone

### The Docker Solution

Docker packages the **application AND its environment** together into a single image. That image runs identically on:

- Your developer laptop (Linux, Mac, or Windows)
- A colleague's machine
- A CI/CD runner
- A staging server
- A Kubernetes cluster in production

Same image. Same behavior. Everywhere.

---

## 1.2 What Is a Container?

### The Shipping Container Analogy

Before the 1960s, cargo ships were loaded piece by piece — each item required custom handling, special tools, and skilled longshoremen. The invention of the standardized ISO shipping container transformed global trade: it doesn't matter whether the container holds bananas, electronics, or machinery. The crane, ship, and truck all handle it the same way.

Docker containers are the same idea for software: a **standardized, portable, isolated unit** that any container runtime can run, regardless of what's inside.

### Container vs Virtual Machine

| Aspect | Virtual Machine | Container |
|--------|----------------|-----------|
| **Boot time** | 1–5 minutes | Milliseconds |
| **Size** | GBs (includes full OS) | MBs (shares host OS kernel) |
| **Isolation** | Strong (separate kernel) | Good (namespaces + cgroups) |
| **OS** | Each VM has its own | All containers share host kernel |
| **Performance** | Near-native but overhead | Near-native |
| **Use case** | Run different OS, strong security boundary | Microservices, CI/CD, packaging apps |
| **Resource usage** | High | Low — run dozens on one machine |

### Containers Share the Host Kernel

VMs run a **full guest OS** on top of a hypervisor. Containers use **Linux kernel features** (namespaces and cgroups) to isolate processes — they still use the host's kernel, just in an isolated view of it.

```
┌─────────────────────────────────────────────────────────────┐
│                        HOST MACHINE                         │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Host OS / Kernel                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                             │                               │
│              ┌──────────────┼──────────────┐               │
│              ▼              ▼              ▼               │
│     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│     │ Container A │ │ Container B │ │ Container C │       │
│     │  (nginx)    │ │  (postgres) │ │  (node app) │       │
│     └─────────────┘ └─────────────┘ └─────────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

All three containers share the same kernel but each sees its own isolated filesystem, network stack, and process tree.

---

## 1.3 What Is Docker?

Docker refers to several related but distinct things:

### Docker Inc.
The company (now merged into Mirantis for Docker Enterprise; Docker Desktop and Hub remain under Docker Inc.) that created and popularized container tooling.

### Docker Engine
The core server-side component that creates and runs containers. It consists of:
- **dockerd** — the long-running daemon process
- **containerd** — manages container lifecycle
- **runc** — low-level OCI container runtime

### Docker CLI
The `docker` command you type in your terminal. It sends commands to the Docker daemon via REST API.

### Docker Hub
The default public registry at `hub.docker.com`. Contains official images for nginx, postgres, python, ubuntu, and hundreds of thousands of community images. Free to use for pulling; rate limits apply for anonymous pulls.

### Docker Desktop
A GUI application for Mac and Windows that bundles Docker Engine (running in a lightweight VM), Docker CLI, Docker Compose, and a dashboard. Not required on Linux.

### Docker Compose
A tool for defining and running **multi-container applications** using a `docker-compose.yml` file. Covered in depth in Chapter 8.

### The Full Docker Ecosystem

```
docker CLI          ← You type commands here
docker Desktop      ← GUI wrapper (Mac/Windows)
Docker Engine       ← Runs containers on your machine
Docker Hub          ← Public image registry
Docker Compose      ← Multi-container orchestration
Docker Swarm        ← Built-in clustering (mostly superseded by Kubernetes)
```

---

## 1.4 Core Concepts (Overview)

Before diving deeper, here's a quick mental model of the key terms:

| Concept | What It Is | Analogy |
|---------|-----------|---------|
| **Image** | Read-only blueprint containing your app and its dependencies | A recipe |
| **Container** | A running (or stopped) instance of an image | A dish made from that recipe |
| **Dockerfile** | A text file with instructions for building an image | The written-down recipe |
| **Registry** | A server that stores and distributes images | A cookbook library |
| **Volume** | Persistent storage that outlives a container | An external hard drive attached to the container |
| **Network** | Virtual network connecting containers | An internal office network |

**Key insight:** an image is immutable (like a class definition); a container is a running instance (like an object). You can run many containers from the same image, and stopping/removing a container never changes the image.

---

## 1.5 Installing Docker

### Ubuntu / Debian

```bash
# 1. Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 2. Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. Add the repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 5. Add your user to the docker group (avoids typing sudo every time)
sudo usermod -aG docker $USER
newgrp docker

# 6. Verify the installation
docker version
docker run hello-world
```

### Post-install Check

After running `docker run hello-world` you should see:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### Other Platforms

- **Mac/Windows:** Download Docker Desktop from `https://docker.com/products/docker-desktop`
- **RHEL/Fedora:** Use `dnf` instead of `apt`; the GPG key URL is slightly different (use `download.docker.com/linux/fedora`)

---

## 1.6 Your First Container

Let's run a few containers to build intuition.

```bash
# 1. The classic smoke test
docker run hello-world
# Docker pulls the image, runs it, prints a message, container exits

# 2. An interactive shell inside Ubuntu
docker run -it ubuntu bash
# -i = interactive (keep STDIN open)
# -t = allocate a pseudo-TTY (terminal)
# You are now root inside an Ubuntu container!
# Type: ls /  cat /etc/os-release  exit

# 3. Run nginx in the background with port mapping
docker run -d -p 8080:80 nginx
# -d = detached (background)
# -p 8080:80 = map host port 8080 to container port 80

# Test it
curl http://localhost:8080
# You should see the nginx welcome page HTML

# 4. See running containers
docker ps

# 5. See ALL containers (including stopped)
docker ps -a

# 6. Stop the nginx container (use the ID or name from docker ps)
docker stop <container_id_or_name>

# 7. Remove it
docker rm <container_id_or_name>
```

### Flag Reference

| Flag | Meaning |
|------|---------|
| `-i` | Keep STDIN open (interactive) |
| `-t` | Allocate a TTY (terminal emulation) |
| `-it` | Use both — gives you a usable shell |
| `-d` | Detached — run in background, print container ID |
| `-p host:container` | Map a host port to a container port |
| `--rm` | Automatically remove the container when it stops |
| `--name` | Give the container a human-readable name |

### Port Mapping Explained

`-p 8080:80` means: "take traffic arriving at port **8080** on the host and forward it to port **80** inside the container."

The format is always `HOST_PORT:CONTAINER_PORT`.

---

## 1.7 Docker vs Podman vs containerd

Docker is not the only container tool. Here's a quick orientation:

| Tool | Description | Key Difference |
|------|-------------|----------------|
| **Docker** | The original, most popular container tool | Requires a daemon (`dockerd`) running as root |
| **Podman** | Drop-in Docker replacement from Red Hat | Daemonless and **rootless** by default; more secure |
| **containerd** | Low-level container runtime | What Kubernetes actually uses under the hood; no CLI for humans |
| **nerdctl** | Docker-compatible CLI for containerd | Used when you want Docker-like UX with containerd directly |

For learning purposes, Docker and Podman are nearly identical at the CLI level. If you learn Docker, you can use Podman with no retraining (the commands are the same). Kubernetes uses containerd directly — Docker knowledge transfers because the image format (OCI) is the same.

---

## Summary

- Docker solves the environment consistency problem by packaging applications with their dependencies
- Containers are lightweight isolated processes sharing the host OS kernel; VMs run a full guest OS
- Key Docker components: Engine (dockerd), CLI, Hub (registry), Compose, Desktop
- Core concepts: image (blueprint) → container (running instance), Dockerfile (recipe), registry (storage)
- Installed Docker and ran `hello-world`, an interactive Ubuntu shell, and nginx with port mapping
- Flags to know: `-it` for interactive, `-d` for background, `-p` for ports, `--rm` for auto-cleanup

---

## Knowledge Check

1. What is the main problem Docker was designed to solve?
2. How does a container differ from a virtual machine in terms of OS usage?
3. What is the difference between a Docker image and a Docker container?
4. What does the `-p 8080:80` flag mean in `docker run -p 8080:80 nginx`?
5. You want to run a container, poke around inside it interactively, and have it automatically deleted when you exit. What flags do you use?

---

## Hands-On Exercise

Complete all of the following steps:

1. **hello-world** — Run `docker run hello-world`. Read the output carefully.
2. **nginx** — Run nginx in detached mode with port mapping (`-d -p 8080:80`). Visit `http://localhost:8080` in your browser or with `curl`. Check `docker ps` to see it running. Stop it with `docker stop`.
3. **alpine** — Run `docker run -it --rm alpine sh`. Alpine Linux is a minimal 5 MB container. Inside, run `ls`, `cat /etc/os-release`, `apk add curl`, `curl https://example.com`. Type `exit`.
4. **Exploration** — For any running container, try:
   - `docker logs <container>` — view stdout/stderr output
   - `docker inspect <container>` — full JSON metadata
   - `docker stats` — live CPU and memory usage

**Goal:** By the end you should feel comfortable starting, stopping, and removing containers.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-architecture-and-internals.md">Next: Docker Architecture & Internals →</a>
</div>
