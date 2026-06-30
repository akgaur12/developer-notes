# Chapter 5 — Multi-stage Builds & BuildKit

> **Docker Course — Chapter 5 of 17**

## Learning Objectives

By the end of this chapter you will be able to:

- Explain the build vs runtime problem and why it matters
- Write multi-stage Dockerfiles that produce minimal production images
- Use multi-stage builds for Node.js, Go, and Python applications
- Target specific build stages for CI testing
- Enable and use BuildKit features: cache mounts, secret mounts, and SSH forwarding
- Quantify image size reductions and explain their security benefits

---

## 5.1 The Build vs Runtime Problem

To build an application you typically need compilers, build frameworks, test runners, linters, and dev dependencies. To **run** the same application you need almost none of those things — just the compiled output and its runtime dependencies.

A single-stage Dockerfile ends up with everything in the final image:

```
Final image = base OS + build tools + dev dependencies + your app
```

This creates two serious problems:

1. **Image size** — a Go application needs the entire Go toolchain (~500 MB) to compile, but the resulting binary is often under 10 MB. Why ship 490 MB of compiler that will never be used at runtime?

2. **Attack surface** — every tool in the image is a potential vector for exploitation. A compiler, a package manager, or debug utilities all expand what an attacker can do if they gain container access.

**Multi-stage builds** solve this by using separate stages: one stage builds, another stage runs. The final image contains only what is copied from the build stage — nothing else.

---

## 5.2 Your First Multi-stage Build

Here is a React application built with Node.js and served by nginx:

```dockerfile
# ── Stage 1: Build ──────────────────────────────────
FROM node:20 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build         # produces /app/dist/

# ── Stage 2: Run ────────────────────────────────────
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Key syntax:**

- `AS builder` — gives the stage a name. You can use any name; `builder` is the convention.
- `--from=builder` — tells `COPY` to pull files from the named stage instead of the build context.
- The final image is built from `nginx:alpine`. It contains only nginx and the compiled static files.

**What is NOT in the final image:**

- Node.js runtime (~150 MB)
- All of `node_modules/` (~500 MB)
- Your source TypeScript/JSX files
- Build toolchain (webpack, babel, eslint...)

The final nginx image is typically **25–30 MB**.

---

## 5.3 Multi-stage: Go Application

Go is where multi-stage builds shine most dramatically. Go compiles to a fully self-contained static binary.

```dockerfile
# ── Stage 1: Build ──────────────────────────────────
FROM golang:1.21 AS builder

WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# ── Stage 2: Run ────────────────────────────────────
FROM scratch

COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**`FROM scratch`** is the ultimate minimalist base — it is completely empty. No shell, no utilities, no OS files. The only thing in this image is your binary.

**Size comparison:**

| Stage | Image | Size |
|-------|-------|------|
| Build stage | golang:1.21 | ~900 MB |
| Final image | scratch + binary | ~8 MB |

**`CGO_ENABLED=0`** disables cgo so the binary does not depend on any shared C libraries — required when using `FROM scratch`.

**Note on `FROM scratch` limitations:** No shell means no `docker exec` for debugging, no `/etc/ssl/certs` for HTTPS (you must copy it from the builder), and no `/etc/passwd` for user management. For easier debugging in development, use `FROM alpine` instead and add `RUN apk add --no-cache ca-certificates`.

---

## 5.4 Multi-stage: Python with Virtual Environment

Python does not compile to a binary, but you can still isolate build-time dependencies (compilers for C extensions, build tools) from the runtime image.

```dockerfile
# ── Stage 1: Build dependencies ──────────────────────
FROM python:3.11-slim AS builder

WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ── Stage 2: Run ─────────────────────────────────────
FROM python:3.11-slim

COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app
COPY . .

RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Why copy the virtual environment?**

The builder stage may install packages that require compilers (e.g., `psycopg2`, `Pillow`, `numpy`). Once installed, the compiled `.so` files are in the venv. We copy the finished venv — not the compilers that built them.

**Result:** The final image has no `gcc`, `make`, or build headers; only the installed Python packages.

---

## 5.5 Targeting Specific Stages

You can build any individual stage by name using `--target`. This is useful in CI pipelines:

```bash
# Build only the builder stage — useful to run tests before building the final image
docker build --target builder -t myapp:test .

# Run tests inside the builder stage
docker run --rm myapp:test npm test

# Only if tests pass, build the final production image
docker build -t myapp:prod .
```

**CI pipeline pattern:**

```bash
# 1. Build test stage and run tests
docker build --target builder -t myapp:ci .
docker run --rm myapp:ci npm test

# 2. Build production image only on success
docker build -t myapp:$GIT_SHA .
docker push myapp:$GIT_SHA
```

---

## 5.6 BuildKit — Modern Docker Build Engine

**BuildKit** is Docker's next-generation build engine. It became the default builder in Docker 23.0+ (released early 2023). If you are on an older version, enable it with:

```bash
export DOCKER_BUILDKIT=1
docker build ...
# or
DOCKER_BUILDKIT=1 docker build ...
```

### Why BuildKit Matters

| Feature | Classic builder | BuildKit |
|---------|----------------|---------|
| Parallel stage execution | No | Yes |
| Cache mounts | No | Yes |
| Secret mounts | No | Yes |
| SSH agent forwarding | No | Yes |
| Build output control | Limited | Rich |
| Unused stage skipping | No | Yes |

### Parallel Stage Execution

BuildKit analyses the dependency graph of your stages. Independent stages build concurrently:

```dockerfile
FROM node:20 AS frontend-builder
RUN npm ci && npm run build

FROM golang:1.21 AS backend-builder
RUN go build -o /server ./cmd/server

FROM alpine AS final
COPY --from=frontend-builder /app/dist /var/www/html
COPY --from=backend-builder /server /server
```

BuildKit runs `frontend-builder` and `backend-builder` in parallel, cutting build time nearly in half.

### Cache Mounts

Cache mounts let you persist package manager caches **across builds** without storing them in the image layer. This is the most impactful BuildKit feature for build speed.

```dockerfile
# npm — cache ~/.npm between builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# pip — cache ~/.cache/pip between builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# apt — cache /var/cache/apt between builds
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl

# Go modules — cache the module download cache
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
```

On first build: packages download normally. On subsequent builds: packages are served from the cache mount. Build times for the install step often drop from 60+ seconds to under 5 seconds.

### Secret Mounts

Secrets needed during build (API keys for private package registries, license keys) should **never** be baked into image layers. BuildKit secret mounts make the secret available only during that `RUN` instruction and never appear in the image history.

```dockerfile
# In Dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm config set //registry.npmjs.org/:_authToken=${NPM_TOKEN} \
    && npm ci \
    && npm config delete //registry.npmjs.org/:_authToken
```

```bash
# Build command — passes secret from a file
docker build --secret id=npm_token,src=./npm-token.txt -t myapp .

# Or from an environment variable
docker build --secret id=npm_token,env=NPM_TOKEN -t myapp .
```

The secret is available at `/run/secrets/npm_token` inside that `RUN` step only. It does not appear in any layer or in `docker history`.

### SSH Agent Forwarding

For cloning private Git repositories during build without embedding SSH keys in the image:

```dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git /app/deps
```

```bash
# Start ssh-agent and add key
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

# Build with SSH forwarding
docker build --ssh default -t myapp .
```

---

## 5.7 Image Size Comparison

The size reductions from multi-stage builds are dramatic:

| Application | Single-stage | Multi-stage | Reduction |
|-------------|-------------|-------------|-----------|
| Node.js React app | ~1.2 GB | ~25 MB | 98% |
| Go REST API | ~900 MB | ~8 MB | 99% |
| Python FastAPI | ~600 MB | ~150 MB | 75% |
| Java Spring Boot | ~800 MB | ~180 MB | 78% |

**Practical impact of smaller images:**

- Faster push/pull from registries (critical in CI/CD and autoscaling)
- Lower storage costs in container registries
- Reduced attack surface — fewer binaries an attacker can leverage
- Faster container startup (less data to unpack)

---

## 5.8 When NOT to Use Multi-stage

Multi-stage builds are not always the right choice:

**Skip multi-stage when:**

- The application is a simple interpreted script with no build step (e.g., a Bash utility)
- You are building a **development image** where you want compilers and debug tools available
- The image size difference is negligible for your use case (e.g., an internal tool run once a day)
- You are prototyping and optimisation is premature

**Always use multi-stage for:**

- Production images of compiled languages (Go, Rust, C/C++, Java, .NET)
- Frontend applications built with webpack/vite/esbuild
- Any image where security hardening is required
- Images pushed to public registries

---

## Summary

- Multi-stage builds separate the build environment from the runtime environment, producing smaller, more secure images
- Use `AS name` to label a stage and `--from=name` to copy artifacts from it
- Go applications can produce images as small as ~8 MB using `FROM scratch`
- Python benefits from copying a pre-built virtual environment without the compilers that built it
- `docker build --target stagename` builds only up to a named stage — useful for CI testing
- BuildKit (default in Docker 23.0+) adds parallel stages, cache mounts, secret mounts, and SSH forwarding
- Cache mounts (`--mount=type=cache`) are the single biggest build-speed improvement available — npm/pip caches persist across builds without entering the image
- Secret mounts (`--mount=type=secret`) allow build-time secrets that never appear in image history

---

## Knowledge Check

1. What two problems does multi-stage builds solve compared to a single-stage build?
2. In a Go multi-stage build using `FROM scratch`, why must you set `CGO_ENABLED=0`?
3. What does `COPY --from=builder /app/dist /usr/share/nginx/html` do?
4. How does a BuildKit cache mount differ from a regular `RUN` that produces cached files in the image layer?
5. Why is `--mount=type=secret` safer than `ARG MY_SECRET` for passing secrets during build?

---

## Hands-on Exercise

**Goal:** Convert a single-stage Dockerfile to multi-stage, measure the difference, and experiment with BuildKit.

**Part 1 — Baseline**

1. Take the single-stage Dockerfile you wrote in the previous chapter (or create a new one)
2. Build it: `docker build -t myapp:single .`
3. Record the image size: `docker images myapp:single`

**Part 2 — Multi-stage Conversion**

1. Add a second `FROM` stage to your Dockerfile
2. Copy only the runtime artifacts from the first stage
3. Build the multi-stage version: `docker build -t myapp:multi .`
4. Compare sizes: `docker images myapp`
5. Run the multi-stage image and verify it still works

**Part 3 — BuildKit Cache Mounts**

1. Add `--mount=type=cache` to your package install step
2. Build once (cold cache): `time docker build -t myapp:cached .`
3. Make a small change to your app source (not dependencies)
4. Build again (warm cache): `time docker build -t myapp:cached .`
5. Compare the time spent on the install step

**Part 4 — Measure with docker history**

1. Run `docker history myapp:single` and `docker history myapp:multi`
2. Identify what layers were eliminated
3. Calculate the percentage reduction in image size

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="04-dockerfile.md">← Previous: Writing Dockerfiles</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="06-volumes-and-storage.md">Next: Volumes & Storage →</a>
</div>
