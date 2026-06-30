# Chapter 8 — GitLab CI

## Learning Objectives

By the end of this chapter you will be able to:

- Describe GitLab CI's architecture and how runners execute jobs
- Write a `.gitlab-ci.yml` file with stages, jobs, caching, and artifacts
- Use predefined CI variables for image tagging and conditional logic
- Compare GitLab CI with GitHub Actions and choose between them

---

## 8.1 GitLab CI Architecture

GitLab CI is built into the GitLab platform — no third-party service required.

```
GitLab Server
    │
    ├── stores .gitlab-ci.yml
    ├── triggers pipelines on push / MR / schedule
    └── presents pipeline UI

GitLab Runner (separate process)
    │
    ├── polls GitLab for pending jobs
    ├── picks up a job, executes it
    └── reports logs and exit code back to GitLab

Executors (how the runner runs a job):
    ├── docker     — most common; spins up a container per job
    ├── shell      — runs directly on the runner host
    ├── kubernetes — creates a pod per job
    └── virtualbox — full VM per job
```

**Key differences from GitHub Actions:**

- The pipeline config (`.gitlab-ci.yml`) lives in the repo root
- Stages are globally declared and jobs are assigned to them — ordering is explicit
- GitLab includes a built-in container registry, making image push/pull seamless
- Self-hosted runners are free and straightforward to set up

---

## 8.2 `.gitlab-ci.yml` Structure

```yaml
# Global defaults — applied to every job unless overridden
default:
  image: node:20-alpine
  before_script:
    - npm ci

# Global variables available to all jobs
variables:
  NODE_ENV: test
  DOCKER_DRIVER: overlay2

# Stages define execution order
# Jobs in the same stage run in parallel
# Jobs in later stages only start after the previous stage succeeds
stages:
  - test
  - build
  - staging
  - production

# Jobs
test:unit:
  stage: test
  script:
    - npm run test:unit
  coverage: '/Statements\s*:\s*(\d+\.\d+)%/'   # regex to parse coverage %
  artifacts:
    reports:
      junit: junit.xml       # parsed and shown inline in merge requests
    paths:
      - coverage/
    expire_in: 1 week

test:integration:
  stage: test
  services:
    - name: postgres:15-alpine
      alias: db              # reachable as hostname "db" inside the job
  variables:
    POSTGRES_DB: testdb
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://postgres:test@db/testdb
  script:
    - npm run test:integration
```

**How stages work:**

- `test:unit` and `test:integration` are both in the `test` stage, so they run in parallel
- The `build` stage only starts after both test jobs succeed
- A job failure in any stage cancels all subsequent stages by default

---

## 8.3 Docker-in-Docker (DinD) for Building Images

Building Docker images inside a CI job requires Docker-in-Docker:

```yaml
build:image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind         # sidecar that provides the Docker daemon
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - |
      if [ "$CI_COMMIT_BRANCH" = "main" ]; then
        docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
        docker push $CI_REGISTRY_IMAGE:latest
      fi
```

**What the variables mean here:**

- `$CI_REGISTRY` — `registry.gitlab.com`
- `$CI_REGISTRY_IMAGE` — `registry.gitlab.com/group/project`
- `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` — auto-generated, scoped to this pipeline job; no manual setup needed

**Alternative: Kaniko (no Docker daemon required)**

```yaml
build:image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.18.0-debug
    entrypoint: [""]
  script:
    - /kaniko/executor
        --context $CI_PROJECT_DIR
        --dockerfile $CI_PROJECT_DIR/Dockerfile
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

Kaniko builds images without a privileged Docker daemon — preferred in locked-down Kubernetes environments.

---

## 8.4 GitLab Predefined Variables

GitLab injects these variables into every job automatically:

| Variable | Example value | Description |
|---|---|---|
| `$CI_COMMIT_SHA` | `a1b2c3d4...` | Full commit SHA |
| `$CI_COMMIT_SHORT_SHA` | `a1b2c3d4` | First 8 characters |
| `$CI_COMMIT_BRANCH` | `main` | Branch name |
| `$CI_COMMIT_REF_SLUG` | `feature-my-thing` | URL-safe branch (slashes → dashes) |
| `$CI_COMMIT_TAG` | `v1.2.3` | Tag name (only set on tag pipelines) |
| `$CI_PIPELINE_ID` | `12345` | Unique pipeline ID |
| `$CI_JOB_ID` | `67890` | Unique job ID |
| `$CI_REGISTRY` | `registry.gitlab.com` | Container registry hostname |
| `$CI_REGISTRY_IMAGE` | `registry.gitlab.com/org/repo` | Full image path for this project |
| `$CI_REGISTRY_USER` | `gitlab-ci-token` | Auto-generated registry username |
| `$CI_REGISTRY_PASSWORD` | `<token>` | Auto-generated registry password |
| `$CI_PROJECT_NAME` | `myapp` | Repository name |
| `$CI_PROJECT_DIR` | `/builds/org/myapp` | Path to checked-out repo |
| `$CI_ENVIRONMENT_NAME` | `staging` | Current environment (if set) |
| `$CI_MERGE_REQUEST_IID` | `42` | MR number (only in MR pipelines) |

Use these to build image tags, construct URLs, and control conditional logic.

---

## 8.5 Rules and Conditions

`rules:` controls when a job runs. It replaces the older `only:` / `except:` syntax.

```yaml
deploy:staging:
  stage: staging
  script: ./deploy.sh staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success
    - when: never     # skip for all other branches

deploy:production:
  stage: production
  script: ./deploy.sh production
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/    # only semver tags
  when: manual        # job appears in UI but requires a human to click Play
  environment:
    name: production
    url: https://myapp.com

# Only run on merge requests (not on branch push)
test:mr-only:
  stage: test
  rules:
    - if: $CI_MERGE_REQUEST_ID
  script: ./test-mr-specific.sh
```

**Common `rules` patterns:**

```yaml
# Run on main branch and all MRs, but not on other branches
rules:
  - if: $CI_COMMIT_BRANCH == "main"
  - if: $CI_MERGE_REQUEST_ID
  - when: never

# Skip if a variable is set (e.g. to skip expensive jobs manually)
rules:
  - if: $SKIP_TESTS == "true"
    when: never
  - when: on_success
```

---

## 8.6 Caching and Artifacts

**Cache** — persists files between pipeline runs (e.g., `node_modules`). Shared across jobs.

**Artifacts** — files produced by a job and passed to later jobs in the same pipeline. Also downloadable from the UI.

```yaml
test:
  cache:
    key:
      files:
        - package-lock.json    # cache key is the hash of this file
    paths:
      - node_modules/
    policy: pull-push          # restore cache at start, update at end

  artifacts:
    paths:
      - dist/                  # pass build output to later stages
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    expire_in: 1 day
```

**Passing artifacts between stages:**

```yaml
build:
  stage: build
  script: npm run build
  artifacts:
    paths:
      - dist/

deploy:
  stage: deploy
  script:
    - ls dist/       # artifacts from build job are automatically available
    - ./deploy.sh dist/
```

By default, artifacts from all previous stages are downloaded into the current job. Use `dependencies: []` to opt out.

---

## 8.7 Environments and Deployments

Environments give you a deployment history and live URL per environment in the GitLab UI:

```yaml
deploy:staging:
  stage: staging
  script: ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.myapp.com
    on_stop: stop:staging      # job to run when environment is stopped

stop:staging:
  stage: staging
  script: ./undeploy.sh staging
  environment:
    name: staging
    action: stop               # marks this as the cleanup job
  when: manual
  rules:
    - if: $CI_MERGE_REQUEST_ID  # available on MR pipelines

deploy:production:
  stage: production
  script: ./deploy.sh production
  environment:
    name: production
    url: https://myapp.com
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

The GitLab **Environments** page shows the current deployed version for each environment, with links to the deployment pipeline and rollback controls.

---

## 8.8 GitLab CI vs GitHub Actions

| Feature | GitLab CI | GitHub Actions |
|---|---|---|
| Config file | `.gitlab-ci.yml` (one file) | `.github/workflows/*.yml` (multiple files) |
| Job ordering | Explicit `stages:` list | `needs:` dependency graph |
| Runner | GitLab Runner (Docker executor default) | GitHub-hosted runners |
| Self-hosted | Free, easy to add | Free, requires setup |
| Container registry | Built-in, same domain | GHCR (separate service) |
| Marketplace | No official marketplace | 10,000+ community actions |
| OIDC to cloud | Supported | Supported |
| Auto DevOps | Yes (language auto-detection) | No |
| Integrated platform | Repo + CI + registry + issues + wiki | Repo + CI (issues/etc. separate concerns) |
| Free minutes (SaaS) | 400 min/month | 2,000 min/month |

**When to choose GitLab CI:**

- You want an all-in-one platform (code, CI, registry, issues) on self-hosted infrastructure
- Your organization already uses GitLab
- You want unlimited CI minutes on your own runners without GitHub Enterprise

**When to choose GitHub Actions:**

- Your code is already on GitHub
- You want access to the large ecosystem of community actions
- You prefer the larger free-tier minute allowance

---

## 8.9 Auto DevOps

GitLab can auto-generate a complete CI/CD pipeline by detecting the project's language — no `.gitlab-ci.yml` needed to get started.

What Auto DevOps provides automatically:

- Build (Dockerfile detection or buildpacks)
- Unit tests
- Static Application Security Testing (SAST)
- Dependency scanning
- Container image scanning
- Deploy to Kubernetes (if configured)
- Performance testing

**To enable:** Settings → CI/CD → Auto DevOps → Enable

Auto DevOps is a good starting point. Once your team has specific requirements — custom test commands, non-standard deploy scripts, performance tuning — write an explicit `.gitlab-ci.yml` that gives you full control.

---

## 8.10 GitLab Runner Setup

```bash
# Install GitLab Runner on Ubuntu/Debian
curl -L \
  https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh \
  | sudo bash
sudo apt-get install gitlab-runner

# Register the runner with GitLab
# (get the registration token from: Settings → CI/CD → Runners)
sudo gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token $REGISTRATION_TOKEN \
  --executor docker \
  --docker-image alpine:latest \
  --description "my-docker-runner" \
  --tag-list docker,linux
```

The runner configuration is stored at `/etc/gitlab-runner/config.toml`. After registration:

```bash
# Start the runner service
sudo systemctl start gitlab-runner
sudo systemctl enable gitlab-runner

# Check status
sudo gitlab-runner status

# View logs
sudo journalctl -u gitlab-runner -f
```

**Runner tags:** Jobs can specify `tags:` to target specific runners. Use tags to route jobs to runners with specific capabilities (GPU, high-memory, specific OS).

---

## Summary

- GitLab CI configuration lives in `.gitlab-ci.yml` at the repo root; `stages:` declares order, jobs are assigned to stages
- GitLab Runner executes jobs; the Docker executor is the most common and provides clean isolated environments
- Predefined variables like `$CI_REGISTRY_IMAGE` and `$CI_COMMIT_SHA` make image tagging straightforward
- `rules:` controls when jobs run — prefer it over the older `only:` / `except:` syntax
- `cache:` persists data between pipeline runs; `artifacts:` passes files between jobs within a pipeline
- Environments give you deployment history, live URLs, and rollback controls in the GitLab UI
- GitLab CI and GitHub Actions are both strong choices; the right pick depends on where your code lives and what platform features matter to your team

---

## Knowledge Check

1. What is a GitLab Runner executor, and what is the difference between the `docker` and `shell` executors?
2. You have three jobs in the `test` stage. How does GitLab determine when to start the `build` stage?
3. What is the difference between `cache:` and `artifacts:` in a `.gitlab-ci.yml`? When would you use each?
4. Write a `rules:` block that runs a `deploy:production` job only when a Git tag matching the pattern `v1.2.3` is pushed, and requires a human to click Play.
5. What is Docker-in-Docker (DinD) and why is it needed for building container images inside a GitLab CI job?

---

## Hands-on Exercise

**Goal:** Create a working `.gitlab-ci.yml` with test, build, and deploy stages.

**Steps:**

1. Create a free GitLab account (or use a local GitLab instance via `docker run gitlab/gitlab-ce`).
2. Push a simple Node.js or Python project to a GitLab repository.
3. Write a `.gitlab-ci.yml` that:
   - Stage 1 (`test`): runs unit tests and publishes a JUnit artifact
   - Stage 2 (`build`): builds a Docker image and pushes it to the GitLab container registry using `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA`
   - Stage 3 (`deploy`): a manual job that echoes the image tag that would be deployed (simulating a deploy script)
4. Add a `cache:` block keyed on the lockfile hash to avoid re-downloading dependencies on every run.
5. Add a `rules:` block so the `deploy` job only appears on the `main` branch.

**Stretch goal:** Compare the pipeline run time and UX with an equivalent GitHub Actions workflow you have written previously.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="07-deployment-strategies.md">← Previous: Deployment Strategies</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="09-jenkins.md">Next: Jenkins →</a>
</div>
