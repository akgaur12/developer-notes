# Chapter 6 — Testing Strategies in CI/CD

## Learning Objectives

By the end of this chapter, you will be able to:

- Describe the test pyramid and how each layer maps to CI job design
- Configure GitHub Actions service containers for integration tests
- Set up test reporting and code coverage in CI pipelines
- Choose between fail-fast and full-run strategies for different test types
- Parallelize tests across matrix shards to reduce total pipeline time
- Add code quality gates (linting, type checking, static analysis) to CI
- Configure Dependabot for automated dependency updates
- Set up branch protection rules that enforce CI passing before merge
- Deploy ephemeral preview environments per pull request

---

## 6.1 The Test Pyramid

The test pyramid is a model for deciding how many tests of each type to write. The shape reflects the cost and speed trade-offs at each level.

```
         ▲
        /E2E\          Slow, costly, few
       /─────\
      /  Integ \       Medium speed, moderate count
     /──────────\
    /    Unit    \     Fast, cheap, many
   ──────────────
```

| Layer | What it tests | Speed | Count | Cost |
|---|---|---|---|---|
| Unit | One function or class in isolation; dependencies mocked | < 1ms each | Thousands | Very low |
| Integration | How components interact; hits real databases, queues, external services | Seconds each | Hundreds | Medium |
| E2E | Full user journey through the browser or API; entire stack running | Minutes each | Dozens | High |

**In CI, the pyramid maps to job design:**

- Run unit tests on every push; they finish in seconds and give immediate signal
- Run integration tests on every push or every PR; they take longer but catch wiring bugs
- Run E2E tests on PRs to main or on a schedule; they are too slow for every commit

---

## 6.2 Running Tests in CI by Type

**Unit tests — fast, no external dependencies:**

```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:unit
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: unit-test-results
          path: junit.xml
```

**Integration tests — with real service dependencies:**

```yaml
  integration-tests:
    runs-on: ubuntu-latest
    services:                         # spin up sidecar containers
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
```

The `options` field passes Docker flags directly to the service container. Health check options make GitHub Actions wait until the service is ready before running your steps.

---

## 6.3 Services in GitHub Actions

The `services` block starts Docker containers as sidecars on the same runner before your steps begin. Key behaviors:

- Services are started before the first step and stopped after the last step
- Ports are automatically mapped to `localhost` on the runner — no need to specify `-p` flags
- Each service gets a hostname matching its key name (e.g., `postgres`, `redis`) — but only if you're running steps inside a container; for host runner steps, use `localhost`
- Health check options (`--health-cmd`, `--health-interval`, `--health-retries`) cause the runner to wait until the service reports healthy before proceeding
- Service containers share the same network bridge

**Connecting from host runner steps:**

```yaml
env:
  DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb
```

**Connecting from container-based steps** (when the job itself uses `container:`):

```yaml
env:
  DATABASE_URL: postgresql://postgres:test@postgres:5432/testdb
```

---

## 6.4 Test Reporting and Coverage

Generate machine-readable test results and coverage reports, then publish them so they appear in the GitHub UI.

```yaml
- name: Run tests with coverage
  run: npm test -- --coverage --reporters=default --reporters=jest-junit
  env:
    JEST_JUNIT_OUTPUT_DIR: ./test-results

- name: Upload test results artifact
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/

- name: Publish test report
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Test Results
    path: test-results/*.xml
    reporter: jest-junit

- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v4
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
    files: ./coverage/lcov.info
    fail_ci_if_error: true
```

`dorny/test-reporter` publishes test results as a GitHub Check, visible in the Checks tab of a PR. Failed tests appear as annotations directly on the PR.

`fail_ci_if_error: true` on the Codecov upload fails the pipeline if the coverage report cannot be processed — preventing silent coverage regressions.

**Coverage enforcement** — add a minimum coverage threshold in your test config (e.g., `jest.config.js`):

```js
coverageThreshold: {
  global: {
    branches: 80,
    functions: 80,
    lines: 80,
    statements: 80,
  },
},
```

---

## 6.5 Fail Fast vs Full Run

```yaml
strategy:
  fail-fast: true   # cancel all matrix jobs if one fails (default)

strategy:
  fail-fast: false  # let all jobs finish regardless of failures
```

**When to use `fail-fast: true` (the default):**

- Unit test matrix across Node versions: if Node 18 fails, Node 20 and 22 will likely also fail; cancel early and save minutes
- You want the fastest possible feedback for most cases

**When to use `fail-fast: false`:**

- Compatibility test matrix: you need to see which specific Node/OS combinations fail, not just "something failed"
- Flaky test investigations: you want a full picture of flakiness across all shards
- Publishing or release jobs: all targets should run even if one fails

---

## 6.6 Parallelizing Tests

For large test suites, split tests across multiple runners using sharding to reduce total wall-clock time.

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx jest --shard=${{ matrix.shard }}/4
```

Jest's built-in `--shard=N/M` flag splits the test suite into M equal parts and runs the Nth part. Four shards in parallel reduce a 4-minute test suite to roughly 1 minute.

**Alternative approaches:**

| Tool | Sharding mechanism |
|---|---|
| Jest | `--shard=N/M` |
| Vitest | `--shard=N/M` |
| pytest | `pytest-xdist` with `-n auto` or `--dist=loadfile` |
| RSpec | `--format` + `parallel_tests` gem |
| Playwright | `--shard=N/M` |

---

## 6.7 Code Quality Gates

Beyond functional tests, CI should enforce code style, type correctness, and dependency security.

```yaml
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint              # ESLint, Prettier
      - run: npm run type-check        # TypeScript compiler check

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      - name: Run CodeQL analysis
        uses: github/codeql-action/init@v3
        with:
          languages: javascript
```

Make these jobs required status checks (covered in section 6.9) so they block merges when they fail. A PR with failing lint or a high-severity vulnerability cannot be merged until the issue is resolved.

---

## 6.8 Static Analysis in CI

Static analysis tools examine your code and dependencies without running them.

| Category | Tool | What it finds |
|---|---|---|
| SAST | CodeQL | Security vulnerabilities in source code |
| SAST | Semgrep | Custom rules + security patterns |
| SAST | Bandit (Python) | Python-specific security issues |
| Dependency scanning | Dependabot | Outdated or vulnerable dependencies |
| Dependency scanning | Snyk | Vulnerable dependencies with fix suggestions |
| Dependency scanning | OWASP Dependency Check | CVE database matching |
| Secret scanning | TruffleHog | Secrets committed to git history |
| Secret scanning | GitLeaks | Hardcoded credentials in diffs |
| Secret scanning | GitHub secret scanning | Automatic detection of known secret patterns |
| License compliance | FOSSA | Open source license policy enforcement |

**Enabling Dependabot for automated dependency updates:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    groups:
      dev-dependencies:
        patterns: ["*"]
        dependency-type: development
    open-pull-requests-limit: 10
```

Grouping dev dependencies into a single PR reduces noise. Dependabot will open one PR per week with all dev dependency updates bundled, rather than one PR per package.

---

## 6.9 Branch Protection Rules

Branch protection rules enforce that CI must pass before any code can be merged to protected branches like `main`.

Navigate to: **GitHub → Settings → Branches → Add branch protection rule**

```
Branch name pattern: main

✓ Require a pull request before merging
  ✓ Require approvals: 1
  ✓ Dismiss stale pull request approvals when new commits are pushed
  ✓ Require review from Code Owners

✓ Require status checks to pass before merging
  ✓ Require branches to be up to date before merging
  Status checks that are required:
    ✓ unit-tests
    ✓ lint
    ✓ integration-tests

✓ Require conversation resolution before merging

✓ Do not allow bypassing the above settings
```

"Require branches to be up to date" means the PR branch must include the latest `main` commits before merging. This prevents a scenario where two PRs each pass CI independently but conflict when merged together.

"Do not allow bypassing" ensures that even repository admins cannot push to `main` without going through the PR and CI process.

---

## 6.10 Test Environments: Ephemeral Review Apps

Deploy a live preview environment for each pull request so reviewers can test changes without running the app locally.

```yaml
deploy-preview:
  if: github.event_name == 'pull_request'
  runs-on: ubuntu-latest
  environment:
    name: preview-pr-${{ github.event.number }}
    url: https://pr-${{ github.event.number }}.preview.myapp.com
  steps:
    - uses: actions/checkout@v4
    - run: ./deploy-preview.sh ${{ github.event.number }}
    - name: Comment PR with preview URL
      uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '🚀 Preview deployed: https://pr-${{ github.event.number }}.preview.myapp.com'
          })
```

**Teardown the preview on PR close:**

```yaml
teardown-preview:
  if: github.event_name == 'pull_request' && github.event.action == 'closed'
  runs-on: ubuntu-latest
  steps:
    - run: ./teardown-preview.sh ${{ github.event.number }}
```

Platforms that support ephemeral environments natively: Vercel, Netlify, Railway, Render, Fly.io. Kubernetes-based setups can use namespace-per-PR patterns with Helm or Argo CD.

---

## Summary

| Topic | Key Point |
|---|---|
| Test pyramid | Write many unit tests, fewer integration tests, fewest E2E tests |
| Service containers | `services:` block starts sidecar containers; health checks wait for readiness |
| Test reporting | SARIF + dorny/test-reporter surfaces results as GitHub Check annotations |
| Coverage enforcement | Set thresholds in test config; `fail_ci_if_error` catches upload failures |
| fail-fast | Use `true` for fast feedback; use `false` when you need a full picture |
| Sharding | `--shard=N/M` splits tests across parallel runners |
| Code quality gates | Lint, type-check, and SAST as required status checks |
| Dependabot | Automates dependency update PRs on a schedule |
| Branch protection | Required status checks block merges until CI passes |
| Review apps | Per-PR environments for live preview and E2E testing |

---

## Knowledge Check

1. Where in the test pyramid do service containers in GitHub Actions primarily help? What problem do they solve compared to using mocks?
2. A team has a matrix test job with `fail-fast: false`. Tests pass on Node 18 and Node 22 but fail on Node 20. What would have happened if `fail-fast: true` (the default) was used instead?
3. Explain the difference between `actions/upload-artifact` and `dorny/test-reporter`. What does each one do with test result files?
4. You have a 6-minute test suite and want to reduce it to under 2 minutes using sharding. Write the `strategy.matrix` and the jest command to split it across 4 shards.
5. A developer pushes directly to `main` and bypasses branch protection. What setting specifically prevents this, and where is it configured?

---

## Hands-on Exercise

**Goal:** Add a full, production-quality test pipeline to a project.

1. Create a GitHub Actions workflow with three separate jobs: `unit-tests`, `integration-tests`, and `lint`.
2. In the `integration-tests` job, add a PostgreSQL service container with a health check. Run a test that actually connects to the database.
3. Add coverage reporting to the `unit-tests` job — output a JUnit XML file and upload it as an artifact. Use `dorny/test-reporter` to publish results as a GitHub Check.
4. Shard the unit tests across 3 parallel runners using a matrix.
5. Enable branch protection on `main` that requires all three jobs to pass. Verify by opening a PR with a deliberately failing test and confirming GitHub blocks the merge.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="05-docker-in-cicd.md">← Previous: Docker in CI/CD</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="07-deployment-strategies.md">Next: Deployment Strategies →</a>
</div>
