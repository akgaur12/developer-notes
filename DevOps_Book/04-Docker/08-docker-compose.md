# Chapter 8 — Docker Compose

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why Docker Compose exists and what problems it solves
- Read and write a `docker-compose.yml` file for a multi-service application
- Use essential Compose CLI commands to manage the full application lifecycle
- Manage secrets and configuration with `.env` files
- Layer multiple Compose files for dev/staging/production environments
- Configure healthchecks and `depends_on` to control startup order

---

## 8.1 Why Docker Compose?

Running a multi-container application with raw `docker run` commands quickly becomes unmanageable:

```bash
# Without Compose — fragile, hard to share, easy to get wrong
docker network create myapp-net
docker volume create pgdata
docker run -d --name db --network myapp-net \
  -e POSTGRES_USER=app -e POSTGRES_PASSWORD=s3cr3t \
  -v pgdata:/var/lib/postgresql/data postgres:15
docker run -d --name redis --network myapp-net redis:7-alpine
docker run -d --name api --network myapp-net \
  -e DATABASE_URL=postgresql://app:s3cr3t@db:5432/appdb \
  -p 3000:3000 myapi:latest
```

Problems with this approach:
- Easy to forget a flag or environment variable
- Hard to share with teammates or check into version control
- No single command to start or stop everything
- No guarantee containers start in the right order

**Docker Compose** solves all of this with a single YAML file and one command: `docker compose up`.

---

## 8.2 docker-compose.yml Structure

```yaml
services:          # define each container
  service-name:
    image:         # pre-built image to use
    build:         # OR build from a Dockerfile
    ports:         # host:container port mappings
    environment:   # environment variables
    volumes:       # volume or bind-mount mappings
    networks:      # which networks to join
    depends_on:    # startup ordering
    restart:       # restart policy
    healthcheck:   # how to check if service is healthy

volumes:           # named volumes used by services
  vol-name:

networks:          # custom networks
  net-name:
```

Compose automatically creates a default network for your project (named `<project>_default`), so all services can reach each other by name even without explicit `networks:` blocks. Defining networks explicitly gives you more control over isolation.

---

## 8.3 A Complete Real-World Example: Web App + DB + Cache

```yaml
services:
  # PostgreSQL database
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}   # from .env file
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # Redis cache
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
    networks:
      - backend

  # Application API
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://app:${DB_PASSWORD}@db:5432/appdb
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
      - frontend

  # Nginx reverse proxy
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/ssl/certs:ro
    depends_on:
      - api
    networks:
      - frontend

volumes:
  pgdata:
  redisdata:

networks:
  backend:
  frontend:
```

Key patterns to notice:
- `db` and `redis` are on the `backend` network only — not directly reachable from `nginx`
- `api` bridges both networks, acting as the trust boundary
- `nginx` only talks to `api` via the `frontend` network
- `api` waits for `db` and `redis` to pass their healthchecks before starting

---

## 8.4 Essential Compose Commands

```bash
# Start all services in the background
docker compose up -d

# Start with a fresh build (rebuild images before starting)
docker compose up -d --build

# Stop all services and remove containers (volumes are preserved)
docker compose down

# Stop and also remove named volumes — DESTRUCTIVE, deletes data
docker compose down -v

# View logs for all services
docker compose logs

# Follow logs for a specific service
docker compose logs -f api

# Show last 50 lines then follow
docker compose logs --tail 50 -f api

# Open a shell in a running service
docker compose exec api bash
docker compose exec db psql -U app -d appdb

# Scale a service to multiple replicas
docker compose up -d --scale api=3

# Show running services and their status
docker compose ps

# Show processes inside each service container
docker compose top

# Restart a single service without affecting others
docker compose restart api

# Rebuild and replace a single service
docker compose up -d --build api
```

---

## 8.5 Environment Variables and .env Files

Never hardcode secrets in your Compose file. Use a `.env` file in the same directory:

```bash
# .env
DB_PASSWORD=supersecret123
REDIS_PASSWORD=redissecret456
NODE_ENV=production
```

Reference variables in your Compose file with `${VARIABLE_NAME}`:

```yaml
environment:
  POSTGRES_PASSWORD: ${DB_PASSWORD}
```

**Important rules:**
- Add `.env` to `.gitignore` — never commit secrets to version control
- Commit a `.env.example` file with placeholder values so teammates know what to set
- For CI/CD, inject variables via the pipeline's secret management (GitHub Actions secrets, GitLab CI variables, etc.)

You can also pass a different env file:

```bash
docker compose --env-file .env.staging up -d
```

---

## 8.6 Multiple Compose Files (Overrides)

Compose supports layering files so you can share a base config and override only what differs per environment.

**Base file** (`docker-compose.yml`) — shared production-like config:

```yaml
services:
  api:
    image: myapi:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
```

**Development override** (`docker-compose.override.yml`) — auto-applied when you run `docker compose up` locally:

```yaml
services:
  api:
    build:
      context: .
    volumes:
      - .:/app          # live code reload
    environment:
      NODE_ENV: development
    command: npm run dev
```

**Production deployment** — explicitly merge two files:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**Test environment:**

```bash
docker compose -f docker-compose.yml -f docker-compose.test.yml up --abort-on-container-exit
```

This pattern keeps your base config DRY while allowing environment-specific customisation without duplication.

---

## 8.7 Healthchecks and depends_on

`depends_on` controls startup order. Without conditions, it only waits for the container to **start**, not for the application inside to be **ready**.

```yaml
depends_on:
  db:
    condition: service_healthy          # waits for healthcheck to pass
  redis:
    condition: service_started          # just waits for container start (default)
  migration:
    condition: service_completed_successfully  # waits for exit code 0
```

For `service_healthy` to work, the dependency must define a `healthcheck`:

```yaml
db:
  image: postgres:15
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U app"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s   # grace period before first check counts as failure
```

This is critical for databases and caches — they need a few seconds to initialise before they can accept connections, and `depends_on: service_started` will not wait for that.

---

## 8.8 Production Considerations

| Topic | Guidance |
|-------|---------|
| Scale | Docker Compose is not designed for large-scale production; use Kubernetes for that |
| Sweet spot | Local development, CI pipelines, small single-server deployments, integration tests |
| Restart policy | Always set `restart: unless-stopped` for production containers |
| Secrets | Never hardcode in the Compose file; always use env vars or Docker Secrets |
| Image tags | Pin to specific versions (e.g. `postgres:15.3-alpine`), never `latest` in production |
| Resource limits | Set `deploy.resources.limits` to prevent a runaway container from starving the host |

```yaml
# Resource limits example
services:
  api:
    image: myapi:1.2.0
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M
```

---

## Summary

- Docker Compose replaces chains of `docker run` commands with a single declarative YAML file.
- All services in a Compose project share a default network and can resolve each other by service name.
- Use `depends_on` with `condition: service_healthy` and proper `healthcheck` blocks to guarantee correct startup ordering.
- Keep secrets in `.env` files, never in `docker-compose.yml`, and never commit `.env` to version control.
- Layer Compose files (`-f base.yml -f override.yml`) to share config across environments without duplication.
- Compose is excellent for local dev and small deployments; Kubernetes is the right tool at scale.

---

## Knowledge Check

1. What is the difference between `docker compose down` and `docker compose down -v`? When is each appropriate?
2. A `db` service takes 20 seconds to be ready after startup. The `api` service fails because it can't connect on boot. How do you fix this using only Compose configuration?
3. You have a secret `DB_PASSWORD` that must not be committed to git. What two files do you create, and what goes in each?
4. A teammate checks out your repo and runs `docker compose up` — their `api` gets a different Dockerfile than production. How did this happen, and is it intentional?
5. You need to run 3 replicas of the `api` service. What single command achieves this without restarting `db` or `redis`?

---

## Hands-on Exercise

**Goal:** Build and operate a real multi-service application with Compose.

1. Create a project directory with a `docker-compose.yml` defining three services: `nginx` (image: `nginx:alpine`), `api` (your choice of simple image, e.g. `node:20-alpine` running a hello-world server), and `db` (image: `postgres:15-alpine`).
2. Add a `healthcheck` to `db` and configure `api` with `depends_on: db: condition: service_healthy`.
3. Create a `.env` file with `DB_PASSWORD` and reference it in the Compose file. Add `.env` to `.gitignore`.
4. Run `docker compose up -d` and verify all services reach `healthy` status with `docker compose ps`.
5. Tail the logs of `api` with `docker compose logs -f api`.
6. Exec into `db` and run `psql -U postgres` to confirm the database is accessible.
7. Run `docker compose down` then `docker compose down -v` and observe the difference in what gets removed.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-networking.md">← Previous: Docker Networking</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-registry-and-image-management.md">Next: Registry & Image Management →</a>
</div>
