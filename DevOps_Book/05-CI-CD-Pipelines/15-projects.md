# Chapter 15 — Hands-On Projects

## Overview
Four projects from beginner to capstone.

---

## Project 1 — Beginner: First CI Pipeline

**Goal:** Add a complete CI pipeline to an existing project.
**Skills:** GitHub Actions basics, tests in CI, branch protection

**Requirements:**
- Run linting on every push and PR
- Run unit tests with coverage reporting
- Upload test artifacts
- Block merges to main if tests fail (branch protection)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm

      - run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run type-check

      - name: Test
        run: npm test -- --coverage --reporters=default --reporters=jest-junit

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            junit.xml
            coverage/
          retention-days: 7
```

**Verification Checklist:**
```
□ Pipeline runs on every push and PR
□ Lint failure blocks the PR
□ Test failure blocks the PR
□ Coverage report visible in artifacts
□ Branch protection rule requires CI to pass before merge
□ Pipeline completes in under 5 minutes
```

**Extensions:**
- Add a Codecov integration for coverage tracking
- Add a badge to your README showing CI status
- Add Dependabot for automated dependency updates

---

## Project 2 — Intermediate: Docker Build and Deploy Pipeline

**Goal:** Build a complete build, push, and deploy pipeline for a containerized app.
**Skills:** Docker in CI, multi-environment deploy, secrets management

**Architecture:**
```
PR → CI (tests + lint)
main push → CI → Build image → Scan → Push to GHCR → Deploy to Staging
release tag → Retag image → Deploy to Production (with approval)
```

```yaml
# .github/workflows/build-deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  release:
    types: [published]

jobs:
  test:
    uses: ./.github/workflows/ci.yml    # reuse CI workflow

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:sha-${{ github.sha }}
          exit-code: '1'
          severity: CRITICAL

  deploy-staging:
    needs: scan
    environment:
      name: staging
      url: https://staging.myapp.com
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          kubectl set image deployment/myapp \
            app=ghcr.io/${{ github.repository }}:sha-${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=5m
        env:
          KUBECONFIG_DATA: ${{ secrets.STAGING_KUBECONFIG }}

      - name: Smoke test
        run: ./scripts/smoke-test.sh https://staging.myapp.com

  deploy-production:
    needs: deploy-staging
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          IMAGE=ghcr.io/${{ github.repository }}:${{ github.event.release.tag_name }}
          kubectl set image deployment/myapp app=$IMAGE
          kubectl rollout status deployment/myapp --timeout=5m
```

---

## Project 3 — Advanced: Full GitOps Pipeline

**Goal:** Implement a true GitOps workflow — CI builds the image, updates a manifest repository, and ArgoCD handles the actual deployment.
**Skills:** GitOps, multi-repo workflows, ArgoCD, automated rollbacks

**Repository structure:**
```
myapp-repo/            ← application code + CI pipeline
myapp-manifests/       ← Kubernetes manifests (the GitOps repo)
  ├── base/
  │   ├── deployment.yaml
  │   └── service.yaml
  └── overlays/
      ├── staging/
      │   └── kustomization.yaml
      └── production/
          └── kustomization.yaml
```

```yaml
# myapp-repo/.github/workflows/gitops-deploy.yml
name: GitOps Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-update-manifest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Update staging manifest
        run: |
          git clone https://x-access-token:${{ secrets.MANIFEST_REPO_TOKEN }}@github.com/myorg/myapp-manifests.git
          cd myapp-manifests

          # Update the image tag in staging overlay
          cd overlays/staging
          kustomize edit set image myapp=ghcr.io/${{ github.repository }}:${{ github.sha }}

          git config user.email "ci@mycompany.com"
          git config user.name "CI Bot"
          git add -A
          git commit -m "chore(staging): update myapp to ${{ github.sha }}"
          git push

          echo "✅ Manifest updated — ArgoCD will sync within 3 minutes"
```

ArgoCD ApplicationSet config in the manifests repo:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-manifests
    targetRevision: main
    path: overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Project 4 — Capstone: Production-Grade CI/CD Platform

**Goal:** Build a complete, enterprise-quality CI/CD platform for a microservices application.

**Architecture:**
```
Multiple Services (api, worker, frontend)
    │
    ▼
Monorepo CI with path-based change detection
    │
    ▼
Parallel builds (only changed services)
    │
    ▼
Security scans (SAST + image scan + secret scan)
    │
    ▼
GitOps: update manifests → ArgoCD syncs to staging
    │
    ▼
Automated E2E tests on staging
    │
    ▼
Slack notification → Manual approval
    │
    ▼
GitOps: update manifests → ArgoCD syncs to production
    │
    ▼
Post-deploy: smoke tests + Datadog deployment marker
    │
    ▼
Automatic rollback on smoke test failure
```

**Implementation checklist:**
```
□ Path-based change detection (only build what changed)
□ Reusable workflow for each service build
□ Matrix build for parallel execution
□ OIDC authentication to AWS (no stored credentials)
□ Trivy scan with SARIF upload to GitHub Security tab
□ TruffleHog secret scanning on every PR
□ Image signed with Cosign on main
□ GitOps workflow updating both staging and prod manifests
□ ArgoCD ApplicationSet for all environments
□ Slack notifications (success and failure)
□ Environment protection rules with required reviewers for production
□ Automatic rollback workflow triggered by health check failure
□ Pipeline metrics published to monitoring system
```

**Knowledge Check:**

1. In Project 1, why is the branch protection rule critical? What would happen without it?
2. In Project 2, why does `deploy-production` have `if: github.event_name == 'release'`?
3. Why does GitOps (Project 3) update a manifest repo instead of running `kubectl apply` directly from CI?
4. What is the purpose of the Trivy scan step, and why does it run after `build` instead of before?
5. In the capstone, why is OIDC preferred over storing AWS credentials as GitHub secrets?

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="14-common-mistakes.md">← Previous: Common Mistakes</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="16-interview-preparation.md">Next: Interview Preparation →</a>
</div>
