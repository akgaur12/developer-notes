# Chapter 5 — Docker in CI/CD

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why Docker and CI/CD pipelines are a natural fit
- Build Docker images in CI using BuildKit and the `build-push-action`
- Push images to multiple registries (GHCR, Docker Hub, ECR) in a single pipeline
- Generate consistent image tags and OCI labels using the metadata action
- Scan images for vulnerabilities before promoting them
- Structure a multi-stage build pipeline so test and production stages share cache
- Run integration tests with Docker Compose service containers
- Build multi-platform images for AMD64 and ARM64 in CI
- Implement an image promotion pattern that avoids rebuilding artifacts

---

## 5.1 Why Docker and CI/CD Go Together

Docker and CI/CD pipelines complement each other naturally:

| Benefit | Explanation |
|---|---|
| Reproducible builds | The same `Dockerfile` produces the same image on every runner |
| Eliminates environment drift | CI builds the same image that production runs — "works on my machine" becomes irrelevant |
| Immutable artifacts | An image digest (`sha256:abc...`) is a permanent, content-addressed artifact |
| Portable stages | Each pipeline step can run inside a specific container, controlling its own dependencies |
| Clear promotion path | The same image bytes move through dev → staging → prod without being rebuilt |

The core principle: **build once, run everywhere**. The CI pipeline builds the image, tags it with the commit SHA, then all subsequent stages (test, staging, production) run that exact image.

---

## 5.2 Building Docker Images in CI

**Basic build:**

```yaml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .
```

**With BuildKit — faster, with layer caching and build secrets:**

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build
  uses: docker/build-push-action@v5
  with:
    context: .
    push: false
    tags: myapp:${{ github.sha }}
    cache-from: type=gha          # read cache from GitHub Actions cache
    cache-to: type=gha,mode=max   # write all layers to cache
```

`mode=max` caches all intermediate layers, not just the final image. This is slower to write but dramatically speeds up subsequent builds when only later layers change.

`push: false` builds the image and loads it into the local Docker daemon without pushing to a registry — useful for test jobs that need to run the image locally before deciding whether to push.

---

## 5.3 Pushing to Multiple Registries

You can log in to multiple registries in the same job and push to all of them with one `build-push-action` call.

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Login to ECR
  uses: aws-actions/amazon-ecr-login@v2

- name: Build and push to all registries
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: |
      ghcr.io/myorg/myapp:${{ github.sha }}
      ghcr.io/myorg/myapp:latest
      myorg/myapp:${{ github.sha }}
      123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }}
```

The `tags` field accepts a multiline string — each line is a separate tag. All tags point to the same image digest; only the references differ.

---

## 5.4 Docker Image Metadata and Labels

The `docker/metadata-action` generates consistent tags and OCI-compliant labels based on git context (branch, tag, PR number, SHA).

```yaml
- name: Docker metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: |
      ghcr.io/myorg/myapp
    tags: |
      type=sha                                   # sha-abc1234
      type=semver,pattern={{version}}            # v1.2.3 (on tag push)
      type=semver,pattern={{major}}.{{minor}}    # v1.2
      type=ref,event=branch                      # main, develop
      type=ref,event=pr                          # pr-42
    labels: |
      org.opencontainers.image.title=My App
      org.opencontainers.image.description=My Application

- uses: docker/build-push-action@v5
  with:
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
```

The `meta` step's outputs adapt automatically to what triggered the workflow:

- On a push to `main`: produces `sha-abc1234` and `main` tags
- On a version tag `v1.2.3`: produces `v1.2.3`, `v1.2`, and `sha-abc1234` tags
- On a PR: produces `pr-42` and `sha-abc1234` tags

---

## 5.5 Vulnerability Scanning in CI

Scan images for known CVEs before they reach production. Fail the pipeline on critical vulnerabilities.

```yaml
- name: Scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: '1'           # fail pipeline if CRITICAL vulnerabilities found

- name: Upload scan results to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  if: always()               # upload even if the scan step failed
  with:
    sarif_file: trivy-results.sarif
```

The SARIF upload makes vulnerability findings appear in GitHub's Security tab under "Code scanning alerts", with file-level annotations in pull requests.

**Other scanners to consider:**

- **Grype** (Anchore): fast, offline-capable
- **Snyk Container**: developer-friendly with fix suggestions
- **Docker Scout**: integrated into Docker Hub
- **Clair**: self-hostable, used in many enterprise setups

---

## 5.6 Multi-stage Build in CI (Test → Build → Push)

Structure your pipeline to separate the test phase from the production build phase, while sharing the layer cache between them.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      # Build only the test stage — fast, uses shared cache
      - name: Build test stage
        uses: docker/build-push-action@v5
        with:
          target: test
          load: true            # load into local Docker daemon (not push)
          tags: myapp:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # Run tests inside the test-stage container
      - name: Run tests
        run: docker run --rm myapp:test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Build production stage and push — only on main
      - uses: docker/build-push-action@v5
        with:
          target: production
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha    # re-uses layers cached by the test job
          cache-to: type=gha,mode=max
```

Your `Dockerfile` should define named stages:

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM deps AS test
COPY . .
CMD ["npm", "test"]

FROM deps AS production
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

The `test` and `production` stages both inherit from `deps`, so the dependency installation layer is cached and shared.

---

## 5.7 Docker Compose in CI (Integration Tests)

Use Docker Compose for integration tests that need multiple services running together.

```yaml
- name: Start services
  run: docker compose -f docker-compose.test.yml up -d

- name: Wait for services to be healthy
  run: |
    timeout 60 sh -c 'until docker compose ps | grep -q "healthy"; do sleep 2; done'

- name: Run integration tests
  run: docker compose exec -T api pytest tests/integration/

- name: Collect logs on failure
  if: failure()
  run: docker compose logs

- name: Teardown
  if: always()
  run: docker compose -f docker-compose.test.yml down -v
```

The `-T` flag in `docker compose exec -T` disables pseudo-TTY allocation, which is required in non-interactive CI environments.

The `if: always()` on teardown ensures containers are cleaned up even when tests fail. The `-v` flag removes named volumes to prevent state leaking between runs.

**Example `docker-compose.test.yml`:**

```yaml
services:
  api:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://postgres:test@postgres:5432/testdb

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

---

## 5.8 Multi-platform Builds in CI

Build images that run natively on both AMD64 (standard cloud servers) and ARM64 (Apple Silicon, AWS Graviton, Raspberry Pi).

```yaml
- uses: docker/setup-qemu-action@v3      # enables ARM emulation on AMD64 runners
- uses: docker/setup-buildx-action@v3   # enables multi-platform BuildKit

- uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ghcr.io/myorg/myapp:${{ github.sha }}
```

The result is a multi-architecture image manifest. When a user pulls `ghcr.io/myorg/myapp:sha`, Docker automatically selects the correct platform variant for their machine.

**Performance note:** ARM builds via QEMU emulation on AMD64 runners are slow — 2–10x slower than native ARM. For large images, consider using GitHub-hosted ARM runners (available on GitHub Team/Enterprise plans) or self-hosted ARM runners.

---

## 5.9 Image Promotion Pattern

Never rebuild an image for staging or production deployments. Instead, promote the image that already passed tests by retagging it.

```
CI builds → ghcr.io/myorg/myapp:sha-abc123   (immutable, tied to commit)
                        ↓ (unit + integration tests pass)
            ghcr.io/myorg/myapp:staging       (mutable alias, retagged)
                        ↓ (staging smoke tests pass, approved)
            ghcr.io/myorg/myapp:prod          (mutable alias, retagged)
            ghcr.io/myorg/myapp:v1.2.3        (immutable release tag)
```

**Retagging without rebuilding:**

```bash
# Pull the tested image
docker pull ghcr.io/myorg/myapp:sha-abc123

# Retag for staging
docker tag ghcr.io/myorg/myapp:sha-abc123 ghcr.io/myorg/myapp:staging
docker push ghcr.io/myorg/myapp:staging
```

Or use the registry's native copy/tag API (faster, no pull/push bandwidth):

```bash
# Using crane (Google's container registry tool)
crane tag ghcr.io/myorg/myapp:sha-abc123 staging
```

The image digest (`sha256:...`) remains the same through all retagging. Deployments that pin the digest guarantee they run the exact bytes that were tested.

---

## Summary

| Topic | Key Point |
|---|---|
| Build reproducibility | Docker ensures CI and prod run identical environments |
| BuildKit caching | `type=gha` cache dramatically speeds up repeated builds |
| Multiple registries | One `build-push-action` call can push to GHCR, Docker Hub, and ECR simultaneously |
| Metadata action | Generates consistent, context-aware tags from git events |
| Vulnerability scanning | Trivy + SARIF upload integrates scan results into GitHub Security tab |
| Multi-stage CI | Test stage and build stage share the same layer cache |
| Docker Compose in CI | Run multi-service integration tests with `always()` teardown |
| Multi-platform | QEMU + Buildx enables `linux/amd64,linux/arm64` from one runner |
| Image promotion | Retag instead of rebuild; digest stays immutable through promotion |

---

## Knowledge Check

1. What does `push: ${{ github.ref == 'refs/heads/main' }}` do in a `build-push-action` step, and why is this a useful pattern?
2. Explain how the `docker/metadata-action` produces different tags depending on whether the trigger is a branch push, a PR, or a version tag.
3. Why is it important to use `if: always()` on the Docker Compose teardown step?
4. You want to build for `linux/amd64` and `linux/arm64` in CI. What two setup actions do you need before `build-push-action`, and what does each one provide?
5. What is the image promotion pattern, and why is it preferable to rebuilding the image for each environment?

---

## Hands-on Exercise

**Goal:** Build a complete Docker CI pipeline using the techniques from this chapter.

1. Write a `Dockerfile` with at least two stages: `test` and `production`.
2. Create a GitHub Actions workflow that:
   - Builds the `test` stage and runs tests inside that container
   - On success, builds the `production` stage
   - Uses `type=gha` caching for both jobs
   - Only pushes to GHCR when the branch is `main`
3. Add the `docker/metadata-action` step to generate tags for SHA, branch name, and semver (on version tags).
4. Add a Trivy vulnerability scan step after the build, uploading results as SARIF.
5. Configure the pipeline to build for both `linux/amd64` and `linux/arm64`.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="04-github-actions-advanced.md">← Previous: GitHub Actions Advanced</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="06-testing-strategies.md">Next: Testing Strategies →</a>
</div>
