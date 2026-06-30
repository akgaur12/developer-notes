# Chapter 17 — Course Summary

## You've Completed CI/CD Pipelines!

---

## What You Learned

### Foundations (Chapters 01–03)
- **CI/CD concepts**: CI = automate test; Continuous Delivery = always deployable; Continuous Deployment = always deployed; DORA metrics
- **Pipeline anatomy**: triggers, jobs, steps, runners, artifacts, cache — same concepts across all tools
- **GitHub Actions fundamentals**: workflow YAML structure, runners, actions, secrets, job dependencies, caching, artifacts, conditionals

### Core Skills (Chapters 04–06)
- **GitHub Actions advanced**: matrix builds, reusable workflows, composite actions, OIDC, concurrency, self-hosted runners, SHA pinning
- **Docker in CI/CD**: BuildKit GHA cache, multi-registry push, metadata-action, Trivy scanning, multi-stage test → build pipeline, image promotion
- **Testing in CI**: test pyramid, service containers for integration tests, test reporting, code quality gates, branch protection rules

### Infrastructure (Chapters 07–09)
- **Deployment strategies**: rolling (kubernetes default), blue/green (instant switch), canary (gradual rollout), feature flags, expand/contract DB migrations, rollback patterns
- **GitLab CI**: `.gitlab-ci.yml`, Docker-in-Docker, predefined variables, rules, environments, comparison with GitHub Actions
- **Jenkins**: Declarative Jenkinsfile, credentials, parallel stages, Docker agents, shared libraries

### Advanced (Chapters 10–12)
- **Secrets management**: GitHub/GitLab secrets, Vault + OIDC, AWS Secrets Manager, secret scanning, masking, rotation
- **Intermediate concepts**: caching strategies, job outputs, dynamic matrix, monorepo path detection, notifications, performance optimization
- **Advanced concepts**: GitOps with ArgoCD/Flux, Argo Rollouts for progressive delivery, Terraform in CI, custom actions, compliance/audit requirements

---

## Completion Checklist

### Beginner
```
□ Write a GitHub Actions workflow that runs tests on every PR
□ Add dependency caching to reduce pipeline time
□ Upload test artifacts and browse them in GitHub UI
□ Set up branch protection requiring CI to pass before merge
□ Understand what each key in a workflow file does
```

### Intermediate
```
□ Build a Docker image in CI and push to GHCR
□ Run Trivy vulnerability scan, interpret the output
□ Use GitHub environments with protection rules
□ Configure OIDC authentication to AWS (no stored credentials)
□ Implement a blue/green or canary deployment in CI
□ Write a reusable workflow used by multiple services
□ Add integration tests using service containers
```

### Advanced
```
□ Implement change-detection for a monorepo (only build what changed)
□ Set up a GitOps workflow — CI updates manifests, ArgoCD deploys
□ Build a Jenkins pipeline from scratch with a Jenkinsfile
□ Configure a GitLab CI pipeline with DinD image builds
□ Add secret scanning (TruffleHog) and SAST (CodeQL) to CI
□ Implement automatic rollback on post-deploy smoke test failure
□ Publish pipeline metrics to a monitoring system
```

---

## Key Commands Reference

```bash
# ─── GitHub CLI ────────────────────────────────────────────────────
gh workflow list                           # list workflows
gh workflow run ci.yml                     # trigger manually
gh run list --workflow=ci.yml             # recent runs
gh run view <run-id>                       # run details
gh run cancel <run-id>                     # cancel run
gh run rerun <run-id> --failed            # rerun failed jobs only
gh secret set MY_SECRET                    # set repository secret
gh secret list                             # list secrets

# ─── Docker in CI ──────────────────────────────────────────────────
docker build -t myapp:$SHA .
docker push ghcr.io/myorg/myapp:$SHA
trivy image ghcr.io/myorg/myapp:$SHA

# ─── Kubernetes Rollout ────────────────────────────────────────────
kubectl set image deployment/myapp app=myapp:$SHA
kubectl rollout status deployment/myapp --timeout=5m
kubectl rollout undo deployment/myapp
kubectl rollout history deployment/myapp

# ─── ArgoCD ────────────────────────────────────────────────────────
argocd app sync myapp-staging
argocd app get myapp-staging
argocd app rollback myapp-staging 3
argocd app history myapp-staging

# ─── Jenkins CLI ───────────────────────────────────────────────────
java -jar jenkins-cli.jar -s http://localhost:8080 build my-pipeline
java -jar jenkins-cli.jar -s http://localhost:8080 console my-pipeline 42
```

---

## The CI/CD Maturity Model

| Level | Characteristics |
|-------|----------------|
| 0 — Manual | No automation. Deploy = SSH + git pull. |
| 1 — Basic CI | Tests run on push. PRs blocked if tests fail. |
| 2 — CI + CD | Automated builds, Docker images, auto-deploy to staging. |
| 3 — Full CD | Production deploys automated with approval gates and rollback. |
| 4 — Advanced | GitOps, progressive delivery, compliance, metrics, self-healing. |

---

## What's Next

You've completed Topic 5: CI/CD Pipelines. The next topic in the DevOps roadmap is:

**Topic 6: Cloud Fundamentals (AWS)**
- EC2, VPCs, security groups, and IAM
- S3 storage and lifecycle policies
- RDS and ElastiCache
- CloudFront CDN and Route53
- AWS CLI and infrastructure automation
- Cost optimization and monitoring

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="16-interview-preparation.md">← Previous: Interview Preparation</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="../06-Cloud-Fundamentals-AWS/00-index.md">Next: Cloud Fundamentals (AWS) →</a>
</div>
