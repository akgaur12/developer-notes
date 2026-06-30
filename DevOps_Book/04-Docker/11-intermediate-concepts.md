# Chapter 11 — Intermediate Concepts

## Learning Objectives

By the end of this chapter you will be able to:
- Apply memory and CPU resource limits to containers and understand what happens when limits are exceeded
- Write effective health checks and understand the container health state lifecycle
- Choose the correct restart policy for different deployment scenarios
- Stream and filter Docker events and container statistics
- Configure logging drivers and set up log rotation
- Use init processes to handle signals correctly inside containers

---

## 11.1 Container Resource Limits

Without limits, a single runaway container can consume all the memory or CPU on a host, affecting every other workload. Resource limits use Linux **cgroups** under the hood.

**Memory limits**

```bash
docker run -d \
  --memory=512m \              # hard limit: container is OOM-killed if exceeded
  --memory-reservation=256m \  # soft limit: hint used by the scheduler under pressure
  --memory-swap=512m \         # total memory + swap; setting equal to --memory disables swap
  nginx
```

When a container exceeds `--memory`, the Linux kernel OOM killer terminates the highest-scoring process inside it. Docker reports this as an exit code of `137` (128 + SIGKILL).

**CPU limits**

```bash
docker run -d \
  --cpus=1.5 \         # the container may use at most 1.5 CPU cores worth of time
  --cpu-shares=512 \   # relative weight for CPU scheduling (default 1024)
  nginx
```

`--cpus` is an absolute limit. `--cpu-shares` only matters when the host is under CPU contention — it is a priority hint, not a hard cap.

**Verifying limits**

```bash
# Live statistics for all running containers
docker stats

# One-shot output with custom columns
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Inspect configured limits
docker inspect --format '{{.HostConfig.Memory}}' mycontainer          # bytes
docker inspect --format '{{.HostConfig.NanoCpus}}' mycontainer        # nanocpus (divide by 1e9)

# Read directly from cgroup (Linux only)
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

**In Compose**

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

> Note: `deploy.resources` is respected by both Docker Compose v2 and Swarm. For standalone Compose (non-Swarm), you can also use the top-level `mem_limit` and `cpus` keys.

---

## 11.2 Docker Health Checks in Depth

A health check is a command Docker runs periodically inside the container to determine whether the application inside is functioning correctly — not just running.

**Dockerfile syntax**

```dockerfile
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=40s \
            --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

| Parameter | Meaning | Default |
|-----------|---------|---------|
| `--interval` | Time between health check runs | 30s |
| `--timeout` | Maximum time for the check to complete | 30s |
| `--start-period` | Grace period after container start before failures count | 0s |
| `--retries` | Consecutive failures before marking `unhealthy` | 3 |

The command must exit `0` (healthy) or `1` (unhealthy). Exit code `2` is reserved (do not use).

**Health state lifecycle**

```
Container starts
      │
      ▼
  [starting]  ◄── start-period grace; failures do not count here
      │
      ▼  (first successful check)
  [healthy]   ◄──────────────────────────────┐
      │                                       │
      │  (retries consecutive failures)       │  (next check passes)
      ▼                                       │
  [unhealthy] ───────────────────────────────►
```

**Inspecting health state**

```bash
# Quick view: STATUS column shows (healthy), (unhealthy), or (starting)
docker ps

# Detailed health history
docker inspect --format '{{json .State.Health}}' mycontainer | jq
```

**Runtime overrides**

```bash
docker run \
  --health-cmd='wget -qO- http://localhost/health || exit 1' \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=2 \
  --health-start-period=20s \
  myapp

# Disable health check entirely (e.g. for debugging)
docker run --no-healthcheck myapp
```

**Compose dependency on health**

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  app:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy   # waits until db is healthy before starting app
```

---

## 11.3 Restart Policies

Restart policies control what Docker does when a container exits.

| Policy | Behaviour | Typical use case |
|--------|-----------|-----------------|
| `no` | Never restart (default) | Development, one-off tasks |
| `on-failure[:n]` | Restart on non-zero exit; optionally limit to n retries | Apps that may crash transiently |
| `always` | Always restart, including after `docker stop` and daemon restart | Critical services |
| `unless-stopped` | Restart unless explicitly stopped with `docker stop` | Production (recommended) |

```bash
# Set at run time
docker run -d --restart=unless-stopped nginx

# Update a running container's restart policy
docker update --restart=unless-stopped existing-container

# Verify
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' existing-container
```

```yaml
# Compose
services:
  app:
    image: myapp:latest
    restart: unless-stopped
```

**`always` vs `unless-stopped`:** With `always`, stopping a container with `docker stop` then restarting the Docker daemon will cause the container to start again automatically. With `unless-stopped`, it stays stopped. Prefer `unless-stopped` for production — it respects intentional stops.

---

## 11.4 Docker Events and Monitoring

The Docker daemon emits a stream of events for every action taken on containers, images, networks, and volumes.

```bash
# Stream all events in real time
docker events

# Filter by resource type
docker events --filter type=container
docker events --filter type=network
docker events --filter type=volume

# Filter by event name
docker events --filter event=die
docker events --filter event=start
docker events --filter event=oom

# Filter by container name or ID
docker events --filter container=my-nginx

# Historical events (last 10 minutes)
docker events --since 10m

# JSON output for log shipping
docker events --format '{{json .}}'
```

**Common events to monitor in production:**

| Event | Meaning |
|-------|---------|
| `die` | Container exited (check exit code) |
| `oom` | Container was OOM-killed |
| `health_status: unhealthy` | Health check failed |
| `destroy` | Container was removed |

**Live resource statistics**

```bash
# Live stats (updates every second)
docker stats

# One-shot snapshot
docker stats --no-stream

# Custom format
docker stats --no-stream \
  --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Container process list
docker top my-nginx
docker top my-nginx aux    # pass arguments to ps
```

---

## 11.5 Exec, Attach, and Debugging Containers

**`docker exec` — run a command in a running container (preferred)**

```bash
# Interactive shell
docker exec -it mycontainer bash
docker exec -it mycontainer sh    # use sh for Alpine (no bash)

# Run a one-off command
docker exec mycontainer cat /etc/hosts
docker exec mycontainer env | sort

# Run as a specific user
docker exec -u root mycontainer bash
```

**`docker attach` — connect to the container's PID 1 stdin/stdout**

```bash
docker attach mycontainer
# Detach without stopping: Ctrl+P then Ctrl+Q
# Note: Ctrl+C sends SIGINT to PID 1, which may stop the container
```

Use `exec` for debugging. Use `attach` only when you specifically need to interact with PID 1's input/output.

**Copying files**

```bash
# Container → host
docker cp mycontainer:/app/logs/app.log ./app.log

# Host → container
docker cp ./config.json mycontainer:/app/config/config.json
```

**Debugging distroless or minimal containers (no shell)**

Distroless images have no shell, so `docker exec -it mycontainer sh` fails. Use an ephemeral sidecar container that shares the namespaces:

```bash
docker run -it --rm \
  --pid=container:mycontainer \
  --net=container:mycontainer \
  --volumes-from=mycontainer \
  busybox sh
# Now you are inside busybox but share the PID and network namespace of mycontainer
```

In Kubernetes this is built in as **ephemeral containers** (`kubectl debug`).

---

## 11.6 Container Logging Drivers

By default Docker captures stdout/stderr from containers and stores them as JSON on disk. This is the `json-file` driver.

```bash
# View logs (json-file driver only — does not work with syslog, fluentd, etc.)
docker logs mycontainer
docker logs --follow mycontainer
docker logs --tail 100 mycontainer
docker logs --since 1h mycontainer
docker logs --timestamps mycontainer
```

**Configuring the logging driver**

```bash
# Per container
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx

# Forward to syslog
docker run --log-driver syslog nginx

# Forward to systemd journal
docker run --log-driver journald nginx

# Forward to Fluentd (log aggregation)
docker run \
  --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  nginx

# Forward to AWS CloudWatch Logs
docker run \
  --log-driver awslogs \
  --log-opt awslogs-group=/docker/myapp \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-stream=mycontainer \
  nginx
```

**Set the default driver for all containers in `/etc/docker/daemon.json`:**

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Apply with `sudo systemctl reload docker`.

> Important: if you switch to a non-`json-file` driver (e.g., `fluentd`), `docker logs` no longer works because Docker is not storing the logs locally.

**Available drivers summary:**

| Driver | Description |
|--------|-------------|
| `json-file` | Default; stores JSON on disk; supports `docker logs` |
| `local` | Faster binary format; smaller on disk; supports `docker logs` |
| `syslog` | Sends to syslog daemon |
| `journald` | Sends to systemd journal |
| `fluentd` | Sends to Fluentd |
| `awslogs` | Sends to AWS CloudWatch |
| `splunk` | Sends to Splunk HTTP event collector |
| `none` | Discards all logs |

---

## 11.7 Docker Content Trust (Image Signing)

Content Trust uses Notary to sign images on push and verify signatures on pull. With it enabled, Docker refuses to pull unsigned images.

```bash
# Enable content trust for the current shell session
export DOCKER_CONTENT_TRUST=1

# Pull — fails if image is not signed
docker pull nginx

# Push — automatically signs the image
docker push myregistry/myapp:1.0
```

The modern approach is **Cosign** (from the Sigstore project), covered in Chapter 9. Cosign stores signatures as OCI artefacts in the registry itself and does not require a separate Notary server.

---

## 11.8 Environment-Specific Configuration Patterns

**Pattern 1: 12-factor app — configuration via environment variables**

```yaml
services:
  app:
    image: myapp:${APP_VERSION}
    environment:
      DATABASE_URL: ${DATABASE_URL}
      LOG_LEVEL: ${LOG_LEVEL:-info}           # default to info if not set
      FEATURE_NEW_UI: ${FEATURE_NEW_UI:-false}
```

Values are read from the shell environment or a `.env` file in the same directory as `docker-compose.yml`.

**Pattern 2: Compose file layering per environment**

```
docker-compose.yml          # base: image, ports, basic config
docker-compose.dev.yml      # dev overrides: bind mounts, debug ports, hot reload
docker-compose.prod.yml     # prod overrides: restart policy, limits, logging
```

```bash
# Development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

```yaml
# docker-compose.dev.yml (overrides)
services:
  app:
    build: .              # build from source in dev
    volumes:
      - .:/app            # live reload via bind mount
    environment:
      LOG_LEVEL: debug

# docker-compose.prod.yml (overrides)
services:
  app:
    restart: unless-stopped
    logging:
      driver: awslogs
      options:
        awslogs-group: /docker/myapp
```

---

## 11.9 init Processes in Containers

**The problem:** PID 1 in a container is special. The kernel does not send SIGTERM to orphaned child processes — that is the init system's job. If your app is not designed to be PID 1, it may:
- Ignore SIGTERM (causing slow or stuck `docker stop`)
- Leave zombie processes (children that have exited but not been reaped)

```dockerfile
# Problem: shell form CMD creates a shell as PID 1, app is PID 2
CMD node server.js          # PID 1 = /bin/sh, PID 2 = node

# Solution 1: exec form CMD — the app becomes PID 1 directly
CMD ["node", "server.js"]   # PID 1 = node
```

For applications that spawn child processes, a minimal init process like **tini** is more robust:

```dockerfile
FROM node:20-alpine
RUN apk add --no-cache tini
WORKDIR /app
COPY --chown=node:node . .
USER node
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

Tini handles:
- Forwarding signals to children
- Reaping zombie processes
- Correct exit code propagation

Alternatively, pass `--init` at runtime (Docker embeds tini):

```bash
docker run --init myapp
```

```yaml
# Compose
services:
  app:
    image: myapp:latest
    init: true
```

---

## Summary

Resource limits prevent runaway containers from affecting neighbouring workloads. Health checks give orchestrators reliable signals about application readiness, not just process liveness. Restart policies control recovery behaviour after crashes. The Docker event stream and stats commands provide real-time observability. Logging drivers decouple log storage from the container lifecycle. Using exec form for CMD and an init process like tini ensures your containers handle signals correctly and clean up after themselves.

---

## Knowledge Check

1. What happens to a container when it exceeds its `--memory` limit? What exit code does it produce?
2. What is the difference between `--memory` and `--memory-reservation`?
3. Describe the three health states a container can be in and the conditions that cause transitions between them.
4. Why is `unless-stopped` generally preferred over `always` for production services?
5. Why does `docker logs` stop working when you switch the logging driver to `fluentd`?

---

## Hands-on Exercise

**Part A — Resource limits and OOM**

1. Run a container with a 64 MB memory limit:
   ```bash
   docker run -d --memory=64m --name mem-test python:3.12-alpine \
     python -c "import time; x=[]; [x.append(b'x'*1024*1024) for _ in range(200)]; time.sleep(60)"
   ```
2. Watch `docker stats mem-test` and observe memory usage
3. Wait for the OOM kill and confirm with `docker inspect --format '{{.State.OOMKilled}}' mem-test`

**Part B — Health check lifecycle**

1. Write a simple Dockerfile for a Python HTTP server with a `/health` endpoint
2. Add a `HEALTHCHECK` instruction with a 5-second interval and `--start-period=10s`
3. Run the container and watch `docker ps` transition from `(starting)` to `(healthy)`
4. Deliberately break the health endpoint and watch it transition to `(unhealthy)`

**Part C — Logging drivers and rotation**

1. Run nginx with `--log-driver json-file --log-opt max-size=1m --log-opt max-file=3`
2. Generate some requests: `for i in $(seq 1 100); do curl -s localhost:80 > /dev/null; done`
3. Locate the log file in `/var/lib/docker/containers/<id>/` and inspect its format
4. Reconfigure to use `--log-driver none` and confirm `docker logs` no longer works

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="10-security.md">← Previous: Docker Security</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="12-advanced-concepts.md">Next: Advanced Concepts →</a>
</div>
