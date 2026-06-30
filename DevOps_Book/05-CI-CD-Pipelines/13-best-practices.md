# Chapter 13 — Best Practices

## Learning Objectives

By the end of this chapter, you will be able to:

- Design pipelines that follow the single-responsibility and fail-fast principles
- Structure branch and trigger strategies for safe, efficient deployments
- Apply the Golden Pipeline pattern to your own projects
- Manage secrets securely using OIDC and environment scoping
- Optimize Docker image workflows for speed and immutability
- Organize workflow files for maintainability and reuse
- Set and enforce performance targets for each pipeline stage
- Build rollback readiness into every production deployment

---

## 13.1 Pipeline Design Principles

Good pipeline design borrows from good software design. These five principles apply universally across tools and teams.

**Single responsibility** — each job does one thing. Test, build, and deploy should be separate jobs, not one long script. This makes failures easier to diagnose and jobs easier to rerun independently.

**Fail fast** — put quick checks (lint, type-check) before slow ones (integration tests, builds). A developer should know their PR has a lint error in 90 seconds, not 12 minutes.

**Idempotent deployments** — running the pipeline twice should produce the same result. No manual state that drifts. No "it works if you only run it once."

**Immutable artifacts** — build once, promote everywhere. Never rebuild the same commit for staging vs. production. The image that passed staging tests is the exact image that goes to production.

**Everything as code** — pipeline config, infrastructure definitions, deployment scripts — all in git. No click-ops in the CI UI that isn't captured somewhere reviewable.

---

## 13.2 Branch and Trigger Strategy

A well-structured trigger strategy prevents wasted runs and ensures the right work happens at the right time.

```yaml
# Recommended trigger strategy
on:
  push:
    branches: [main]            # full CI + deploy to staging
  pull_request:
    branches: [main]            # CI only (no deploy)
  release:
    types: [published]          # deploy to production on release tag
  workflow_dispatch:            # emergency manual deploy
    inputs:
      environment: { type: choice, options: [staging, production] }
```

Branch rules to enforce alongside this:

- `main` is always deployable and protected — no direct pushes
- Feature branches run CI only; they never trigger a deploy
- All changes to `main` go through a pull request with passing CI
- `workflow_dispatch` provides an escape hatch for emergency manual deploys without bypassing protection rules

---

## 13.3 The Golden Pipeline

This is the canonical pipeline order. Each gate blocks the next — a failure stops progression.

```
Trigger (push/PR)
    │
    ▼
1. Lint + Type Check (fast, <2 min)
    │
    ▼
2. Unit Tests + Coverage (fast, <5 min)
    │
    ▼
3. Security Scan (SAST, dependency audit) (<3 min)
    │
    ▼
4. Build Artifact / Docker Image (<5 min)
    │
    ▼
5. Integration Tests (with real DB, cache) (<10 min)
    │
    ▼
6. Image Vulnerability Scan (Trivy) (<2 min)
    │
    ▼
7. Deploy to Staging (auto)
    │
    ▼
8. Smoke Tests on Staging
    │
    ▼
9. [Human approval] → Deploy to Production
    │
    ▼
10. Post-deploy Health Check + Rollback on failure
```

**Time targets:** under 15 minutes to staging, under 20 minutes to production (including approval wait time for the deploy trigger, not counting human deliberation).

The ordering matters. Security scanning happens before the image build so you catch dependency issues before wasting build time. Integration tests run after the image is built so they test the actual artifact. Smoke tests on staging give you real-environment confidence before the production gate.

---

## 13.4 Secrets Best Practices

```
□ Use OIDC instead of long-lived credentials wherever possible
□ Scope secrets to environments (staging secrets ≠ production secrets)
□ Scan for accidentally committed secrets (TruffleHog in CI)
□ Rotate secrets on a schedule; auto-rotate when possible
□ Mask dynamically fetched secrets: echo "::add-mask::$SECRET"
□ Never log secrets; never pass them as URL parameters
□ Use Vault or cloud secrets managers for complex setups
□ Require PR review before changes to production secrets
```

OIDC (OpenID Connect) is the most important item on this list. With OIDC, your workflow requests a short-lived token from GitHub that your cloud provider (AWS, GCP, Azure) trusts directly — no stored secret, no rotation needed, no risk of credential leak. Set it up once per cloud account and never store a cloud credential in GitHub Secrets again.

Environment-scoped secrets are the second most impactful practice. GitHub Environments let you define secrets that only apply when deploying to `production`. A compromised workflow running on a PR cannot access production credentials.

---

## 13.5 Docker Image Best Practices in CI

```
□ Tag images with git SHA (immutable) + branch/version (mutable)
□ Never overwrite a pushed SHA tag
□ Scan images with Trivy — fail on CRITICAL in CI
□ Sign images with Cosign on main branch
□ Use BuildKit GHA cache for fast rebuilds
□ Multi-stage builds: never include build tools in final image
□ Push to private registry; use lifecycle policies to clean old images
□ Promote images (retag) rather than rebuilding for deployment
```

Tagging strategy in practice:

```bash
# Build and push with both tags
docker build -t myapp:$GITHUB_SHA -t myapp:latest .
docker push myapp:$GITHUB_SHA   # immutable — never changes
docker push myapp:latest        # mutable — points to newest

# Promote to production (no rebuild)
docker tag myapp:$GITHUB_SHA myapp:prod-$GITHUB_SHA
docker push myapp:prod-$GITHUB_SHA
```

BuildKit cache example:

```yaml
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## 13.6 Testing Best Practices

```
□ Unit tests: run on every commit, < 5 minutes, cover critical paths
□ Integration tests: use service containers (not mocks) for DB/cache
□ Use fail-fast: true for unit tests (fast feedback), false for matrix builds
□ Upload test results as artifacts — always, including on failure
□ Enforce coverage thresholds — fail if coverage drops below baseline
□ Run E2E tests on staging after deploy, not in CI (too slow/flaky)
□ Flaky tests: track and fix immediately; flaky CI is ignored CI
```

Service containers for integration tests:

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

Always upload test results, even on failure — that is when you need them most:

```yaml
- name: Upload test results
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: test-results
    path: test-results/
```

---

## 13.7 Workflow File Organization

A well-organized `.github/` directory scales to a large team without becoming a mess.

```
.github/
├── workflows/
│   ├── ci.yml            # unit tests, lint, type-check (every PR)
│   ├── build.yml         # Docker build and push (main only)
│   ├── deploy.yml        # deployment workflow (called by build or manual)
│   ├── security.yml      # scheduled security scans
│   └── release.yml       # version tagging and changelog
├── actions/
│   ├── setup-env/        # composite action: checkout + setup tools
│   │   └── action.yml
│   └── notify-slack/     # composite action: slack notification
│       └── action.yml
└── dependabot.yml        # automated dependency updates
```

The pattern here is: workflows orchestrate, composite actions provide reusable steps. If you find yourself copying the same 5-step setup block across three workflow files, extract it into a composite action.

---

## 13.8 Performance Targets

| Stage | Target | Red Flag |
|-------|--------|----------|
| Lint + type check | < 2 min | > 5 min |
| Unit tests | < 5 min | > 10 min |
| Docker build (cached) | < 3 min | > 8 min |
| Docker build (cold) | < 8 min | > 15 min |
| Integration tests | < 10 min | > 20 min |
| Full pipeline (PR) | < 10 min | > 20 min |
| Deploy to staging | < 5 min | > 15 min |

When a stage hits the red flag threshold, it becomes a bottleneck that developers route around — they start skipping CI, pushing hotfixes directly, or merging without waiting for green. Speed is a correctness property of CI.

Common fixes for slow pipelines:

- Add dependency caching (most impactful, often 2-3x speedup)
- Parallelize independent jobs
- Split slow test suites across a matrix
- Move E2E tests out of CI and onto post-deploy
- Use `paths` filters to skip jobs when unrelated files change

---

## 13.9 Rollback Readiness

Every production deployment must have a tested rollback path. This is non-negotiable.

```yaml
deploy-prod:
  steps:
    - name: Deploy
      id: deploy
      run: kubectl set image deployment/myapp app=myapp:${{ github.sha }}

    - name: Wait for rollout
      run: kubectl rollout status deployment/myapp --timeout=5m

    - name: Smoke test
      run: ./smoke-test.sh https://api.myapp.com

    - name: Rollback on failure
      if: failure()
      run: |
        kubectl rollout undo deployment/myapp
        echo "Deployment failed — rolled back to previous version"
        exit 1   # mark job as failed even after rollback
```

The `exit 1` at the end of the rollback step is intentional. A successful rollback is still a failed deployment — the pipeline should be red so the team knows something needs attention.

Test your rollback procedure at least quarterly. A rollback plan you have never executed is not a rollback plan.

---

## Summary

- **Pipeline design principles**: single responsibility, fail fast, idempotent, immutable artifacts, everything as code
- **Trigger strategy**: full CI+deploy on `main`, CI only on PRs, production deploy on release tags
- **The Golden Pipeline**: 10-stage sequence from lint to post-deploy health check, targeting under 20 minutes end-to-end
- **Secrets**: prefer OIDC over stored credentials; scope secrets to environments; scan for accidental commits
- **Docker**: tag with git SHA, scan with Trivy, sign with Cosign, use BuildKit cache, never rebuild for promotion
- **Testing**: unit tests fast and always, integration tests with real service containers, E2E tests post-deploy
- **Organization**: separate workflow files by concern, extract reusable logic into composite actions
- **Performance targets**: lint < 2 min, unit tests < 5 min, full PR pipeline < 10 min
- **Rollback**: automated rollback on failure, test the procedure quarterly

---

## Knowledge Check

1. What is the difference between an immutable artifact tag and a mutable tag? Give an example of each.
2. Why should lint and type-checking run before unit tests in the pipeline order?
3. What is OIDC, and why is it preferable to storing cloud credentials as GitHub Secrets?
4. What does `if: always()` do in a GitHub Actions step, and when should you use it?
5. Why should the rollback step end with `exit 1` even if the rollback succeeds?

---

## Hands-on Exercise

**Pipeline Audit**

Audit your current pipeline against the Golden Pipeline and Best Practices checklists in this chapter.

1. Map your existing pipeline stages to the 10-stage Golden Pipeline. Which stages are missing?
2. Check each of the four checklists (Secrets, Docker, Testing, Organization). Mark each item as passing, failing, or not applicable.
3. Measure the current runtime of each stage. Compare against the performance targets table.
4. Identify the top 3 gaps — prioritize by impact (a missing rollback plan outweighs a missing Cosign signature).
5. Implement fixes for those 3 gaps. Document the before/after pipeline runtime.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="12-advanced-concepts.md">← Previous: Advanced Concepts</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="14-common-mistakes.md">Next: Common Mistakes →</a>
</div>
