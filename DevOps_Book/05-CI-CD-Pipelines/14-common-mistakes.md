# Chapter 14 — Common Mistakes & Pitfalls

## Learning Objectives

By the end of this chapter, you will be able to:

- Recognize the most common CI/CD mistakes before they reach production
- Understand why each mistake happens and what it costs
- Apply concrete fixes using real workflow examples
- Use emergency recovery commands when pipelines break under pressure

---

## Overview

CI/CD pipelines accumulate bad habits gradually. Each mistake in this chapter is common, has a real cost (security risk, wasted time, production incident, or all three), and is fixable. For each one: what it looks like, why it happens, and how to fix it.

---

## Mistake 1: Hardcoding Secrets in Workflow Files

```yaml
# WRONG — secret committed to git, visible in repository history
env:
  DB_PASSWORD: mysecret123
  API_KEY: sk-prod-abc123xyz

# CORRECT
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  API_KEY: ${{ secrets.API_KEY }}
```

**Why it happens:** Copy-pasting from local `.env` files, moving fast during initial setup, not understanding that workflow files are committed to the repository.

**Why it matters:** Even a private repository can have contributors who should not see production credentials. The secret also lives in git history forever — a future leak of the repository exposes all past credentials.

**Fix:** Use `git filter-repo` to scrub the secret from history. Rotate the compromised credentials immediately — assume they are compromised the moment they touch a git commit.

```bash
# Remove secret from entire git history
git filter-repo --replace-text <(echo 'mysecret123==>REDACTED')
```

**Prevention:** Add a TruffleHog scan to CI to catch accidentally committed secrets before they merge.

---

## Mistake 2: Running Full CI on Every Push to Every Branch

```yaml
# WRONG — runs expensive integration tests on every tiny commit to every branch
on:
  push:

# CORRECT — right work at the right trigger
on:
  push:
    branches: [main]         # full pipeline on main
  pull_request:
    branches: [main]         # CI on PRs to main only
```

**Why it happens:** The default `on: push` is the simplest trigger and it is what most tutorials show first.

**Why it matters:** Developers pushing work-in-progress commits to feature branches run full integration test suites on every save. This burns GitHub Actions minutes, slows down runner queues, and provides no value — these commits are not ready for review.

**Fix:** Add branch filters. For most teams, feature branches need only lightweight checks (lint, unit tests) while `main` and open PRs to `main` run the full suite.

---

## Mistake 3: Not Pinning Action Versions

```yaml
# WRONG — @main or @latest can change without notice (supply chain risk)
- uses: actions/checkout@main
- uses: some-org/risky-action@latest

# CORRECT — pin to SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

**Why it happens:** SHA pins look unwieldy. Using a tag feels cleaner and easier to read.

**Why it matters:** A tag like `@v3` is mutable — the action author can push a new commit to that tag at any time. Your pipeline would silently start running different code. SHA pinning ensures the exact code that ran last week is the exact code running today. This is a supply chain security control (similar to the `tj-actions/changed-files` incident in 2023, where a compromised action leaked secrets from thousands of pipelines).

**Fix:** Pin all third-party actions to a full SHA. Add a comment with the human-readable version tag.

**Automation:** Use Renovate or Dependabot to keep SHA pins up-to-date automatically — you get the security of pinning without the maintenance burden.

```yaml
# dependabot.yml — keep action SHAs current
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

---

## Mistake 4: Rebuilding Images for Each Environment

```
WRONG:
  CI builds "staging image" → staging tests it
  CI builds "prod image"   → production runs it
  (different builds = different code = staging tested something different from what prod runs)

CORRECT:
  CI builds ONE image tagged with git SHA
  → staging pulls and tests that exact SHA
  → production retags and deploys that same SHA
  → the artifact that was tested is the artifact that runs
```

**Why it happens:** Separate `Dockerfile.staging` and `Dockerfile.prod` files seem like a reasonable way to handle environment-specific config. Or the deploy pipeline just runs `docker build` again because it is the easiest approach.

**Why it matters:** Environment-specific builds mean environment-specific bugs. An issue that only manifests in the production build will never appear in staging. You are testing one thing and deploying another.

**Fix:** Build once. Pass environment config via environment variables at runtime, not at build time. Use the same image across all environments.

```bash
# Promote image to production without rebuilding
REGISTRY=ghcr.io/myorg/myapp
docker pull $REGISTRY:$GITHUB_SHA
docker tag  $REGISTRY:$GITHUB_SHA $REGISTRY:prod
docker push $REGISTRY:prod
```

---

## Mistake 5: No Caching

```yaml
# WRONG — installs 500+ packages fresh every run
- run: npm install    # 90 seconds every time

# CORRECT — cached
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: npm        # 5 seconds on cache hit
```

**Why it happens:** The pipeline works without caching. Adding caching requires a few extra lines and some knowledge of cache keys.

**Why it matters:** For a team of 10 pushing 20 PRs a day, saving 90 seconds per run is 30 minutes of runner time per day, 150 minutes per week. Caching also makes CI feel fast to developers, which encourages them to actually wait for and trust CI results.

**Common caches to add:**

| Language | Action | Cache key |
|----------|--------|-----------|
| Node.js | `actions/setup-node` | `cache: npm` or `cache: pnpm` |
| Python | `actions/setup-python` | `cache: pip` |
| Go | `actions/setup-go` | `cache: true` |
| Docker layers | `docker/build-push-action` | `cache-from: type=gha` |
| Maven/Gradle | `actions/cache` | `~/.m2` or `~/.gradle` |

**Impact:** 2-3x pipeline speedup just from dependency caching, with no changes to test logic.

---

## Mistake 6: Ignoring Flaky Tests

**What it looks like:** A test passes on one run, fails on the next, passes again — without any code changes. The team starts re-running CI until it goes green and merges anyway.

**Why it happens:** Flaky tests are caused by timing dependencies, shared mutable state between tests, reliance on external services, or non-deterministic test ordering. They are annoying to fix and easy to defer.

**Why it matters:** Flaky CI is ignored CI. Once the team learns that a red pipeline might just need a rerun, they stop treating red pipelines as a signal. A real bug sneaks through because everyone assumes it is just flakiness.

**Fix immediately:**
1. Quarantine the flaky test (move to a separate job that does not block merges)
2. Identify the root cause: timing issue, shared state, async race condition
3. Fix and restore to the main suite

**Track flakiness:**
```bash
# Find recently failing runs to spot patterns
gh run list --workflow=ci.yml --status=failure --limit=20
```

**Rule:** A flaky test is a P1 bug. Fix it before the sprint ends.

---

## Mistake 7: No Rollback Plan

```
WRONG:
  Deploy → something breaks → scramble to figure out how to roll back
  → 45 minutes of downtime, all hands on deck, post-mortem required

CORRECT:
  Deploy → smoke test → automatic rollback on failure → < 5 minutes recovery
  → on-call engineer sees alert, reviews auto-rollback, investigates calmly
```

**Why it happens:** Rollback is often an afterthought. The deploy works, so planning for failure feels pessimistic.

**Why it matters:** Every deployment will eventually fail in production. The question is how long recovery takes. An untested rollback procedure fails under pressure.

**Fix:** Build rollback into every deployment pipeline (see Chapter 13, section 13.9). Test the rollback procedure quarterly — run a planned drill where you deliberately deploy a broken version and verify the automated rollback works.

---

## Mistake 8: Overly Broad Permissions

```yaml
# WRONG — full write access to repository contents, packages, issues, PRs
permissions:
  contents: write
  packages: write
  id-token: write
  issues: write
  pull-requests: write

# CORRECT — least privilege: only what this specific job needs
permissions:
  contents: read           # read checkout only
  packages: write          # only if this job pushes a Docker image
```

**Why it happens:** Copy-pasting a permissions block from a tutorial that was written for a more complex workflow.

**Why it matters:** A compromised third-party action running in your workflow inherits the permissions you declared. Broad permissions turn a compromised action into a supply chain attack on your repository, packages, and issues.

**Fix:** Audit permissions in every workflow. Set the minimum required. Set a default at the workflow level and override only for jobs that need more:

```yaml
# Deny everything by default at workflow level
permissions: {}

jobs:
  deploy:
    permissions:
      packages: write    # only this job gets this permission
```

---

## Mistake 9: Long-Running Jobs Without Timeouts

```yaml
# WRONG — job hangs forever, burns minutes, blocks other runners
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test       # what if a test opens a port and waits forever?

# CORRECT — timeout at both job and step level
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15       # job-level: kill everything after 15 minutes
    steps:
      - run: npm test
        timeout-minutes: 10   # step-level: this specific command gets 10 minutes
```

**Why it happens:** Tests usually finish quickly, so timeouts seem unnecessary. Until a test hangs waiting for a network connection that never arrives.

**Why it matters:** A hung job occupies a runner indefinitely. On GitHub-hosted runners, this burns paid minutes. On self-hosted runners, it blocks every other job waiting for that runner. The default GitHub Actions job timeout is 6 hours — a hung job at 2 AM might not be noticed until morning.

**Rule of thumb:** Set `timeout-minutes` to 2x the typical runtime of the job. It should never trigger under normal conditions, but should catch a stuck job within a reasonable window.

---

## Mistake 10: Deploying Without Health Checks

```yaml
# WRONG — declare victory as soon as kubectl apply exits 0
- run: kubectl apply -f deployment.yaml
- run: echo "Deployed!"   # pods might be CrashLoopBackOff right now

# CORRECT — wait for actual pod readiness, then verify behavior
- run: kubectl apply -f deployment.yaml
- run: kubectl rollout status deployment/myapp --timeout=5m
- run: ./smoke-test.sh https://api.myapp.com
```

**Why it happens:** `kubectl apply` returning 0 feels like success. The deployment is "done."

**Why it matters:** Kubernetes accepts a deployment spec immediately and then works to reconcile it asynchronously. Pods may be crashing, image pulls may be failing, or readiness probes may be timing out — all after `kubectl apply` returns 0. Without a health check, the pipeline is green and users are getting 500 errors.

**Smoke test script pattern:**

```bash
#!/bin/bash
# smoke-test.sh
URL=$1
MAX_RETRIES=5
for i in $(seq 1 $MAX_RETRIES); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL/health")
  if [ "$STATUS" = "200" ]; then
    echo "Health check passed ($URL returned 200)"
    exit 0
  fi
  echo "Attempt $i: got $STATUS, retrying..."
  sleep 10
done
echo "Health check failed after $MAX_RETRIES attempts"
exit 1
```

---

## Mistake 11: Not Cleaning Up Ephemeral Resources

```yaml
# WRONG — Docker containers/networks/volumes accumulate on self-hosted runners
steps:
  - run: docker compose up -d
  - run: npm run test:integration
  # test fails here — the next step never runs, containers are never stopped

# CORRECT — cleanup runs regardless of whether previous steps failed
steps:
  - run: docker compose up -d
  - run: npm run test:integration

  - name: Cleanup
    if: always()              # this runs even if npm test failed
    run: docker compose down -v --remove-orphans
```

**Why it happens:** Sequential steps feel safe — the cleanup is right there after the test. The problem is that a failed step stops execution by default.

**Why it matters:** On self-hosted runners, leaked containers consume memory, ports, and disk. After enough failed test runs, the runner runs out of resources and all jobs start failing with cryptic errors. Debugging is painful because the root cause was a leak from hours ago.

**Rule:** Any job that creates a resource (container, temp file, cloud resource) must clean it up with `if: always()`.

---

## Mistake 12: One Massive Workflow File

**What it looks like:** A single `ci-cd.yml` with 500+ lines covering lint, test, build, deploy-staging, approval gate, deploy-prod, and notification — all in one file.

**Why it happens:** It starts small and grows. Each addition seems like a minor extension.

**Why it matters:** When something breaks, debugging a 500-line workflow is hard. Reusing a step from it in another workflow is impossible. Reviewing changes to it requires reading the whole thing to understand what might be affected.

**Fix:** Split by concern. Each file should be independently understandable.

```
ci.yml        — runs on every PR: lint, type-check, unit tests
build.yml     — runs on main: build and push Docker image
deploy.yml    — called by build.yml or manually: deploy to an environment
security.yml  — runs on a schedule: dependency audit, SAST, container scan
release.yml   — runs on version tag: changelog, GitHub release, prod deploy
```

Extract repeated setup steps into composite actions so they are maintained in one place.

---

## Mistake 13: Not Testing the Pipeline Itself

**What it looks like:** A developer makes a change to `ci.yml`, merges it, and discovers it is broken when the next PR runs. Now all CI is broken until a fix is merged — which requires CI to pass first.

**Why it happens:** Pipeline config files feel like infrastructure, not code. Testing them requires extra discipline.

**Fix:**

1. Test workflow changes on a branch before merging — open a PR and observe the workflow run
2. Use `act` for rapid local iteration without waiting for GitHub

```bash
# Install act (local GitHub Actions runner)
brew install act       # macOS
# or
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Run a specific workflow locally
act push -W .github/workflows/ci.yml

# Run with specific event payload
act pull_request -W .github/workflows/ci.yml -e event.json
```

`act` is not a perfect replica of GitHub-hosted runners, but it catches syntax errors, missing environment variables, and logic bugs without burning cloud minutes.

---

## Mistake 14: Ignoring Pipeline Costs

**What it looks like:** The bill comes in and GitHub Actions minutes are 3x what was expected. Nobody knows where they went.

**Why it happens:** Minutes feel abstract until the bill arrives. No one owns pipeline cost the way someone owns infrastructure cost.

**Common cost traps:**

| Trap | Cost multiplier |
|------|----------------|
| `macos-latest` runner | 10x vs. `ubuntu-latest` |
| No `cancel-in-progress` on PRs | Multiple runs for rapid pushes |
| No path filters — runs on doc changes | Wasted full pipeline runs |
| Integration tests on every push | Should only run on PRs to main |
| Self-hosted runner running idle 24/7 | Pay for compute 24 hours a day |

**Fix: Add concurrency and path filters:**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true    # cancel old run when new push arrives

on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
      - '.github/workflows/**'
    # ignores: docs/**, *.md — no need to run CI for documentation changes
```

**Audit monthly:**
```bash
# Check cache storage usage
gh api /repos/myorg/myrepo/actions/cache/usage

# See most expensive workflows by run count
gh api /repos/myorg/myrepo/actions/workflows | jq '.workflows[] | {name, id}'
```

---

## Emergency Recovery Commands

When pipelines break under pressure, these commands help you regain control quickly.

```bash
# Cancel a stuck or runaway workflow run
gh run cancel <run-id>

# Re-run only the failed jobs (not the whole pipeline)
gh run rerun <run-id> --failed

# Check how many runs are currently queued
gh api /repos/myorg/myrepo/actions/runs?status=queued | jq '.workflow_runs | length'

# List recent failures to find a pattern
gh run list --workflow=ci.yml --status=failure --limit=20

# Force-bypass branch protection in a critical hotfix (admin only, use sparingly)
git push --force-with-lease origin hotfix/critical:main

# Roll back the last Kubernetes deployment
kubectl rollout undo deployment/myapp

# Check rollout history before deciding which version to go back to
kubectl rollout history deployment/myapp

# Roll back to a specific revision
kubectl rollout undo deployment/myapp --to-revision=3

# Roll back a Docker Swarm service
docker service rollback myapp

# Check runner queue health on a self-hosted runner
gh api /repos/myorg/myrepo/actions/runners | jq '.runners[] | {name, status, busy}'
```

---

## Summary

| # | Mistake | Key Fix |
|---|---------|---------|
| 1 | Hardcoded secrets | Use `${{ secrets.NAME }}`; add TruffleHog scan |
| 2 | Full CI on all branches | Add branch filters; use path filters |
| 3 | Unpinned action versions | Pin to SHA; use Dependabot to update |
| 4 | Rebuilding per environment | Build once, tag with SHA, promote by retag |
| 5 | No caching | Add `cache:` to language setup actions |
| 6 | Ignoring flaky tests | Quarantine immediately; fix root cause |
| 7 | No rollback plan | Automate rollback; run rollback drills |
| 8 | Overly broad permissions | Use `permissions: {}` default; grant minimally |
| 9 | No timeouts | Set `timeout-minutes` on all jobs and slow steps |
| 10 | Deploy without health check | Add `rollout status` + smoke test after deploy |
| 11 | No cleanup for ephemeral resources | Use `if: always()` on cleanup steps |
| 12 | One massive workflow file | Split by concern; extract composite actions |
| 13 | Not testing the pipeline | Test on branch first; use `act` locally |
| 14 | Ignoring costs | Add concurrency + cancel-in-progress + path filters |

---

## Knowledge Check

1. Why is `if: always()` necessary for cleanup steps, rather than just putting the cleanup step last?
2. What is the supply chain risk of using `- uses: some-action@latest`, and how does SHA pinning mitigate it?
3. Explain why rebuilding a Docker image for production (instead of promoting the staging image) violates the "test what you ship" principle.
4. A developer says "the flaky test usually passes, so we can just rerun until it goes green." What is the systemic risk of this approach?
5. What does `cancel-in-progress: true` in a concurrency block do, and when would you NOT want to use it?

---

## Hands-on Exercise

**Pipeline Audit and Fix**

1. Open a real workflow file from your project (or a public repository if you do not have one).
2. Work through the 14-mistake checklist. Mark each as: present, not present, or not applicable.
3. Pick at least 3 mistakes that are present and implement fixes:
   - If no caching exists, add it and measure the before/after runtime
   - If permissions are not scoped, add a `permissions:` block with least privilege
   - If there are no timeouts, add `timeout-minutes` to all jobs
4. Open a pull request with your changes and verify the pipeline runs faster or cleaner.
5. Document: which 3 mistakes you fixed, what the measurable improvement was, and which mistake would be highest priority to fix next.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="13-best-practices.md">← Previous: Best Practices</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="15-projects.md">Next: Hands-On Projects →</a>
</div>
