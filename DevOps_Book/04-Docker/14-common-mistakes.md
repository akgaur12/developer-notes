# Chapter 14 — Common Mistakes & Pitfalls

## Learning Objectives

By the end of this chapter, you will be able to:

- Identify the most common Docker mistakes by recognising their symptoms
- Understand why each mistake occurs and what the real-world impact is
- Apply the correct pattern immediately, without having to look it up
- Use the emergency recovery commands to diagnose a broken container in production

---

## How to Read This Chapter

Each mistake is presented with four parts:

1. **The wrong pattern** — code you will encounter in the wild
2. **Why it happens** — the misunderstanding that leads to it
3. **The correct fix** — drop-in replacement
4. **How to prevent it** — lint rules, CI checks, or habits

---

## Mistake 1: Running as Root

```dockerfile
# WRONG (this is the default — Docker runs as root unless you say otherwise)
FROM node:20
COPY . .
RUN npm install
CMD ["node", "server.js"]
# Root inside the container = highest privilege if the container is escaped

# CORRECT
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN addgroup -S app && adduser -S app -G app
USER app
CMD ["node", "server.js"]
```

**Why it happens:** Root is the default. Developers often don't think about it during development and then forget to add the `USER` instruction before shipping.

**Impact:** If an attacker exploits a vulnerability in your application and escapes the container (via a kernel bug or misconfigured mount), they land as root on the host.

**Prevention:** Add a `USER` lint rule to your CI Dockerfile linter (Hadolint rule `DL3002`).

---

## Mistake 2: Using `latest` Tag in Production

```bash
# WRONG
FROM python:latest          # which version will this be next month?
docker pull nginx:latest    # silent unexpected upgrade on next deploy

# CORRECT
FROM python:3.11.8-slim-bookworm

# In docker-compose.yml — always explicit
image: myapp:${APP_VERSION:-v1.2.3}
```

**Why it happens:** `latest` is convenient during development. The mistake is carrying that habit into production pipelines.

**Impact:** A base image maintainer releases a new major version, your next deploy picks it up silently, and something breaks in production with no obvious cause.

**Prevention:** Hadolint rule `DL3007` flags `latest`. Add it to your CI pipeline.

---

## Mistake 3: Not Using .dockerignore

```bash
# Without .dockerignore, "COPY . ." sends the entire build context to the daemon
# Typical expensive mistakes:
#   node_modules/   (500 MB+)
#   .git/           (full repo history)
#   .env            (secrets!)
#   coverage/       (test artifacts)
#   *.log

# Check your build context size:
docker build . 2>&1 | head -3
# "Sending build context to Docker daemon  512MB"   ← bad
# "Sending build context to Docker daemon  85kB"    ← good

# Minimum .dockerignore for a Node.js project:
node_modules/
.git/
.env
*.log
dist/
coverage/
__pycache__/
.DS_Store
```

**Why it happens:** Developers learn `COPY . .` before they learn `.dockerignore`. The two must always be used together.

**Impact:** Slower builds (large context transfer), accidental inclusion of secrets in the image, and bloated image layers.

**Prevention:** Create `.dockerignore` before writing the first `COPY . .` instruction. Treat it as mandatory, not optional.

---

## Mistake 4: Baking Secrets into Images

```dockerfile
# WRONG — the secret is stored in an image layer permanently
ARG API_KEY
ENV API_KEY=$API_KEY
RUN curl -H "Authorization: $API_KEY" https://api.example.com/setup

# WRONG — deleting in a later layer does NOT help
RUN echo "API_KEY=$API_KEY" > .env
RUN rm .env      # too late! the previous layer still contains the file

# Verify the damage:
docker history myapp:latest   # secrets visible in layer metadata
docker save myapp:latest | tar xO | strings | grep KEY

# CORRECT — BuildKit secret mount (never written to any image layer)
# Build: docker build --secret id=api_key,src=./secrets/api_key .
RUN --mount=type=secret,id=api_key \
    curl -H "Authorization: $(cat /run/secrets/api_key)" https://api.example.com/setup
```

**Why it happens:** `ARG` and `ENV` look like a natural way to pass configuration. Developers don't realize the values are permanently recorded in layer metadata.

**Impact:** Secrets end up in the registry. Anyone with pull access to the image can extract them with `docker history` or by mounting the layer tarball.

**Prevention:** Never use `ARG` or `ENV` for secrets. Use BuildKit `--secret` mounts or inject secrets at runtime via a secrets manager (HashiCorp Vault, AWS SSM Parameter Store).

---

## Mistake 5: Installing devDependencies in the Production Image

```dockerfile
# WRONG — includes jest, webpack, typescript, eslint, and thousands of other packages
RUN npm install

# CORRECT — production dependencies only
RUN npm ci --only=production

# BEST — multi-stage keeps dev tools out of the final image entirely
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # install everything including devDeps
COPY . .
RUN npm run build             # compile TypeScript, bundle assets, etc.

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production  # only production deps in final image
CMD ["node", "dist/server.js"]
```

**Why it happens:** `npm install` is the first command every Node.js developer learns. The `--only=production` flag is a separate thing to remember.

**Impact:** Images that are 5–10x larger than necessary, longer pull times, and a vastly larger attack surface (thousands of dev tool packages with their own CVEs).

---

## Mistake 6: Wrong COPY Order Breaking Cache

```dockerfile
# WRONG — changes to any source file force npm install to re-run
COPY . .
RUN npm install

# CORRECT — npm install only re-runs when package.json changes
COPY package*.json ./
RUN npm install
COPY . .
```

**Why it happens:** `COPY . .` feels natural — "copy all my files, then install". Developers don't yet have a mental model of layer caching.

**Impact:** Every single code change causes a full `npm install` / `pip install` / `go mod download`. On a large project this adds minutes to every build.

**Prevention:** The rule: dependency manifest files first, source code last.

---

## Mistake 7: Not Setting Resource Limits

```yaml
# WRONG — one runaway container can exhaust memory and OOM-kill the host
services:
  app:
    image: myapp

# CORRECT
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: '1.0'
        reservations:
          memory: 256m
          cpus: '0.25'
```

**Why it happens:** Resource limits are not required — Docker starts containers without them by default. This feels harmless during development.

**Impact:** A memory leak, infinite loop, or traffic spike can cause a single container to consume all available memory, triggering the Linux OOM killer on the host and taking down unrelated services.

**Prevention:** Always set limits. Start with a generous estimate, then tune based on observed usage from `docker stats`.

---

## Mistake 8: Anonymous Volumes

```yaml
# WRONG — creates an anonymous volume with a random UUID name
services:
  db:
    image: postgres
    volumes:
      - /var/lib/postgresql/data    # anonymous volume

# The data persists, but:
# - You can't reference it by name in another service
# - docker compose down --volumes deletes it
# - It accumulates as docker volume ls zombie entries

# CORRECT — named volume
services:
  db:
    image: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

**Why it happens:** The anonymous form is shorter and works during development. The distinction between anonymous and named volumes is not obvious to beginners.

**Impact:** Data appears to persist but is difficult to find, cannot be referenced by name for backup jobs, and is easily lost with `docker compose down --volumes`.

---

## Mistake 9: Hardcoding Secrets in Compose Files

```yaml
# WRONG — committed to version control, visible to everyone with repo access
services:
  db:
    environment:
      POSTGRES_PASSWORD: mysupersecret123
      AWS_SECRET_KEY: AKIAIOSFODNN7EXAMPLE

# CORRECT — reference from environment or .env file
services:
  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      AWS_SECRET_KEY: ${AWS_SECRET_KEY}
```

```bash
# .env file (this file must be in .gitignore, never committed)
DB_PASSWORD=mysupersecret123
AWS_SECRET_KEY=AKIAIOSFODNN7EXAMPLE
```

**Why it happens:** Hardcoding is faster to type and "just works". The risk is invisible until the repository is cloned by an attacker.

**Impact:** Credential exposure via version control history. Even after removal, the secret remains in `git log` unless history is rewritten.

**Prevention:** Add a pre-commit hook (`detect-secrets`, `gitleaks`) to block secrets from being staged.

---

## Mistake 10: Not Configuring Log Rotation

```yaml
# WRONG — json-file logs grow forever; no rotation = disk exhaustion
services:
  app:
    image: myapp

# CORRECT
services:
  app:
    image: myapp
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
```

**Why it happens:** Log rotation is invisible — it only becomes a problem after days or weeks of running.

**Impact:** The host disk fills up. Docker and the OS start behaving erratically. The container itself may stop logging or crash.

**Prevention:** Add log rotation to your Compose template. For production, also configure a log shipping agent (Fluent Bit, Promtail) so logs are forwarded off-host before rotation discards them.

---

## Mistake 11: Publishing Ports to 0.0.0.0

```yaml
# WRONG — port 5432 is exposed on every network interface, including the public one
ports:
  - "5432:5432"    # your PostgreSQL database is now reachable from the internet

# CORRECT — only accessible from the host machine (where your reverse proxy runs)
ports:
  - "127.0.0.1:5432:5432"
```

**Why it happens:** The short form `"5432:5432"` is shown in most tutorials. The bind address is an advanced detail that developers don't encounter until something goes wrong.

**Impact:** Internal services (databases, admin panels, metrics endpoints) become reachable from the public internet, bypassing your firewall rules entirely.

**Prevention:** Use `127.0.0.1:HOST:CONTAINER` form for all services that should not be publicly exposed. Only publish ports that must be reachable from outside the host.

---

## Mistake 12: Ignoring Container Exit Codes

```bash
# WRONG — the script continues even if the container exited with error code 1
docker run myapp migrate
echo "Migration complete, proceeding..."    # runs even if migration failed!

# CORRECT
docker run myapp migrate || { echo "Migration failed!"; exit 1; }

# In a shell script, enable strict mode at the top
set -euo pipefail

# In Docker Compose
command: ["sh", "-c", "migrate && start-server"]
# If migrate fails, start-server never runs
```

**Why it happens:** Developers coming from other tools don't realize that `docker run` returns the container's exit code, and that ignoring it is the default behavior in many shell scripts.

**Impact:** A failed database migration is silently swallowed, and the next command — starting the application — runs against a corrupt or outdated schema.

---

## Mistake 13: Using Shell Form for CMD / ENTRYPOINT

```dockerfile
# WRONG: shell form — PID 1 is /bin/sh, not your application
CMD node server.js
ENTRYPOINT python app.py

# When "docker stop" sends SIGTERM, it goes to /bin/sh (PID 1), not your app.
# /bin/sh does not forward signals to child processes.
# Docker waits 10 seconds (the default stop timeout), then sends SIGKILL.
# Your app gets killed without a chance to finish in-flight requests.

# CORRECT: exec form — PID 1 is your application
CMD ["node", "server.js"]
ENTRYPOINT ["python", "app.py"]
# SIGTERM goes directly to your app → graceful shutdown
```

**Why it happens:** Shell form is shorter and works for simple cases. The signal-forwarding implication is not obvious.

**Impact:** Every `docker stop` / `docker restart` / rolling deploy triggers a hard kill after 10 seconds. In-flight HTTP requests are dropped, database transactions are aborted, and write buffers may not be flushed.

**Prevention:** Always use exec form (JSON array syntax) for `CMD` and `ENTRYPOINT`. Hadolint rules `DL3025` and `DL3000` flag shell form usage.

---

## Mistake 14: Running Docker in Production Without Orchestration

```
Symptom: you are running docker run or docker-compose up on a single machine
         and calling it a production deployment.

A single Docker host provides:
  ✓ Container isolation
  ✓ Restart on crash (restart: unless-stopped)
  ✗ High availability (one host dies = service completely down)
  ✗ Automatic restart across machines
  ✗ Rolling deploys with zero downtime
  ✗ Service discovery at scale
  ✗ Horizontal scaling

Solutions by scale:
  Small    → Docker Compose + reverse proxy (Caddy/Traefik) + health monitoring
             Keep it simple: a second VM with a hot standby is often enough.
  Medium   → Docker Swarm: minimal operational overhead, built into Docker.
  Large    → Kubernetes: covered in Topic 8 of this roadmap. Use it when you
             need multi-cluster, advanced scheduling, or a large team of operators.
```

---

## Emergency Recovery Commands

```bash
# Container won't start — inspect the last 100 lines of logs
docker logs --tail 100 container-name

# Container keeps crashing in a loop — start it with a shell instead
docker run -it --entrypoint sh myapp:latest
# Now you can run your start command manually and see the real error

# Out of disk space — check what is using it
docker system df
# Reclaim unused images, containers, volumes, and networks
docker system prune -a --volumes   # WARNING: removes all stopped containers and unused images

# Can't reach a service inside a container
docker inspect mycontainer | grep -E "IPAddress|Ports"
docker exec mycontainer ss -tlnp   # what is actually listening inside?

# Container was OOM-killed (Out Of Memory)
docker inspect mycontainer | jq '.[0].State'
# Look for: "OOMKilled": true
# Fix: increase the memory limit in your Compose file

# Unknown what is happening inside a running container
docker exec -it mycontainer sh     # drop into a shell
docker exec mycontainer env        # check environment variables
docker exec mycontainer cat /proc/1/status  # check PID 1 state
```

---

## Summary

| # | Mistake | Key Fix |
|---|---------|---------|
| 1 | Running as root | `USER` instruction in Dockerfile |
| 2 | `latest` tag in production | Pin to exact version tag |
| 3 | No `.dockerignore` | Create one before the first `COPY . .` |
| 4 | Secrets in image layers | BuildKit `--secret` mounts or runtime injection |
| 5 | devDependencies in production | `npm ci --only=production` or multi-stage build |
| 6 | Wrong `COPY` order | Dependencies manifest before source code |
| 7 | No resource limits | `deploy.resources.limits` in Compose |
| 8 | Anonymous volumes | Named volumes declared in `volumes:` block |
| 9 | Hardcoded secrets in Compose | `${ENV_VAR}` references + `.env` file |
| 10 | No log rotation | `logging.options.max-size` and `max-file` |
| 11 | Ports bound to `0.0.0.0` | `127.0.0.1:HOST:CONTAINER` form |
| 12 | Ignoring exit codes | `||` error handling or `set -euo pipefail` |
| 13 | Shell form CMD/ENTRYPOINT | Exec form (JSON array) |
| 14 | No orchestration | Swarm for medium scale, Kubernetes for large |

---

## Knowledge Check

1. Why does `RUN rm .env` in a Dockerfile layer not actually remove the secret from the image?
2. What is the difference between an anonymous volume and a named volume, and why does it matter for `docker compose down`?
3. Why does shell form for `CMD` prevent graceful shutdown, and what exactly receives the SIGTERM?
4. What is the risk of publishing a database port as `"5432:5432"` instead of `"127.0.0.1:5432:5432"`?
5. You deploy a container and it exits with code 137. What happened, and how do you confirm it?

---

## Hands-on Exercise

**Fix the Deliberately Broken Deployment**

Below is a Dockerfile and `docker-compose.yml` that contain at least 5 of the mistakes covered in this chapter. Your task is to identify every mistake, explain why each one is harmful, and rewrite both files correctly.

**Broken Dockerfile:**
```dockerfile
FROM python:latest
COPY . .
RUN pip install -r requirements.txt
ARG SECRET_KEY
ENV SECRET_KEY=$SECRET_KEY
RUN python manage.py collectstatic --noinput
CMD python manage.py runserver 0.0.0.0:8000
```

**Broken docker-compose.yml:**
```yaml
version: "3.8"
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://admin:password123@db:5432/app
    depends_on:
      - db

  db:
    image: postgres
    volumes:
      - /var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password123
```

Steps:

1. List every mistake you can find in both files (aim for at least 7).
2. Rewrite the Dockerfile: fix the base image tag, add `.dockerignore`, correct the `COPY` order, remove the secret from the build, add a non-root user, and use exec form for `CMD`.
3. Rewrite the `docker-compose.yml`: named volume, secret from `.env`, log rotation, resource limits, healthcheck, restart policy, and localhost port binding.
4. Build and run the corrected version. Verify the application starts, the healthcheck passes, and `docker inspect` shows no hardcoded secrets.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="13-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="15-projects.md">Next: Hands-On Projects →</a>
</div>
