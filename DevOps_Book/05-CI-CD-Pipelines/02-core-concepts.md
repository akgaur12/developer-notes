# Chapter 2 — Core Concepts & Architecture

## Learning Objectives

By the end of this chapter you will be able to:

- Identify the core primitives of any CI/CD system (pipeline, job, step, runner, artifact)
- Map GitHub Actions, GitLab CI, and Jenkins terminology to each other
- Explain all trigger types and write the YAML for each
- Distinguish artifacts from cache and know when to use each
- Explain why pipeline-as-code matters for production systems
- Describe the security risks that pipelines introduce

---

## 2.1 Pipeline Anatomy

Every CI/CD system — regardless of vendor — is built from the same set of primitives. The names differ, but the concepts are identical. Understanding this helps you switch tools and read documentation from any platform.

| Concept | GitHub Actions | GitLab CI | Jenkins |
|---------|---------------|-----------|---------|
| Configuration file | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` |
| Trigger | Event (push, PR, schedule…) | Pipeline trigger | Webhook / schedule / manual |
| Top-level unit | Workflow | Pipeline | Pipeline |
| Parallel group | Job | Stage | Stage |
| Work unit | Step | Job | Step |
| Execution environment | Runner | GitLab Runner | Agent / Node |
| Reusable config | Action / Reusable Workflow | CI Component / Include | Shared Library |

### Anatomy diagram

```
Workflow / Pipeline
├── Job A  (runs-on: ubuntu-latest)
│   ├── Step 1: checkout code
│   ├── Step 2: install dependencies
│   └── Step 3: run tests
│
└── Job B  (needs: A — runs after A passes)
    ├── Step 1: build Docker image
    └── Step 2: push image to registry
```

Jobs run **in parallel** by default. When you declare a dependency (`needs:` in GitHub Actions), the dependent job waits for its prerequisite to succeed before starting. This creates a Directed Acyclic Graph (DAG) of work.

---

## 2.2 Triggers

A trigger is the event that causes a pipeline to start. Choosing the right trigger for each workflow is one of the most important design decisions in pipeline architecture.

### Trigger types

| Trigger | When to use |
|---------|-------------|
| Push | Run CI on every commit — catch bugs as they land |
| Pull Request / Merge Request | Validate changes before they merge; block bad PRs |
| Schedule (cron) | Nightly builds, weekly security scans, dependency updates |
| Manual (workflow_dispatch) | Deployment approvals, on-demand tasks |
| API / Webhook | Triggered by an external system (another pipeline, a release tool) |
| Tag push | Version releases — trigger build + publish when a tag like `v1.2.3` is pushed |
| Path filter | Only trigger when specific files change (avoid wasting runner time) |

### Complete trigger example

```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
  
  pull_request:
    branches: [main]
  
  schedule:
    - cron: '0 2 * * *'       # daily at 2:00 AM UTC
  
  workflow_dispatch:            # adds a "Run workflow" button in the GitHub UI
    inputs:
      environment:
        description: 'Deployment target'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
  
  push:
    tags:
      - 'v*.*.*'               # trigger on semantic version tags
```

**Path filters** are a performance optimization. If your monorepo contains a frontend and a backend, there is no reason to run the backend's full test suite when only a CSS file changed. Path filters keep pipelines fast and relevant.

---

## 2.3 Runners / Agents

A runner is the machine (or container) that executes a job. The choice of runner affects cost, speed, security, and capability.

### GitHub-hosted runners

| Runner | OS | Notes |
|--------|----|-------|
| `ubuntu-latest` | Ubuntu 24.04 | Most common, fastest, cheapest |
| `ubuntu-22.04` | Ubuntu 22.04 | Use when you need a pinned version |
| `windows-latest` | Windows Server 2022 | For Windows-specific builds |
| `macos-latest` | macOS 14 | For iOS/macOS builds; billed at higher rate |

GitHub-hosted runners provision a **fresh virtual machine for every job**. This guarantees a clean environment and eliminates "it works on the runner" problems — but it also means every job must reinstall dependencies (which is why caching matters so much).

### Self-hosted runners

Your own machine, VM, or container that registers with the CI system and accepts jobs. Useful when:

- You need access to internal network resources
- You need specialized hardware (GPU, specific CPU architecture)
- You need to avoid GitHub's per-minute billing at scale
- Compliance requirements prevent data from leaving your infrastructure

```yaml
runs-on: self-hosted                      # any available self-hosted runner
runs-on: [self-hosted, linux, x64]        # runner matching all these labels
runs-on: [self-hosted, gpu]               # runner with a GPU label
```

### Docker-based runners (GitLab default)

GitLab Runner executes each job inside a Docker container. You specify the image:

```yaml
# .gitlab-ci.yml
test:
  image: node:20-alpine
  script:
    - npm ci
    - npm test
```

This is powerful because you get per-job container isolation without managing VMs.

---

## 2.4 Artifacts and Caching

These are two different mechanisms for storing files across pipeline boundaries. Confusing them is one of the most common beginner mistakes.

### Artifacts

An artifact is a file or directory that a job **produces** and that needs to be:
- Passed to a downstream job in the same pipeline run, **or**
- Stored for later download by a human (test reports, binaries, coverage reports)

Artifacts are scoped to a **single pipeline run**. They do not persist across runs.

Examples:
- A compiled binary that the `build` job produces and the `deploy` job consumes
- A coverage report that a developer wants to download after a test failure
- A JAR file produced by the build that gets deployed to staging

### Cache

A cache is a directory that is saved **between pipeline runs** to avoid repeating expensive work (primarily dependency installation).

Examples:
- `~/.npm` or `node_modules/` — npm packages
- `~/.m2/` — Maven packages
- `~/.cache/pip/` — Python packages

Cache entries are keyed by a hash, typically of the lock file. When the lock file changes (new dependency added), the cache key changes and a fresh install happens. When nothing changed, the cache is restored and installation is instant.

### Comparison

| | Artifact | Cache |
|--|---------|-------|
| Scope | Single pipeline run | Across runs |
| Purpose | Pass output between jobs | Speed up dependency installs |
| Typical content | Build outputs, test reports | Package directories |
| Expires | After N days (configurable) | After N days of no access |

---

## 2.5 Environments and Deployment Protection

In a complete CI/CD system, pipelines deploy through a series of environments:

```
feature branch  →  [CI tests pass]
                         ↓
main branch     →  staging (automatic deploy)
                         ↓
                   production (requires approval)
```

GitHub Actions, GitLab CI, and Jenkins all support **environment protection rules**:

- **Required reviewers**: specific people or teams must approve before the deploy job runs
- **Wait timer**: introduce a delay (e.g., 10 minutes) between approval and actual deployment, giving time to abort
- **Branch restriction**: only the `main` branch can deploy to production (prevents accidental deploys from feature branches)

Environment-scoped secrets and variables let you keep `PRODUCTION_DB_URL` separate from `STAGING_DB_URL` without extra logic in your pipeline code.

---

## 2.6 Pipeline as Code

One of the most important principles in modern CI/CD is that **pipeline configuration lives in the repository alongside the application code**.

This means:
- Pipeline changes go through the same PR review process as code changes
- You can use `git log` to see who changed the pipeline and why
- Rolling back to a previous application version also rolls back to the pipeline configuration from that version
- New developers can read the pipeline config to understand exactly how the project is built and deployed

The configuration files are:

| Tool | File location |
|------|--------------|
| GitHub Actions | `.github/workflows/*.yml` |
| GitLab CI | `.gitlab-ci.yml` (or included files) |
| Jenkins | `Jenkinsfile` (at repo root) |

Treat your pipeline configuration with the same care as your application code — review it, test changes in branches, and never make unreviewed changes directly to production pipeline config.

---

## 2.7 Idempotency and Immutability

Two properties that separate robust pipelines from fragile ones:

### Idempotent pipeline

Running the same pipeline twice with the same inputs produces the same result. Idempotent pipelines are safe to retry — if a network blip causes a step to fail, you can just rerun it without fear.

Non-idempotent antipattern: a deploy step that blindly applies a migration every time it runs, even if the migration already ran. This breaks on the second run.

### Immutable artifact

Once an artifact (particularly a Docker image) is built and tagged, it never changes. The same tag always refers to the same binary.

Antipattern: pushing to `:latest` continuously. If a deployment goes wrong, you cannot roll back because `:latest` has already been overwritten. Instead, use content-addressable tags:

```bash
docker build -t myapp:$GIT_SHA .
docker push myapp:$GIT_SHA
```

Now you can always deploy (or roll back to) any exact commit.

---

## 2.8 The Feedback Loop

The entire value of CI depends on feedback being **fast**. A pipeline that takes 45 minutes to run trains developers to push their code and context-switch — by the time the result comes back, they have forgotten what they were working on.

Guidelines for feedback loop targets:

| Stage | Target time |
|-------|-------------|
| Linting + unit tests | < 3 minutes |
| Full test suite | < 10 minutes |
| Build + integration tests | < 15 minutes |
| Full pipeline (all the way to staging) | < 30 minutes |

**Optimization strategies:**

- **Parallelism**: run independent jobs simultaneously (lint, test, security scan all at once)
- **Caching**: restore dependencies from cache instead of reinstalling
- **Fail fast**: if any required check fails, cancel the remaining jobs immediately
- **Path filtering**: skip jobs that are irrelevant to the changed files
- **Test splitting**: distribute test suites across multiple runners

The goal is a pipeline that developers trust, run frequently, and act on immediately when it fails.

---

## 2.9 Security in the Pipeline

CI/CD pipelines are high-value targets for attackers. A compromised pipeline has access to:

- Production credentials and secrets
- The ability to push arbitrary code to your Docker registry
- Deployment access to your production infrastructure

### Supply chain attacks on CI/CD

Several high-profile attacks have targeted CI/CD systems specifically:

- **SolarWinds (2020)**: attackers inserted malicious code into the build pipeline that was then shipped in signed software updates
- **CodeCov (2021)**: a compromised CI tool was used to exfiltrate environment variables (secrets) from thousands of CI runs

### Defensive principles

**Principle of least privilege**: each job should have the minimum permissions it needs. A test job does not need write access to your container registry.

```yaml
permissions:
  contents: read        # only read access to the repo
  packages: write       # write access to GHCR (only for the build job)
```

**Pin action versions to SHA**, not just to a tag. Tags can be moved; a SHA hash is immutable:

```yaml
# Vulnerable: tag can be moved to point to malicious code
- uses: actions/checkout@v4

# Safe: this exact SHA cannot change
- uses: actions/checkout@1f9a0c22da41e6ebfa534300ef656657ea2c6707
```

**Never print secrets in logs**: environment variables set from secrets are masked automatically by GitHub, but avoid patterns like `echo $SECRET` or piping secrets through tools that might log them.

**Audit third-party actions**: before adding an action from the marketplace, review its source code and check its permissions.

---

## Summary

- Every CI/CD system has the same primitives: triggers, pipelines, jobs, steps, runners — only the names differ
- Triggers determine when a pipeline runs; choose the right trigger to balance coverage and speed
- Runners can be cloud-hosted (fresh VM per job) or self-hosted (your own infrastructure)
- Artifacts pass outputs between jobs in one run; cache persists dependency directories across runs
- Pipeline configuration lives in version control alongside the application code
- Immutable artifacts (tagged with git SHA) enable reliable rollbacks
- Keep pipelines fast (< 10 min for tests) or developers stop trusting them
- Pipelines have access to production secrets — apply least-privilege, pin dependencies, and audit third-party actions

---

## Knowledge Check

1. In a GitHub Actions workflow, what does `needs: [test, lint]` on a job do?
2. What is the difference between an artifact and a cache? Give a concrete example of each.
3. Why is it safer to pin an action to a commit SHA than to a version tag?
4. A developer says the test suite takes 40 minutes. Name three concrete strategies to bring it under 10 minutes.
5. What is an idempotent pipeline? Why does idempotency matter for deployments?

---

## Hands-On Exercise

Choose a real project you are familiar with (or a public repository on GitHub). On paper or in a diagram tool:

1. List every step that happens when a change goes from a developer's commit to running in production
2. Group those steps into pipeline jobs
3. Draw the dependency graph — which jobs must wait for others, and which can run in parallel?
4. Identify what artifacts (if any) pass between jobs
5. Identify what secrets the pipeline would need and which jobs need them

This exercise forces you to think about pipeline design before you write a single line of YAML — and it is exactly the kind of whiteboard question interviewers ask for senior DevOps roles.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./01-introduction.md">← Previous: Introduction to CI/CD</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./03-github-actions-fundamentals.md">Next: GitHub Actions Fundamentals →</a>
</div>
