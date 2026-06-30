# Chapter 13 — Best Practices

## Learning Objectives

By the end of this chapter, you will be able to:

- Write Dockerfiles that are optimized for layer caching and minimal image size
- Apply a security hardening checklist to any container workload
- Configure Docker Compose for production-grade reliability
- Set up a CI/CD pipeline that builds, scans, and publishes images safely
- Audit an existing deployment against a production readiness checklist

---

## 13.1 Dockerfile Best Practices

### Order Instructions for Cache Efficiency

Docker builds images layer by layer and caches each layer. A layer is invalidated the moment its instruction changes — and every subsequent layer is also invalidated. The rule is simple: put things that change least often near the top, and things that change most often near the bottom.

```dockerfile
# BAD order
FROM node:20-alpine
COPY . .                     # copies everything — cache broken on any change
RUN npm install              # re-runs every time
CMD ["node", "server.js"]

# GOOD order
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./        # only changes when deps change
RUN npm ci --only=production
COPY . .                     # source code — changes often
USER node
CMD ["node", "server.js"]
```

With the good ordering, `npm ci` is only re-run when `package.json` or `package-lock.json` changes. A code-only change still uses the cached dependency layer.

### Minimize Layers

Each `RUN` instruction creates a new layer. Intermediate files written in one `RUN` and deleted in another still exist inside the intermediate layer — they inflate the image even though they appear deleted at the final layer.

```dockerfile
# BAD: 3 layers, the package manager cache lives in layer 2 forever
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD: 1 layer, cleanup happens before the layer is committed
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

The `--no-install-recommends` flag prevents apt from pulling in suggested packages that are rarely needed, often cutting tens of megabytes from the install.

### Use Specific Tags — Never `latest` in Production

`latest` is a mutable tag. A base image upgrade can silently break your build or introduce a vulnerability. Always pin to a specific version.

```dockerfile
# BAD
FROM node:latest
FROM python:latest

# GOOD
FROM node:20.11.0-alpine3.19
FROM python:3.11.8-slim-bookworm
```

### Pin Base Image Digests for Full Reproducibility

Even pinned tags can be overwritten by a registry maintainer. For maximum reproducibility (e.g., regulated environments), pin the image digest:

```dockerfile
FROM node:20-alpine@sha256:a5127df1c6b5...
```

The digest is immutable — it refers to exactly one image manifest, forever.

---

## 13.2 Image Size Optimization Checklist

```
□ Use alpine or slim base images
□ Multi-stage builds (separate build and runtime stages)
□ .dockerignore file (exclude node_modules, .git, build artifacts, etc.)
□ --no-install-recommends with apt-get
□ --no-cache-dir with pip install
□ npm ci --only=production (exclude devDependencies)
□ Clean up package manager caches in the same RUN layer they were created
□ Remove build tools (gcc, make, etc.) in the same layer they were installed
□ Use COPY --from=builder to bring only compiled artifacts into the final stage
```

A well-optimized Node.js application image typically drops from 1–2 GB (node:latest with devDeps) to under 150 MB (node:alpine with production deps only, multi-stage).

---

## 13.3 Security Best Practices Summary

```
□ Never run as root (USER instruction)
□ --cap-drop ALL, add only the specific capabilities required
□ --read-only filesystem with --tmpfs for writable paths
□ Never bake secrets into image layers (not even in build args)
□ Scan images with Trivy in CI (fail on CRITICAL severity)
□ Use distroless or scratch for the final stage where possible
□ Don't mount the Docker socket (/var/run/docker.sock) in production containers
□ Pin image versions to prevent supply-chain surprises
□ Enable Docker Content Trust (DOCKER_CONTENT_TRUST=1) for signed images
```

---

## 13.4 Docker Compose Best Practices

```yaml
# Always set a restart policy in production
services:
  app:
    restart: unless-stopped

# Always add healthchecks so dependent services wait for readiness
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

# Use depends_on with condition: service_healthy (not just depends_on: db)
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    depends_on:
      db:
        condition: service_healthy

# Always use named volumes (not anonymous or bind mounts for persistent data)
volumes:
  pgdata:          # named — persists across down/up, inspectable, backupable
                   # (NOT "-v /var/lib/postgresql/data" which creates anonymous)

# Use .env for secrets, never hardcode in the compose file
environment:
  DB_PASSWORD: ${DB_PASSWORD}    # from .env file (not committed to git)
```

---

## 13.5 Networking Best Practices

By default, all services on a `docker-compose.yml` share a single bridge network. This means every service can reach every other service. Use explicit networks to enforce isolation.

```yaml
# Create explicit networks, don't rely on the default bridge
networks:
  frontend:
  backend:

services:
  nginx:
    networks: [frontend]
  api:
    networks: [frontend, backend]   # api bridges both zones
  db:
    networks: [backend]             # db is unreachable from nginx

# Bind published ports to localhost when sitting behind a reverse proxy
ports:
  - "127.0.0.1:3000:3000"   # not "3000:3000" which binds to 0.0.0.0
```

Binding to `127.0.0.1` means the port is only reachable from the host machine itself. A reverse proxy (nginx, Caddy) running on the same host handles public traffic and forwards to the container.

---

## 13.6 Logging Best Practices

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
        labels: "service,version"
```

Key rules:

- Always configure log rotation. The default `json-file` driver grows unbounded and will eventually fill the disk on a busy host.
- For production, ship logs to a centralized system (ELK stack, Grafana Loki, AWS CloudWatch).
- Structure logs as JSON inside the application for easier parsing and querying.
- Never log secrets, tokens, or personally identifiable information in application logs.

---

## 13.7 CI/CD Pipeline Best Practices

```yaml
# GitHub Actions pattern

- name: Build
  run: |
    docker build \
      --cache-from ghcr.io/${{ github.repository }}:cache \
      --build-arg BUILDKIT_INLINE_CACHE=1 \
      -t ghcr.io/${{ github.repository }}:${{ github.sha }} \
      -t ghcr.io/${{ github.repository }}:cache \
      .

- name: Scan
  run: trivy image --exit-code 1 --severity CRITICAL \
         ghcr.io/${{ github.repository }}:${{ github.sha }}

- name: Push
  run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Tagging strategy:**

- Tag images with the git SHA for immutability. Never overwrite a pushed SHA tag — it is your audit trail.
- Tag mutably with branch name or `latest` so tooling can always find the newest build.
- The scan step gates the push: if Trivy finds a CRITICAL vulnerability, the push is blocked and the pipeline fails.

---

## 13.8 Production Deployment Checklist

```
Build:
□ Multi-stage Dockerfile (build tools excluded from final image)
□ Non-root user (USER instruction in final stage)
□ Pinned base image tag + digest
□ .dockerignore present and comprehensive
□ Layer cache optimized (dependencies before source code)

Security:
□ Image scanned in CI (no CRITICAL vulnerabilities)
□ Secrets from Vault / AWS SSM, not environment variables
□ Read-only filesystem where possible (--read-only)
□ Linux capabilities dropped (--cap-drop ALL, add only what is needed)

Runtime:
□ Memory and CPU limits set (deploy.resources.limits)
□ Healthcheck defined and tested
□ restart: unless-stopped configured
□ Log rotation configured
□ Named volumes for all persistent data

Registry:
□ Image tagged with git SHA
□ Pushed to private registry
□ Old images cleaned up via lifecycle / retention policy
```

---

## Summary

- Order Dockerfile instructions from least-changed to most-changed to maximize cache reuse.
- Collapse related commands into a single `RUN` layer and clean up in the same layer.
- Pin base image tags and digests; never use `latest` in production.
- Use multi-stage builds, `alpine`/`slim` bases, and `.dockerignore` to keep images small.
- Run as a non-root user, drop capabilities, and scan images in CI.
- In Docker Compose, always set restart policies, healthchecks, resource limits, log rotation, and named volumes.
- Bind published ports to `127.0.0.1` when a reverse proxy sits in front.
- Tag immutably with git SHA; gate pushes on passing security scans.

---

## Knowledge Check

1. Why does putting `COPY . .` before `RUN npm install` harm build performance?
2. Why does deleting a file in a separate `RUN` instruction still leave it in the final image?
3. What is the difference between pinning a tag (`node:20-alpine`) and pinning a digest (`node:20-alpine@sha256:...`)?
4. What does `condition: service_healthy` in `depends_on` improve over a plain `depends_on`?
5. Why should published ports be bound to `127.0.0.1` rather than `0.0.0.0` when using a reverse proxy?

---

## Hands-on Exercise

**Audit and Fix an Existing Dockerfile**

1. Find or create a Dockerfile and `docker-compose.yml` for a simple web application (Node.js or Python).
2. Audit the Dockerfile against the checklist in section 13.2 and the security checklist in section 13.3. Record every violation.
3. Rewrite the Dockerfile to fix all violations: correct layer order, multi-stage build, non-root user, pinned tag, `.dockerignore`.
4. Measure the image size before and after (`docker image ls`).
5. Audit the `docker-compose.yml` against sections 13.4–13.6: add healthchecks, restart policy, named volumes, log rotation, and explicit networks.
6. Run `trivy image` against the final image and confirm zero CRITICAL findings.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="12-advanced-concepts.md">← Previous: Advanced Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="14-common-mistakes.md">Next: Common Mistakes →</a>
</div>
