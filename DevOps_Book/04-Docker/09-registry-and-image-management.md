# Chapter 9 — Registry & Image Management

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what a container registry is and name the major public and cloud options
- Push and pull images to Docker Hub, GitHub Container Registry, and AWS ECR
- Apply a consistent, production-safe image tagging strategy
- Scan images for vulnerabilities using Docker Scout and Trivy
- Sign images with Cosign for supply-chain integrity
- Run a local private registry and explain when to use Harbor instead
- Clean up unused images and monitor Docker disk usage

---

## 9.1 What Is a Container Registry?

A container registry is a content-addressable storage service for OCI (Open Container Initiative) images — think of it as npm for container images. You `push` an image to share it and `pull` it to deploy it.

**Public registries** (free tier available):

| Registry | URL | Notes |
|----------|-----|-------|
| Docker Hub | `docker.io` | Default registry; largest public image library |
| GitHub Container Registry | `ghcr.io` | Tied to GitHub org/user; excellent GitHub Actions integration |
| Quay.io | `quay.io` | Red Hat; strong scanning; popular for open-source projects |

**Managed cloud registries** (private, pay-per-use):

| Provider | Service | URL pattern |
|----------|---------|-------------|
| AWS | Elastic Container Registry (ECR) | `<account>.dkr.ecr.<region>.amazonaws.com` |
| Google Cloud | Artifact Registry | `<region>-docker.pkg.dev/<project>/<repo>` |
| Azure | Azure Container Registry (ACR) | `<name>.azurecr.io` |

**Self-hosted:**
- **Harbor** — CNCF graduated project; adds RBAC, vulnerability scanning, replication, and a web UI on top of a basic registry
- **Nexus Repository** — polyglot (npm, Maven, Docker, etc.) artifact store
- **GitLab Container Registry** — built into GitLab, zero additional setup

---

## 9.2 Docker Hub

```bash
# Authenticate
docker login

# Tag your local image with your Docker Hub username
docker tag myapp:1.0 yourusername/myapp:1.0

# Push to Docker Hub
docker push yourusername/myapp:1.0

# Pull from Docker Hub (public repo — no login required)
docker pull yourusername/myapp:1.0

# Log out when done (especially on shared machines)
docker logout
```

Docker Hub rate limits unauthenticated pulls (100/6h per IP) and authenticated free-tier pulls (200/6h per user). In CI, always authenticate to avoid hitting these limits.

---

## 9.3 GitHub Container Registry (ghcr.io)

```bash
# Create a Personal Access Token (PAT) with read:packages and write:packages scopes
# Then authenticate:
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Tag for ghcr.io
docker tag myapp:1.0 ghcr.io/your-org/myapp:1.0

# Push
docker push ghcr.io/your-org/myapp:1.0
```

In **GitHub Actions**, no manual secret is needed — the built-in `GITHUB_TOKEN` has package write permissions:

```yaml
- name: Log in to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## 9.4 AWS Elastic Container Registry (ECR)

ECR authentication tokens expire every 12 hours, so you must re-authenticate before each push in CI.

```bash
# Authenticate Docker to your ECR registry
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create a repository (one-time setup)
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

# Tag using the full ECR URI
docker tag myapp:1.0 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

**ECR Lifecycle Policies** — automatically expire old images to control storage costs:

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep only the last 10 images",
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 10
    },
    "action": { "type": "expire" }
  }]
}
```

```bash
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy file://lifecycle-policy.json
```

---

## 9.5 Image Tagging Strategy

A consistent tagging strategy is critical for traceability and safe rollbacks.

```bash
# Semantic versioning (recommended for releases)
docker tag myapp:latest myapp:v2.1.0
docker tag myapp:latest myapp:v2.1
docker tag myapp:latest myapp:v2

# Git commit SHA (immutable — always points to exact code)
docker tag myapp:latest myapp:$(git rev-parse --short HEAD)

# Combined: version + SHA (best of both worlds)
VERSION=v2.1.0
SHA=$(git rev-parse --short HEAD)
docker tag myapp myapp:${VERSION}-${SHA}

# Branch-based (for ephemeral dev/staging environments)
docker tag myapp myapp:main
docker tag myapp myapp:feature-login
```

**Production rule: never deploy with the `latest` tag.**

`latest` is mutable — it moves to a different image on every push. Using it in production means you cannot reliably roll back, reproduce a deploy, or audit what is running. Always pin to a specific version or SHA tag.

| Tag type | Mutable | Safe for prod | Use case |
|----------|---------|---------------|----------|
| `latest` | Yes | No | Local development only |
| `main` / branch | Yes | No | Staging / ephemeral envs |
| `v2.1.0` | No | Yes | Releases |
| `abc1234` (SHA) | No | Yes | Audit trails, GitOps |

---

## 9.6 Image Scanning and Security

Vulnerability scanning checks the OS packages and language dependencies inside your image against known CVE databases.

```bash
# Docker Scout (built into Docker Desktop and Docker CLI)
docker scout cves myapp:1.0
docker scout recommendations myapp:1.0    # suggests base image upgrades

# Trivy (open-source, fast, comprehensive — recommended for CI)
trivy image myapp:1.0

# Filter by severity
trivy image --severity HIGH,CRITICAL myapp:1.0

# Fail the CI pipeline if any CRITICAL CVE is found
trivy image --exit-code 1 --severity CRITICAL myapp:1.0

# Snyk (commercial; deep language ecosystem support)
snyk container test myapp:1.0
```

**Integrate scanning into CI** — run it as a gate before pushing to your registry:

```yaml
# GitHub Actions step
- name: Scan image
  run: trivy image --exit-code 1 --severity CRITICAL myapp:${{ github.sha }}
```

Scanning after build but before push ensures you never ship a critically vulnerable image to production.

---

## 9.7 Image Signing

Image signing proves that an image was built by your CI pipeline and has not been tampered with — a key part of software supply chain security.

**Cosign** (CNCF project — signs OCI images stored in any registry):

```bash
# Generate a key pair (store the private key securely — e.g. in CI secrets)
cosign generate-key-pair

# Sign the image after pushing it
cosign sign --key cosign.key myregistry.io/myapp:1.0

# Verify the signature before deploying
cosign verify --key cosign.pub myregistry.io/myapp:1.0
```

The signature is stored as an OCI artifact alongside the image in the registry — no separate storage needed. Policy engines like **Kyverno** or **OPA Gatekeeper** can enforce that only signed images are admitted to a Kubernetes cluster.

---

## 9.8 Running a Private Registry

For testing or air-gapped environments, you can run Docker's official `registry:2` image:

```bash
# Start a local registry on port 5000
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v registrydata:/var/lib/registry \
  --restart unless-stopped \
  registry:2

# Tag an image for the local registry
docker tag myapp:1.0 localhost:5000/myapp:1.0

# Push to local registry
docker push localhost:5000/myapp:1.0

# Pull from local registry
docker pull localhost:5000/myapp:1.0

# List images in the registry via its HTTP API
curl http://localhost:5000/v2/_catalog
```

For production self-hosting, use **Harbor** instead. It adds:
- Role-based access control (RBAC)
- Integrated vulnerability scanning (Trivy)
- Image replication between registries
- Web UI and audit logging
- Proxy cache for Docker Hub (avoids rate limits)

---

## 9.9 Image Optimization and Maintenance

**Inspect image layers:**

```bash
# See total image size
docker image inspect myapp:1.0 | jq '.[0].Size'

# See the size contribution of each layer
docker history --human myapp:1.0

# Dive — interactive TUI for exploring layer contents
dive myapp:1.0
```

**Clean up disk space:**

```bash
# Remove dangling images (untagged layers from failed/superseded builds)
docker image prune

# Remove ALL images not currently used by a running container
docker image prune -a

# See Docker's overall disk usage (images, containers, volumes, build cache)
docker system df

# Nuclear option — remove everything unused (images, stopped containers, volumes, networks, build cache)
docker system prune --volumes
```

Run `docker system df` regularly in CI environments — build caches accumulate quickly and can fill disks.

---

## 9.10 CI/CD Integration Pattern

A complete build-tag-scan-push pipeline in GitHub Actions:

```yaml
name: Build and publish

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build image
        run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .

      - name: Scan for vulnerabilities
        run: trivy image --exit-code 1 --severity CRITICAL \
               ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Push image
        run: |
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

          # Also tag as latest on main branch
          if [ "${{ github.ref }}" = "refs/heads/main" ]; then
            docker tag ghcr.io/${{ github.repository }}:${{ github.sha }} \
                       ghcr.io/${{ github.repository }}:latest
            docker push ghcr.io/${{ github.repository }}:latest
          fi
```

Key points in this pipeline:
- The scan runs **before** push — a critical CVE aborts the pipeline without publishing the image
- The SHA tag is always pushed — provides an immutable reference for GitOps/deployments
- `latest` is updated only on main — branch builds get a SHA-only tag

---

## Summary

- Container registries store and distribute OCI images; choose the registry that matches your infrastructure (Docker Hub, ghcr.io, ECR, GCR, ACR).
- Tag images with semantic versions and/or git SHAs; never deploy `latest` to production.
- Integrate vulnerability scanning (Trivy or Docker Scout) as a CI gate — scan before push, fail on CRITICAL.
- Sign images with Cosign for supply-chain integrity in regulated or high-security environments.
- The `registry:2` image works for local testing; use Harbor for self-hosted production registries.
- Run `docker system prune` and `docker image prune -a` regularly to reclaim disk space, especially in CI.

---

## Knowledge Check

1. Why is `latest` unsafe to use in production deployments? What tag strategy should replace it?
2. What is the difference between `docker image prune` and `docker image prune -a`? Which command is safe to run on a production host?
3. ECR authentication tokens expire every 12 hours. What does this mean for a CI pipeline that builds and pushes images?
4. You want to guarantee that only images built by your CI pipeline are deployed to your Kubernetes cluster. Which tool enables this, and what does it create in the registry?
5. A teammate reports that the CI server's disk is full. You suspect Docker is the culprit. What command shows you a breakdown of Docker's disk usage?

---

## Hands-on Exercise

**Goal:** Complete the full image lifecycle from build to scan to publish.

1. Build a simple Docker image (use any small app or just `FROM alpine` with a `CMD`).
2. Create a free account on Docker Hub or GitHub, then authenticate Docker to that registry.
3. Tag the image with both a semantic version (`v0.1.0`) and the current git SHA (`git rev-parse --short HEAD`).
4. Install Trivy (`brew install trivy` on Mac, or the official install script on Linux) and run a scan: `trivy image <your-image>:v0.1.0`.
5. Push both tags to the registry and verify they appear in the web UI.
6. Run `docker system df` to see current disk usage, then run `docker image prune -a` (after confirming no containers rely on those images) and run `docker system df` again to see the difference.
7. Pull your own image from the registry on a clean session (`docker logout` first if testing public access) to confirm it is publicly accessible.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-docker-compose.md">← Previous: Docker Compose</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-security.md">Next: Docker Security →</a>
</div>
