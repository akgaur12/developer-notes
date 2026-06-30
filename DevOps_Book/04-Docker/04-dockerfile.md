# Chapter 4 — Writing Dockerfiles

> **Docker Course — Chapter 4 of 17**

## Learning Objectives

By the end of this chapter you will be able to:

- Understand what a Dockerfile is and how Docker processes it
- Use every major Dockerfile instruction correctly
- Write production-ready Dockerfiles for Node.js and Python applications
- Understand the difference between `ENTRYPOINT` and `CMD`
- Optimize builds using layer caching
- Create `.dockerignore` files to speed up builds and reduce image size
- Build and tag images from the command line

---

## 4.1 What Is a Dockerfile?

A **Dockerfile** is a plain text file (no extension) that contains a sequence of instructions telling Docker how to assemble an image. Think of it as a recipe: Docker reads it top to bottom and executes each instruction in order.

Key properties:

- Each instruction creates a new **read-only layer** in the image
- Layers are cached — if nothing has changed, Docker reuses the cached layer
- The order of instructions matters enormously for build performance
- The file is conventionally named `Dockerfile` (capital D, no extension)

When you run `docker build`, Docker:

1. Reads the Dockerfile
2. Executes each instruction in sequence
3. Produces a final image made of stacked layers
4. Tags the image with the name you provide

---

## 4.2 Core Instructions

Every Dockerfile instruction follows the pattern `INSTRUCTION arguments`. Here is every instruction you will encounter regularly, with a real example and a common mistake for each.

### FROM — Base Image

```dockerfile
FROM node:20-alpine
```

Always the **first** instruction (comments and `ARG` can precede it). Defines the starting point for your image. Choose the smallest base that meets your needs.

- `node:20` — full Debian-based Node image (~1 GB)
- `node:20-alpine` — Alpine-based, much smaller (~150 MB)
- `scratch` — completely empty, used for statically compiled binaries

**Common mistake:** Using `:latest` tag. It makes builds non-reproducible. Always pin a specific version.

```dockerfile
# Bad
FROM node:latest

# Good
FROM node:20-alpine
```

---

### RUN — Execute a Command During Build

```dockerfile
RUN npm ci --only=production
```

Creates a new layer with the result of the command. Used for installing packages, compiling code, creating directories, etc.

**Common mistake:** Running multiple separate `RUN` statements when they can be chained. Each `RUN` creates a layer.

```dockerfile
# Bad — 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good — 1 layer
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

---

### COPY — Copy Files from Build Context

```dockerfile
COPY package*.json ./
COPY src/ ./src/
```

Copies files or directories from your **build context** (the directory you pass to `docker build`) into the image. Prefer `COPY` over `ADD` for simple file copying.

**Common mistake:** Copying everything before installing dependencies (breaks cache — see section 4.6).

---

### ADD — Like COPY with Extras

```dockerfile
ADD archive.tar.gz /app/
ADD https://example.com/config.yaml /app/config.yaml
```

`ADD` can:
- Extract tar archives automatically
- Fetch files from URLs

**Best practice:** Use `COPY` unless you specifically need tar extraction or URL fetching. `ADD`'s implicit behaviour makes Dockerfiles harder to reason about.

---

### WORKDIR — Set Working Directory

```dockerfile
WORKDIR /app
```

Sets the working directory for all subsequent `RUN`, `COPY`, `ADD`, `CMD`, and `ENTRYPOINT` instructions. Creates the directory if it does not exist.

**Common mistake:** Using `RUN cd /app` instead — it does not persist across instructions.

```dockerfile
# Bad — cd has no effect on next RUN
RUN cd /app
RUN npm install     # still runs in /

# Good
WORKDIR /app
RUN npm install     # runs in /app
```

---

### ENV — Set Environment Variables

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

Sets environment variables that persist into the running container. Accessible via `process.env.NODE_ENV` in Node.js, `os.environ["NODE_ENV"]` in Python, etc.

**Common mistake:** Using `ENV` for secrets. Environment variables can be inspected with `docker inspect`. Use secrets management instead (covered in the security chapter).

---

### ARG — Build-time Variables

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
```

Defines variables that can be passed at build time with `--build-arg`. Unlike `ENV`, they are **not available** in the running container.

```bash
docker build --build-arg NODE_VERSION=18 -t myapp .
```

**Common mistake:** Using `ARG` for secrets — ARG values appear in `docker history`. Use `--mount=type=secret` (BuildKit) instead.

---

### EXPOSE — Document Port

```dockerfile
EXPOSE 3000
EXPOSE 3000/tcp
EXPOSE 5353/udp
```

Documents which port(s) the container listens on. It does **not** publish or open ports — that is done with `-p` in `docker run`. Think of it as metadata for developers and tools.

---

### CMD — Default Command

```dockerfile
CMD ["node", "server.js"]
CMD ["nginx", "-g", "daemon off;"]
```

Specifies the default command to run when the container starts. Can be overridden by passing a command to `docker run`.

```bash
# Uses CMD from Dockerfile
docker run myapp

# Overrides CMD
docker run myapp node debug.js
```

---

### ENTRYPOINT — Fixed Executable

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

Like `CMD` but the command is **not** overridable with `docker run <image> <args>`. Instead, anything passed to `docker run` becomes arguments to the entrypoint.

See section 4.5 for the full `ENTRYPOINT` vs `CMD` breakdown.

---

### USER — Switch to Non-root User

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

Switches the user for subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions. Containers run as root by default — always switch to a non-root user before the final `CMD` for production images.

---

### VOLUME — Declare Mount Point

```dockerfile
VOLUME ["/var/lib/postgresql/data"]
VOLUME /uploads
```

Declares a directory as a mount point for persistent data. When a container starts without an explicit volume mount, Docker creates an anonymous volume for this path. Best practice is to manage volumes explicitly with `-v` rather than relying on this declaration.

---

### HEALTHCHECK — Container Health

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

Defines a command Docker runs periodically to check if the container is healthy. Exit code 0 = healthy, 1 = unhealthy. Docker Compose and Kubernetes use this to decide whether to route traffic to the container.

Options:
- `--interval` — how often to run (default: 30s)
- `--timeout` — max time for the check to succeed (default: 30s)
- `--start-period` — grace period before failures count (default: 0s)
- `--retries` — consecutive failures before marking unhealthy (default: 3)

---

### LABEL — Add Metadata

```dockerfile
LABEL maintainer="team@example.com"
LABEL version="1.0.0"
LABEL description="My production app"
```

Adds key-value metadata to the image. Accessible via `docker inspect`. Useful for CI/CD pipelines, image registries, and automated tooling.

---

## 4.3 A Real-World Dockerfile: Node.js App

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files first (layer caching)
COPY package*.json ./
RUN npm ci --only=production

# Copy app source
COPY . .

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

**Line-by-line walkthrough:**

| Line | Why |
|------|-----|
| `FROM node:20-alpine` | Alpine base = smaller image (~150 MB vs ~1 GB for full Debian) |
| `WORKDIR /app` | All subsequent commands run in `/app`; no `cd` needed |
| `COPY package*.json ./` | Copy dependency manifests first (before app source) to maximise cache hits |
| `RUN npm ci --only=production` | `npm ci` is reproducible (uses package-lock.json); `--only=production` skips dev dependencies |
| `COPY . .` | Copy app source after dependencies — changes here don't invalidate the `npm ci` cache |
| `addgroup / adduser / USER` | Never run Node.js as root in production |
| `EXPOSE 3000` | Documents port; does not publish it |
| `HEALTHCHECK` | Docker and orchestrators can detect unhealthy containers and restart them |
| `CMD ["node", "server.js"]` | Exec form (array) is preferred — no shell wrapper, signals go directly to the process |

---

## 4.4 A Real-World Dockerfile: Python App

```dockerfile
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Notable choices:**

- `python:3.11-slim` — Debian-slim base; smaller than full Debian, more compatible than Alpine for Python (Alpine can cause issues with compiled packages)
- `PYTHONDONTWRITEBYTECODE=1` — prevents `.pyc` files from being written (cleaner image)
- `PYTHONUNBUFFERED=1` — ensures stdout/stderr is flushed immediately (important for log aggregation)
- `--no-cache-dir` on pip — avoids storing the pip download cache inside the image
- `--host 0.0.0.0` — binds to all interfaces, required for the port to be reachable from outside the container

---

## 4.5 ENTRYPOINT vs CMD

This is one of the most confusing aspects of Dockerfiles. The key insight: **ENTRYPOINT sets what runs, CMD sets the default arguments**.

| | ENTRYPOINT | CMD |
|---|---|---|
| Purpose | Fixed executable | Default arguments / default command |
| Override method | `docker run --entrypoint <cmd>` | `docker run <image> <new-command>` |
| When both are set | CMD becomes arguments passed to ENTRYPOINT | — |
| Typical use | Wrapper scripts, always-the-same binary | Default flags that users may want to change |

### Pattern 1: CMD only (most flexible)

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Users can completely replace the command:

```bash
docker run myimage nginx -g "daemon off;" -c /custom/nginx.conf
```

### Pattern 2: ENTRYPOINT only

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

Users can pass extra arguments but cannot change `python app.py`:

```bash
docker run myimage --port 8080
# runs: python app.py --port 8080
```

### Pattern 3: ENTRYPOINT + CMD (shell/wrapper scripts)

```dockerfile
ENTRYPOINT ["./entrypoint.sh"]
CMD ["server"]
```

The entrypoint script receives `server` as `$1`. Users can override just the argument:

```bash
docker run myimage migrate
# runs: ./entrypoint.sh migrate
```

### Shell form vs exec form

```dockerfile
# Shell form — Docker wraps with /bin/sh -c
CMD node server.js

# Exec form — runs directly, NO shell (PREFERRED)
CMD ["node", "server.js"]
```

The exec form is strongly preferred because:

1. Signals (SIGTERM for graceful shutdown) go directly to your process, not to a shell
2. No shell expansion surprises
3. Slightly faster startup (no shell process)

---

## 4.6 Layer Caching — The Critical Optimization

Docker checks each instruction against its build cache. If the instruction and all its inputs are identical to a previous build, Docker reuses the cached layer and skips re-executing it. This can turn a 3-minute build into a 5-second one.

**The golden rule: put things that change less at the top, things that change more at the bottom.**

### Bad example — cache-busting on every build

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .              # changes every time any file changes
RUN npm install       # re-runs on EVERY build, even if dependencies didn't change
```

Every time you edit a single line of app code, `COPY . .` is invalidated, and `npm install` runs again from scratch.

### Good example — dependency layer isolated

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./     # only changes when package.json or package-lock.json changes
RUN npm install           # cached unless package files changed
COPY . .                  # app code changes often — placed AFTER install
```

Now editing app code only invalidates `COPY . .` onwards. The `npm install` layer is reused, saving minutes per build.

### Cache invalidation rules

- `FROM` — invalidated if the base image digest changes
- `RUN` — invalidated if the instruction string changes, or if a previous layer changed
- `COPY` / `ADD` — Docker checksums the files; invalidated if any copied file changes

---

## 4.7 .dockerignore

`.dockerignore` works like `.gitignore` but for the Docker build context. Before Docker sends your files to the daemon for building, it filters out everything matching `.dockerignore` patterns.

**Typical `.dockerignore` for a Node.js project:**

```
node_modules/
.git/
.env
.env.*
*.log
.DS_Store
dist/
coverage/
.nyc_output/
*.test.js
*.spec.js
README.md
```

**Typical `.dockerignore` for a Python project:**

```
__pycache__/
*.pyc
*.pyo
.git/
.env
.venv/
venv/
*.log
.DS_Store
.pytest_cache/
.mypy_cache/
dist/
*.egg-info/
```

**Why it matters:**

- Without `.dockerignore`, `node_modules/` (potentially hundreds of MB) gets sent to the Docker daemon on every build, even if you are about to `COPY` it and overwrite it with `RUN npm install`
- `.git/` can be large and changes on every commit, busting your caches
- `.env` files contain secrets that must never enter the image

**Rule of thumb:** Always create `.dockerignore` before your first `docker build`.

---

## 4.8 Build and Tag

```bash
# Basic build — tags image as myapp:1.0, uses Dockerfile in current directory
docker build -t myapp:1.0 .

# Specify a different Dockerfile
docker build -t myapp:1.0 -f Dockerfile.prod .

# Force rebuild without cache
docker build --no-cache -t myapp:1.0 .

# Pass a build argument
docker build --build-arg NODE_ENV=production -t myapp:1.0 .

# Multiple tags in one build
docker build -t myapp:1.0 -t myapp:latest .

# View resulting image size
docker images myapp

# View all layers and their sizes
docker history myapp:1.0
```

**Tag naming convention:** `registry/repository:tag`

```
myapp:1.0                        # local image
docker.io/myuser/myapp:1.0       # Docker Hub
ghcr.io/myorg/myapp:1.0          # GitHub Container Registry
123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0  # AWS ECR
```

---

## Summary

- A Dockerfile is a recipe of sequential instructions; each creates an immutable layer
- `FROM` sets the base image — always pin a specific version, prefer slim/alpine variants
- `RUN`, `COPY`, `WORKDIR`, `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `USER`, `HEALTHCHECK` are the most-used instructions
- Exec form (`["command", "arg"]`) is always preferred over shell form for `CMD` and `ENTRYPOINT`
- `ENTRYPOINT` sets the fixed executable; `CMD` provides default arguments (or the default command when used alone)
- Layer caching is critical: copy dependency files before source code, install before copying the rest
- Always create `.dockerignore` to exclude `node_modules`, `.git`, `.env`, and other noise from the build context
- Use `docker build -t name:tag .` to build; `docker history` to inspect layer sizes

---

## Knowledge Check

1. What is the difference between `COPY` and `ADD`? When should you prefer each?
2. You have a Node.js app. In what order should you write `COPY . .`, `RUN npm install`, and `COPY package*.json ./` to maximise layer caching?
3. What happens when you specify both `ENTRYPOINT ["python"]` and `CMD ["app.py"]`?
4. Why is exec form (`CMD ["node", "server.js"]`) preferred over shell form (`CMD node server.js`)?
5. A teammate runs `docker build -t myapp:latest .` and notices it is very slow even on the second build. What is the most likely cause, and how would you fix it?

---

## Hands-on Exercise

**Goal:** Write a Dockerfile from scratch, build it, run it, and observe caching behaviour.

**Part 1 — Write and Build**

1. Create a minimal Node.js (Express) or Python (FastAPI) app with a `/health` endpoint
2. Write a production-quality Dockerfile using the patterns from this chapter
3. Create a `.dockerignore` file
4. Build the image: `docker build -t myapp:1.0 .`
5. Run it: `docker run -d -p 3000:3000 --name myapp myapp:1.0`
6. Verify: `curl http://localhost:3000/health`
7. Check the health status: `docker inspect myapp | grep -A 5 Health`

**Part 2 — Experiment with Caching**

1. Edit only your app's source code (e.g., change a log message), then rebuild. Observe which steps are cached.
2. Edit `package.json` (add a dependency), then rebuild. Observe that `npm install` runs again.
3. Move `COPY . .` before `RUN npm install` (break the cache), rebuild, and compare times.

**Part 3 — Inspect**

1. Run `docker history myapp:1.0` and identify the largest layers
2. Try `docker build --no-cache -t myapp:1.0 .` and time the difference

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="03-images-and-containers.md">← Previous: Images & Containers</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="05-multi-stage-builds.md">Next: Multi-stage Builds →</a>
</div>
