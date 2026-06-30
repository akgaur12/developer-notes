# Chapter 4 — GitHub Actions Advanced

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure matrix builds to run jobs across multiple configurations in parallel
- Create reusable workflows and composite actions to eliminate repetition
- Set up deployment environments with protection rules and approvals
- Implement keyless OIDC authentication to cloud providers
- Control concurrency to prevent conflicting pipeline runs
- Manage self-hosted runners securely
- Apply least-privilege permissions to workflows
- Pin actions to immutable SHAs to prevent supply chain attacks
- Debug failing workflows using built-in and manual techniques

---

## 4.1 Matrix Builds

Matrix builds let you run the same job across multiple configurations simultaneously. Instead of copy-pasting jobs, you declare a matrix and GitHub Actions fans out automatically.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 21]
        os: [ubuntu-latest, windows-latest]
      fail-fast: false        # don't cancel all if one fails
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test
```

This produces 3 × 2 = 6 parallel jobs. Setting `fail-fast: false` lets all jobs run to completion even if one fails — useful when you want a full picture of which configurations are broken.

### Include and Exclude Patterns

You can add extra variables to specific combinations with `include`, or skip certain combinations with `exclude`:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node-version: [18, 20]
    include:
      - os: ubuntu-latest
        node-version: 22
        experimental: true    # extra variable added to this combination only
    exclude:
      - os: windows-latest
        node-version: 18      # skip this specific combo
```

---

## 4.2 Reusable Workflows

Reusable workflows let you define a workflow once and call it from many other workflows — similar to a function call. This is ideal for shared build, test, or deploy logic.

**Define the reusable workflow:**

```yaml
# .github/workflows/reusable-build.yml
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
    secrets:
      registry-token:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t ${{ inputs.image-name }}:${{ github.sha }} .
```

**Call it from another workflow:**

```yaml
# .github/workflows/main.yml
jobs:
  build-api:
    uses: ./.github/workflows/reusable-build.yml
    with:
      image-name: myapi
    secrets:
      registry-token: ${{ secrets.GITHUB_TOKEN }}
```

Reusable workflows run as a separate workflow context. They appear as a nested workflow in the Actions UI, and each called workflow can have its own jobs.

---

## 4.3 Composite Actions

Composite actions package multiple steps into a reusable action that lives inside your repository. Unlike reusable workflows, composite actions can be used as a single `step` inside a job.

**Define the composite action:**

```yaml
# .github/actions/setup-and-cache/action.yml
name: Setup and Cache
description: Setup Node.js with npm cache
inputs:
  node-version:
    description: Node.js version
    default: '20'
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash
```

**Use it in a workflow:**

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-and-cache
    with:
      node-version: '20'
  - run: npm test
```

Composite actions require `shell:` to be specified on every `run` step inside them. They cannot use `secrets` directly — pass secrets through inputs if needed.

---

## 4.4 Environments and Deployment Protection

GitHub Environments let you add approval gates, wait timers, and branch restrictions before deployments proceed.

```yaml
jobs:
  deploy-staging:
    environment: staging       # references GitHub Environment settings
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging

  deploy-prod:
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com   # shown in GitHub UI as a clickable link
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production
```

**Configure in GitHub:** Settings → Environments → select or create an environment, then configure:

- **Required reviewers**: one or more people must approve before the job runs
- **Wait timer**: delay the deployment by N minutes (useful for staged rollouts)
- **Deployment branches**: restrict which branches can deploy to this environment (e.g., only `main` can deploy to production)

Environment secrets are only available to jobs that reference that environment, adding an extra security layer over repository-level secrets.

---

## 4.5 OIDC — Keyless Authentication to Cloud

OpenID Connect (OIDC) lets your workflows authenticate to cloud providers like AWS, Azure, and GCP without storing long-lived credentials as GitHub Secrets.

```yaml
# No stored AWS credentials — uses GitHub's OIDC provider
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: us-east-1
      - run: aws s3 ls   # works without any stored credentials!
```

**How it works:**

1. GitHub generates a short-lived JWT (JSON Web Token) for the workflow run
2. The JWT contains claims about the repository, branch, workflow, and environment
3. Your cloud provider is configured to trust GitHub's OIDC issuer
4. The cloud provider validates the JWT and issues temporary credentials
5. Temporary credentials expire automatically — no rotation needed

**AWS setup (one-time):** Create an IAM Identity Provider for `token.actions.githubusercontent.com`, then create a role with a trust policy that checks claims like `repo:myorg/myrepo:ref:refs/heads/main`.

---

## 4.6 Concurrency Control

Concurrency groups prevent multiple runs from interfering with each other — for example, two deploys happening simultaneously.

```yaml
# Cancel in-progress runs on the same branch when a new push arrives
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

```yaml
# For production deploys — queue instead of cancel
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

Concurrency can be set at the workflow level (applies to all jobs) or at the job level (applies to one job). Use `cancel-in-progress: true` for CI workflows where the latest commit supersedes older ones. Use `cancel-in-progress: false` for deployments where you want each queued run to complete in order.

---

## 4.7 Self-hosted Runners

GitHub-hosted runners work for most cases, but self-hosted runners give you control over hardware, network access, and cost.

**Register a self-hosted runner (Linux):**

```bash
# 1. GitHub → Settings → Actions → Runners → New self-hosted runner
# 2. Follow the instructions:
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.x.tar.gz -L https://github.com/actions/runner/releases/...
tar xzf actions-runner-linux-x64-2.x.tar.gz
./config.sh --url https://github.com/myorg/myrepo --token <TOKEN>
./run.sh   # or install as service: sudo ./svc.sh install
```

**Use cases for self-hosted runners:**

| Use case | Why self-hosted |
|---|---|
| Private network access | Runner can reach internal databases, APIs |
| Specific hardware | GPU builds, Apple Silicon, ARM |
| Cost | GitHub-hosted charges for private repos beyond free minutes |
| Compliance | Data must not leave your infrastructure |

**Security warning:** Never use self-hosted runners on public repositories. A pull request from a fork can trigger workflows that run code on your runner — including malicious code submitted by strangers. For public repos, always use GitHub-hosted runners or isolate self-hosted runners in throwaway VMs with no access to production systems.

---

## 4.8 Workflow Permissions

Every workflow run gets a `GITHUB_TOKEN` with a set of permissions. By default, permissions are more permissive than needed. Apply the principle of least privilege.

```yaml
# Workflow-level permissions (applies to all jobs)
permissions:
  contents: read         # read repo contents
  packages: write        # push to GHCR
  id-token: write        # OIDC
  pull-requests: write   # comment on PRs
  issues: write          # create issues

# Job-level permissions (override workflow-level for that job)
jobs:
  build:
    permissions:
      packages: write
      contents: read
```

**Common permission values:** `read`, `write`, `none`

**Avoid** `permissions: write-all` — it grants everything and makes it hard to reason about what a compromised action could do. Instead, grant only what each job needs.

---

## 4.9 Pinning Actions to SHA

Action tags like `v4` can be moved to point at different commits. If a publisher's account is compromised, a malicious commit can be pushed under the same tag — a supply chain attack.

```yaml
# BAD — tag can be overwritten (supply chain attack vector)
- uses: actions/checkout@v4

# GOOD — SHA is immutable; this exact commit cannot be changed
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

**Keeping pins updated:** Use Dependabot or Renovate Bot to automatically open PRs when a new version of a pinned action is released. The comment after the SHA keeps the human-readable version visible.

In your `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

---

## 4.10 Debugging Workflows

When a workflow fails and the logs aren't enough, use these techniques.

**Add debug logging at runtime:**

```yaml
- name: Enable debug
  run: echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV
```

**Print all context objects:**

```yaml
- name: Dump context
  run: |
    echo "github context:"
    echo '${{ toJSON(github) }}'
    echo "env context:"
    echo '${{ toJSON(env) }}'
```

**Re-run with debug logging from the UI:**
Actions → select the failed run → Re-run jobs → check "Enable debug logging"

This sets two debug variables (`ACTIONS_STEP_DEBUG` and `ACTIONS_RUNNER_DEBUG`) that cause the runner to emit verbose output for every step and runner operation.

**Other debugging tips:**

- Use `tmate` action to open an SSH session into the runner for interactive debugging
- Check the "Set up job" step for environment variable and secret availability issues
- Add `if: failure()` steps to print diagnostic information only when a job fails

---

## Summary

| Feature | Purpose |
|---|---|
| Matrix builds | Run one job across many configurations in parallel |
| Reusable workflows | Share full workflows across repositories or pipelines |
| Composite actions | Package steps into reusable single-step actions |
| Environments | Add approval gates and restrictions to deployments |
| OIDC | Authenticate to cloud without stored credentials |
| Concurrency | Prevent conflicting or redundant pipeline runs |
| Self-hosted runners | Private network, specific hardware, cost control |
| Permissions | Least-privilege GITHUB_TOKEN scoping |
| SHA pinning | Immutable action references, supply chain safety |
| Debug logging | Verbose runner output for troubleshooting |

---

## Knowledge Check

1. A matrix with `os: [ubuntu, windows]` and `node: [18, 20, 22]` produces how many jobs? What does `fail-fast: false` change about this?
2. What is the difference between a reusable workflow and a composite action? When would you choose one over the other?
3. Explain the OIDC authentication flow. What makes it more secure than storing cloud credentials as GitHub Secrets?
4. You have a production deploy job that should never run two instances concurrently, but also should not cancel a running deploy when a new commit arrives. How do you configure `concurrency` for this?
5. Why is pinning an action to a tag like `@v4` potentially dangerous? What is the safe alternative, and how do you keep it maintained?

---

## Hands-on Exercise

**Goal:** Refactor and secure a workflow using the advanced features from this chapter.

1. Take a workflow that runs tests on a single Node version and convert it to a matrix build across at least two Node versions and two operating systems.
2. Extract the build logic into a reusable workflow. Call it from the main workflow using `uses:`.
3. Add an OIDC permissions block (`id-token: write`) and the `aws-actions/configure-aws-credentials` step to a deploy job — even if you don't have an AWS account, write the configuration as if you do.
4. Add a `concurrency` block that cancels in-progress CI runs on the same branch but does not cancel production deploys.
5. Find all `uses:` references in your workflow and pin them to their current SHA. Add a comment with the human-readable version.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="03-github-actions-fundamentals.md">← Previous: GitHub Actions Fundamentals</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="05-docker-in-cicd.md">Next: Docker in CI/CD →</a>
</div>
