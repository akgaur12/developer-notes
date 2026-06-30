# Chapter 15 — Hands-On Projects

## Overview
Four projects from beginner to production-grade capstone, applying everything in this course.

---

## Project 1 — Beginner: Containerize a Web App

**Goal:** Take an existing web application and containerize it properly from scratch.
**Skills:** Dockerfile, .dockerignore, image layers, non-root user, port mapping

**Choose one stack:**
- Node.js/Express
- Python/FastAPI
- Go

### Node.js Example

```
project/
├── src/
│   └── server.js
├── package.json
├── Dockerfile
└── .dockerignore
```

```javascript
// src/server.js
const express = require('express')
const app = express()
const port = process.env.PORT || 3000

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Docker!', version: '1.0' })
})

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() })
})

app.listen(port, () => {
  console.log(`Server running on port ${port}`)
})
```

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY src/ ./src/

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "src/server.js"]
```

```
# .dockerignore
node_modules/
.git/
*.log
.env
coverage/
```

**Verification Checklist:**
```bash
□ docker build -t myapp:1.0 .
□ docker run -d -p 3000:3000 --name myapp myapp:1.0
□ curl http://localhost:3000/
□ curl http://localhost:3000/health
□ docker inspect --format '{{.Config.User}}' myapp   # should show appuser
□ docker images myapp  # check image size (should be < 100MB)
□ docker logs myapp    # check no errors
```

**Extensions:**
- Add environment variable for configuring the greeting message
- Add a `/metrics` endpoint
- Push to Docker Hub

---

## Project 2 — Intermediate: Multi-Service Application with Compose

**Goal:** Build a full 3-tier application stack using Docker Compose with proper networking, persistence, and health checks.

**Architecture:**
```
Internet
    │
    ▼
[nginx:80]
    │
    ▼
[API: Node/Python :8000]
    │
    ├──► [PostgreSQL :5432]
    └──► [Redis :6379]
```

### File Structure
```
project/
├── api/
│   ├── src/main.py          (FastAPI app)
│   ├── requirements.txt
│   └── Dockerfile
├── nginx/
│   └── default.conf
├── .env
├── .env.example
└── docker-compose.yml
```

### docker-compose.yml
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend
    restart: unless-stopped

  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql://app:${DB_PASSWORD}@db:5432/appdb
      REDIS_URL: redis://:${REDIS_PASSWORD}@cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - frontend
      - backend
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
    networks:
      - backend
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:

networks:
  frontend:
  backend:
```

### Tasks
1. Implement the API with at least 3 endpoints: `GET /health`, `GET /items`, `POST /items`
2. Verify nginx correctly proxies to the API
3. Verify PostgreSQL data persists across `docker compose down` and `docker compose up`
4. Test network isolation: from the nginx container, can you reach `db` directly? (should be no)
5. Add log rotation to all services

---

## Project 3 — Advanced: Multi-stage Production Image with CI/CD

**Goal:** Build a production-grade image pipeline with multi-stage builds, vulnerability scanning, and automated push.

### Requirements
- Multi-stage Dockerfile (build → test → production)
- Image size < 50MB for a Node.js or < 100MB for Python app
- Pass Trivy scan with no CRITICAL vulnerabilities
- Automated via GitHub Actions (or equivalent)
- Image tagged with both version and git SHA

### Dockerfile (3-stage)
```dockerfile
# ── Stage 1: Dependencies ──────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ── Stage 2: Test ──────────────────────────────────
FROM deps AS test
COPY . .
RUN npm test

# ── Stage 3: Build ─────────────────────────────────
FROM deps AS builder
COPY . .
RUN npm run build

# ── Stage 4: Production ────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### GitHub Actions Workflow
```yaml
name: Build and Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build (test stage)
        run: docker build --target test -t myapp:test .

      - name: Build (production stage)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          load: true

      - name: Scan for vulnerabilities
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --exit-code 1 \
            --severity CRITICAL \
            ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Push
        if: github.ref == 'refs/heads/main'
        run: |
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
          docker push ghcr.io/${{ github.repository }}:latest
```

---

## Project 4 — Capstone: Production-Ready Microservices Stack

**Goal:** Deploy a complete microservices application that mirrors real production architecture.

### Architecture
```
                    ┌────────────┐
                    │  Traefik   │  :80/:443
                    │  (Proxy)   │  Auto TLS
                    └─────┬──────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
     [auth-svc]      [api-svc]       [static]
     :8001           :8002           (nginx)
          │               │
          └───────┬───────┘
                  ▼
            [PostgreSQL]   [Redis]
```

### Components

**1. Traefik as Reverse Proxy**
```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=you@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt
```

**2. Service with Traefik Labels**
```yaml
  api:
    image: myapi:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=myresolver"
      - "traefik.http.services.api.loadbalancer.server.port=8000"
```

**3. Observability with Prometheus + Grafana**
```yaml
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - monitoring
```

### Implementation Checklist
```
□ All services run as non-root
□ All services have healthchecks
□ restart: unless-stopped on all services
□ Named volumes for all persistent data
□ Network segmentation (frontend/backend/monitoring)
□ Secrets via environment, not hardcoded
□ Log rotation configured
□ Traefik handles TLS automatically
□ Prometheus scraping all services
□ Grafana dashboard showing request rate, errors, latency
```

### Extensions
- Add distributed tracing with Jaeger
- Implement rate limiting at the Traefik layer
- Add automated image updates with Watchtower
- Set up weekly security scanning with Trivy

**Knowledge Check** (5 questions focused on architecture decisions), no hands-on exercise (the capstone IS the exercise).

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="14-common-mistakes.md">← Previous: Common Mistakes</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="16-interview-preparation.md">Next: Interview Preparation →</a>
</div>
