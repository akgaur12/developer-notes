# CI/CD Integration

[Chapter 14](./14-docker-integration.md) built a Dockerfile that only produces a correct, lean image if every `uv sync` call inside it is invoked exactly right — `--frozen`, `--no-dev`, in the right order, against the right workspace member. Trusting a human to type that correctly by hand, every time, on every machine that ever builds ExpenseFlow's image, is exactly the kind of trust CI/CD exists to remove from the equation. This chapter wires uv into GitHub Actions so that every pull request automatically proves ExpenseFlow's dependencies are locked and installable, its code passes lint/type/test gates, it runs correctly across every Python version the team supports, and — bridging back to this repo's sibling Alembic course one more time — its migration chain actually applies cleanly against a real, disposable PostgreSQL instance before anything reaches production.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure `astral-sh/setup-uv` in a GitHub Actions workflow to install a pinned uv version and restore uv's cache between runs.
- Explain why CI should always use `uv sync --locked` and never a bare `uv sync`, connecting this directly to Chapter 8's lockfile-reproducibility lesson.
- Write jobs that run `uv run ruff check`, `uv run mypy`, and `uv run pytest` against ExpenseFlow's workspace.
- Build a matrix-testing job that installs and tests against Python 3.11, 3.12, and 3.13 using `uv python install`, one interpreter per matrix leg.
- Write a migration-drift/smoke-check job that runs `uv run alembic upgrade head` against an ephemeral PostgreSQL service container, and explain what specific class of bug it catches.
- Assemble a complete, working GitHub Actions workflow for ExpenseFlow that ties all of the above together with correct job dependencies.

---

## Prerequisites for This Chapter

- The distinction between `--frozen` and `--locked`, and why CI specifically should prefer `--locked` — [Chapter 8: Lock Files & Reproducibility](./08-lock-files-and-reproducibility.md).
- Dev dependencies (`ruff`, `mypy`, `pytest`) as their own group, run via `uv run` — [Chapter 10: Development Dependencies & Tooling](./10-development-dependencies-and-tooling.md).
- ExpenseFlow's workspace structure and the full `packages/api/pyproject.toml` — [Chapter 12: Workspaces & Monorepos](./12-workspaces-and-monorepos.md) and [Chapter 13](./13-fastapi-sqlalchemy-alembic-integration.md).
- The Dockerfile this chapter's pipeline will eventually build and could push to a registry — [Chapter 14: Docker Integration](./14-docker-integration.md).
- This chapter assumes familiarity with `alembic upgrade head` and ephemeral-database testing from this repo's sibling [Alembic course, Chapter 15: CI/CD Integration](../../Databases/alembic-course/15-cicd-integration.md) — that chapter's service-container and drift-check material is the direct model for Section 7 here, applied through uv rather than a bare `pip`/`alembic` setup.

---

## 1. What ExpenseFlow's Pipeline Needs to Verify

Before writing any YAML, it's worth being precise about what a uv-aware CI pipeline is actually trying to prove — it's easy to assemble a pipeline that runs green while quietly skipping the checks that actually catch real incidents. There are five distinct claims worth verifying, each independent of the others:

1. **The lockfile is honest** — `uv.lock` genuinely reflects what `pyproject.toml` declares, with nothing stale and nothing hand-edited out of sync. `uv sync --locked` (Section 4) verifies this directly.
2. **The dependency set installs cleanly and reproducibily**, using exactly the versions the lockfile pins — not whatever a fresh resolution happens to produce on a given runner.
3. **The code passes its static-analysis and test gates** — `ruff check`, `mypy`, `pytest` — using the exact same tool versions every developer's local `uv run` invocations use, because they're all reading the same `uv.lock`.
4. **The application works across every Python version the team actually supports in production** — for ExpenseFlow, 3.11 through 3.13 — not just whatever version happens to be on a given developer's laptop.
5. **The migration chain genuinely applies against real PostgreSQL**, and the SQLAlchemy models haven't drifted out of sync with it — the exact concern the sibling Alembic course's own Chapter 15 covers in depth, now expressed through `uv run` instead of a bare `alembic` invocation.

Sections 2–3 set up the tooling; Section 4 covers claim 1–2; Section 5 covers claim 3; Section 6 covers claim 4; Section 7 covers claim 5; Section 8 assembles all of it into one workflow.

---

## 2. `astral-sh/setup-uv`: Getting uv Into GitHub Actions

### 2.1 The action itself

Astral publishes and maintains [`astral-sh/setup-uv`](https://github.com/astral-sh/setup-uv), a GitHub Action that installs a specific uv version onto the runner, in one step, without any of the manual `curl | sh` installer invocation Chapter 1 taught for a developer's own machine:

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v5
  with:
    version: "0.5.11"
    enable-cache: true
```

Pinning `version` explicitly (rather than leaving it to whatever `setup-uv`'s own default resolves to) matters for exactly the same reason Chapter 14 pinned a specific `ghcr.io/astral-sh/uv` tag in the Dockerfile — an unannounced uv version bump changing resolver behavior mid-project should be something the team opts into deliberately, via a reviewed change to this one line, not something that silently happens because a workflow's uv-version default moved underneath it.

### 2.2 What `enable-cache: true` actually does

`enable-cache: true` tells the action to automatically save and restore uv's own cache directory (the same content-addressable cache Chapter 3 covered in depth) using GitHub Actions' built-in cache mechanism, keyed by a hash of the repository's lockfile by default. This is the action doing, automatically, exactly what Section 3 below explains manually — restoring a warm cache between workflow runs so dependency installation is fast on every run after the first.

---

## 3. Caching uv's Cache Between Runs

### 3.1 Why this matters even with a lockfile

`uv sync --locked` (Section 4) guarantees *which* versions get installed, but it doesn't by itself make installing them fast — without a warm cache, every CI run downloads and unpacks every dependency from scratch, every single time, which for ExpenseFlow's dependency set (`fastapi`, `sqlalchemy`, `alembic`, `asyncpg`, and their transitive closures, plus the dev group) adds real, repeated seconds to every single workflow run across every PR and every push.

### 3.2 The caching mechanism

`setup-uv`'s built-in caching (Section 2.2) handles the common case well, but it's worth understanding what it's doing underneath, because some teams need more control than the default provides — for instance, wanting the cache key to also vary by Python version in a matrix job:

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v5
  with:
    version: "0.5.11"
    enable-cache: true
    cache-dependency-glob: "uv.lock"
```

`cache-dependency-glob` controls which files' contents feed the cache key — pointing it at `uv.lock` specifically (rather than the action's broader default glob) means the cache is invalidated exactly when the lockfile changes, and reused exactly when it doesn't, matching the same "dependency changes invalidate, source changes don't" caching principle [Chapter 14, Section 3](./14-docker-integration.md#3-layer-ordering-copying-lockfiles-before-source) applied to Docker layers — here applied to GitHub Actions' cache instead.

```mermaid
flowchart LR
    A["Workflow run starts"] --> B{"Cache key\n(hash of uv.lock)\nfound?"}
    B -->|"Hit — uv.lock unchanged\nsince last run"| C["Restore uv's cache directory\n(no re-download needed)"]
    B -->|"Miss — first run,\nor uv.lock changed"| D["uv sync downloads/builds\npackages fresh, populating\nthe cache for next time"]
    C --> E["uv sync --locked\n(fast — cache hit)"]
    D --> F["uv sync --locked\n(slower — cold cache)"]
    E --> G["Cache saved back\n(unchanged if already restored)"]
    F --> G
```

---

## 4. `uv sync --locked`: Never a Bare `uv sync` in CI

### 4.1 The rule, restated for CI specifically

Chapter 8 introduced `--frozen` and `--locked` as two distinct strictness levels, both stricter than a bare `uv sync`, which will happily re-resolve and silently update `uv.lock` if `pyproject.toml` and the existing lockfile have drifted apart. CI is precisely the environment where that silent re-resolution is least acceptable — a pipeline that "passes" is supposed to mean "this exact, committed lockfile works," and a bare `uv sync` can pass while quietly using a *different* resolution than what's actually committed to the repository, defeating the entire point of running CI against a lockfile at all.

```yaml
- name: Install dependencies
  run: uv sync --locked
```

`--locked` specifically fails the build, loudly, the instant `uv.lock` would need to change to satisfy `pyproject.toml` — which is exactly the signal a PR should surface: "someone changed a dependency declaration and forgot to run `uv lock` before committing." This is a strictly more useful failure for CI than `--frozen`'s "use the lockfile exactly as-is, don't even check whether it's stale" behavior, which is why Chapter 14's Dockerfile uses `--frozen` (a Docker build has no interest in discovering staleness, it just wants a deterministic build) while this chapter's CI workflow uses `--locked` (CI's entire job here is to *catch* that staleness before it reaches anyone else).

### 4.2 The incident this prevents, made concrete

Chapter 8 walked through a hypothetical "it works on my machine" incident — a teammate's older uv/Python combination resolving different transitive dependency versions than what's actually committed to `uv.lock`. `uv sync --locked` in CI is the mechanical enforcement of that chapter's lesson: if a developer's local environment somehow ends up with a `uv.lock` that doesn't match what `pyproject.toml` currently declares (a merge conflict resolved incorrectly, a manual edit, a forgotten `uv lock` after adding a dependency by hand-editing `pyproject.toml`), the very first CI run on their PR fails immediately, with a clear message about exactly which part of the lockfile is out of date — catching the mismatch in code review, not in a Docker build three stages downstream, and certainly not in production.

---

## 5. Lint, Type-Check, Test

With dependencies installed via `uv sync --locked`, running ExpenseFlow's quality gates is the same set of commands Chapter 10 established for local development — the only difference is *where* they run:

```yaml
- name: Lint
  run: uv run ruff check .

- name: Format check
  run: uv run ruff format --check .

- name: Type check
  run: uv run mypy .

- name: Test
  run: uv run pytest -v --cov=app --cov-report=xml
```

This is worth pausing on precisely because there's nothing new to teach here — that's the entire point. Because `ruff`, `mypy`, and `pytest` are versioned dev dependencies in `packages/api/pyproject.toml` (Chapter 10's decision, reinforced in Chapter 13's full `pyproject.toml`), and CI installs them via the same `uv sync --locked` a developer's laptop would use, there is no meaningful way for CI to be running a different `ruff`/`mypy`/`pytest` version than what any developer just ran locally before pushing — the single most common source of "CI fails but it passed on my machine" confusion when tools are installed as ambient global installs instead (a mistake [Chapter 11](./11-tool-management-uvx.md) warned against directly).

---

## 6. Matrix Testing Across Python 3.11, 3.12, and 3.13

### 6.1 Why ExpenseFlow tests against three Python versions

ExpenseFlow itself runs on Python 3.13 in production (pinned via `.python-version`, Chapter 4), but the team supports being installed as a library dependency (`expenseflow-shared`, once it's published independently in [Chapter 16](./16-publishing-packages.md)) by other internal services that may not yet have upgraded to 3.13. Testing across 3.11, 3.12, and 3.13 in CI catches version-specific behavior differences before they surface as a consuming team's incident rather than ExpenseFlow's own.

### 6.2 The matrix, using `uv python install` per leg

```yaml
jobs:
  test-matrix:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.5.11"
          enable-cache: true

      - name: Install Python ${{ matrix.python-version }}
        run: uv python install ${{ matrix.python-version }}

      - name: Sync dependencies for this Python version
        run: uv sync --locked --python ${{ matrix.python-version }}

      - name: Run tests
        run: uv run --python ${{ matrix.python-version }} pytest -v
```

`uv python install ${{ matrix.python-version }}` is doing exactly what Chapter 4 described — downloading a `python-build-standalone` prebuilt interpreter for that specific version, side by side with any others already present on the runner, with no `pyenv` involved at all. Passing `--python` explicitly to both `uv sync` and `uv run` (rather than relying on `.python-version`, which pins a single version for local development) tells uv, for this specific matrix leg, to resolve and run against *that* interpreter rather than the project's usual default — the matrix's whole purpose is exercising every supported version, not just the one `.python-version` happens to name.

```mermaid
flowchart TD
    A["Matrix job starts, one leg per Python version"] --> B1["Leg: Python 3.11"]
    A --> B2["Leg: Python 3.12"]
    A --> B3["Leg: Python 3.13"]
    B1 --> C1["uv python install 3.11\nuv sync --locked --python 3.11\nuv run --python 3.11 pytest"]
    B2 --> C2["uv python install 3.12\nuv sync --locked --python 3.12\nuv run --python 3.12 pytest"]
    B3 --> C3["uv python install 3.13\nuv sync --locked --python 3.13\nuv run --python 3.13 pytest"]
    C1 --> D["All three legs must pass\nfor the matrix job to succeed"]
    C2 --> D
    C3 --> D
```

`fail-fast: false` ensures a failure on the 3.11 leg doesn't cancel the still-running 3.12 and 3.13 legs — worth setting deliberately, since the whole point of a version matrix is seeing the *complete* picture of which versions actually fail, not stopping at the first one.

---

## 7. Migration Drift and Smoke-Check Against Ephemeral PostgreSQL

### 7.1 Bridging to the Alembic course's own CI chapter

This section is the uv-specific expression of exactly what the sibling Alembic course's [Chapter 15: CI/CD Integration](../../Databases/alembic-course/15-cicd-integration.md) already teaches in full depth — an ephemeral PostgreSQL service container, `alembic upgrade head` run against it, and `alembic check` catching model/migration drift. The only thing this chapter adds is running every one of those commands through `uv run`, so the same reproducibility guarantee covering `ruff`/`mypy`/`pytest` (Section 5) also covers `alembic` itself and the SQLAlchemy version it runs against.

### 7.2 The job

```yaml
  migration-check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: expenseflow
          POSTGRES_PASSWORD: expenseflow
          POSTGRES_DB: expenseflow_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U expenseflow"
          --health-interval=5s
          --health-timeout=5s
          --health-retries=10
    env:
      DATABASE_URL: postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow_test
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.5.11"
          enable-cache: true

      - name: Install Python
        run: uv python install 3.13

      - name: Sync dependencies
        run: uv sync --locked --package expenseflow-api

      - name: Run migrations against ephemeral Postgres
        working-directory: packages/api
        run: uv run alembic upgrade head

      - name: Check for model/migration drift
        working-directory: packages/api
        run: uv run alembic check

      - name: Verify downgrade paths
        working-directory: packages/api
        run: |
          uv run alembic downgrade base
          uv run alembic upgrade head
```

Everything about *why* this job matters — why `alembic check` catches drift a normal `pytest` run using `Base.metadata.create_all()` would miss, why the upgrade-downgrade-upgrade cycle proves rollback paths actually work, why a PostgreSQL service container (not SQLite) is the right target — is the sibling Alembic course's material, not repeated here. What's specific to *this* course is `uv run` in front of every `alembic` invocation: it guarantees the exact `alembic` and `sqlalchemy` versions this job runs are the same ones `uv.lock` pins, the same ones a developer's own `uv run alembic upgrade head` (Chapter 13) would use locally.

### 7.3 Why this job stays independent of the test-matrix job

`migration-check` deliberately runs against a single, fixed Python version (3.13 — ExpenseFlow's actual production version) rather than participating in Section 6's matrix. Running a full PostgreSQL-backed migration check three times, once per matrix leg, would mostly test whether Alembic and `asyncpg` behave consistently across Python versions — a real but secondary question — at the cost of tripling this job's runtime and its ephemeral-database provisioning overhead for comparatively little additional signal, since the production Python version is the one that actually matters for a real migration run.

---

## 8. The Complete GitHub Actions Workflow

Putting Sections 2 through 7 together, here is ExpenseFlow's full pipeline, run on every pull request and on pushes to `main`:

```yaml
name: ExpenseFlow CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.5.11"
          enable-cache: true
          cache-dependency-glob: "uv.lock"

      - name: Install Python
        run: uv python install 3.13

      - name: Sync dependencies (locked)
        run: uv sync --locked --package expenseflow-api

      - name: Ruff lint
        working-directory: packages/api
        run: uv run ruff check .

      - name: Ruff format check
        working-directory: packages/api
        run: uv run ruff format --check .

      - name: mypy
        working-directory: packages/api
        run: uv run mypy .

  test-matrix:
    needs: [lint-and-typecheck]
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.5.11"
          enable-cache: true

      - name: Install Python ${{ matrix.python-version }}
        run: uv python install ${{ matrix.python-version }}

      - name: Sync dependencies for this Python version
        run: uv sync --locked --python ${{ matrix.python-version }} --package expenseflow-api

      - name: Run unit tests (no database dependency)
        working-directory: packages/api
        run: uv run --python ${{ matrix.python-version }} pytest -v -m "not requires_db"

  migration-check:
    needs: [lint-and-typecheck]
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: expenseflow
          POSTGRES_PASSWORD: expenseflow
          POSTGRES_DB: expenseflow_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U expenseflow"
          --health-interval=5s
          --health-timeout=5s
          --health-retries=10
    env:
      DATABASE_URL: postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow_test
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.5.11"
          enable-cache: true

      - name: Install Python
        run: uv python install 3.13

      - name: Sync dependencies
        run: uv sync --locked --package expenseflow-api

      - name: Run migrations against ephemeral Postgres
        working-directory: packages/api
        run: uv run alembic upgrade head

      - name: Check for model/migration drift
        working-directory: packages/api
        run: uv run alembic check

      - name: Verify downgrade paths
        working-directory: packages/api
        run: |
          uv run alembic downgrade base
          uv run alembic upgrade head

      - name: Full test suite against migrated database
        working-directory: packages/api
        run: uv run pytest -v --cov=app --cov-report=xml

  build-image:
    needs: [test-matrix, migration-check]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build production image
        run: docker build -f packages/api/Dockerfile -t expenseflow-api:${{ github.sha }} .

      - name: Push to registry
        run: echo "docker push expenseflow-api:${{ github.sha }} (registry auth omitted for brevity)"
```

Reading it top to bottom: `lint-and-typecheck` is the fastest gate and runs first, failing loudly on style/type problems before any of the more expensive jobs even start; `test-matrix` and `migration-check` both depend on it and run in parallel, one proving cross-version compatibility, the other proving the migration chain and full test suite work against real PostgreSQL; and `build-image` — [Chapter 14](./14-docker-integration.md)'s Dockerfile, now driven by CI rather than a developer's local `docker build` — only runs once both of those have passed, and only on `main`, so no image ever gets built from code that hasn't cleared every gate.

---

## Real-World Scenario

Priya opens a PR that bumps `sqlalchemy` from `2.0.34` to `2.0.36` in `packages/api/pyproject.toml`, intending to pick up a bugfix, and runs `uv add "sqlalchemy>=2.0.36,<2.1"` locally, which correctly regenerates `uv.lock`. She commits both files together — exactly the discipline Chapter 8 teaches. The PR's `lint-and-typecheck` and `test-matrix` jobs pass cleanly across all three Python versions. But `migration-check` fails, at the `uv run alembic upgrade head` step specifically, with an error somewhere deep in a compiled SQL statement that changed between SQLAlchemy patch versions.

Because `migration-check` runs independently, against real PostgreSQL, rather than being folded into the matrix job, the failure is immediately attributable: it's specifically the migration chain that broke, not the application code, not a lint rule, not a Python-version incompatibility — the error message and the job name together tell Priya exactly where to look before she's read a single line of the traceback. She finds that one of ExpenseFlow's older migrations used a SQLAlchemy Core construct whose exact compiled SQL output shifted subtly between `2.0.34` and `2.0.36` in a way that's harmless for ORM queries (which is why `test-matrix`, running application-level tests, never noticed) but broke a raw `op.execute()` call written to depend on the exact previous output. She adjusts the migration to use a version-stable construct instead, pushes, and `migration-check` passes. Without a dedicated migration-verification job independent of the general test suite, this exact class of bug — a dependency bump that's perfectly fine for the application but subtly wrong for one specific historical migration — would have shipped straight to the `deploy-migration` step against production, exactly the scenario the sibling Alembic course's own CI chapter warns about in the abstract, now caught concretely by this chapter's `migration-check` job.

---

## Best Practices

- Pin an explicit uv version in `astral-sh/setup-uv`, mirroring the same pin used in Chapter 14's Dockerfile, so uv upgrades are a deliberate, reviewed change rather than a silent default shift.
- Enable uv's cache (`enable-cache: true`) and scope its invalidation key to `uv.lock` specifically, so dependency-unrelated commits don't pay a cold-cache tax.
- Always run `uv sync --locked` in CI, never a bare `uv sync` — a bare sync can silently re-resolve and mask a stale lockfile that should have failed the build instead.
- Run `ruff`, `mypy`, and `pytest` through `uv run`, never through an ambient install, so CI is guaranteed to use the exact tool versions every developer's local `uv run` invocations also use.
- Matrix-test across every Python version your project genuinely needs to support, using `uv python install` per leg — don't assume the version on your CI runner's default image is the only one that matters.
- Keep the migration/drift-check job independent of the general test matrix — running it once, against the production Python version, against real PostgreSQL, catches an entirely different class of bug than application-level tests do.
- Gate any image-building or deployment job on both the test matrix and the migration check passing, using `needs:`, so a broken migration chain can never be followed by a deploy that assumes it succeeded.

---

## Common Mistakes

- Running a bare `uv sync` in CI "to keep things simple," silently accepting whatever resolution results instead of failing loudly when `uv.lock` doesn't match `pyproject.toml`.
- Installing `ruff`/`mypy`/`pytest` as separate `pip install` steps in a CI workflow rather than via `uv sync` reading the project's own dev-dependency group, reintroducing exactly the version-drift risk Chapter 11 warned about for global tool installs.
- Forgetting `enable-cache`/an equivalent cache step entirely, paying a full cold-install cost on every single workflow run indefinitely.
- Folding the PostgreSQL-backed migration check into the Python-version test matrix, tripling ephemeral-database provisioning cost for little additional signal, since the production Python version is what actually matters for a real migration run.
- Testing only against the single Python version a developer's laptop happens to have installed, and discovering a version-specific incompatibility only after a consuming service (or a production upgrade) hits it first.
- Building and pushing a Docker image before the migration-check and test-matrix jobs have both passed, rather than gating the build job on `needs: [test-matrix, migration-check]`.
- Pinning uv's version inconsistently between the Dockerfile (Chapter 14) and the CI workflow (this chapter), so a CI run and a local Docker build can silently use different uv resolver/build behavior.

---

## Summary

- A uv-aware CI pipeline for ExpenseFlow verifies five independent things: lockfile honesty, reproducible installs, code-quality gates, cross-Python-version compatibility, and real migration-chain correctness (Section 1).
- `astral-sh/setup-uv` installs a pinned uv version and, with `enable-cache: true`, restores uv's cache between runs, keyed by `uv.lock`'s contents (Sections 2–3).
- `uv sync --locked` — never a bare `uv sync` — is the CI-specific enforcement of Chapter 8's lockfile discipline, failing loudly the instant `uv.lock` and `pyproject.toml` disagree (Section 4).
- `ruff check`, `mypy`, and `pytest` all run through `uv run`, guaranteeing CI uses the exact same tool versions every developer's local invocations do (Section 5).
- Matrix testing across Python 3.11–3.13, using `uv python install` per leg, catches version-specific incompatibilities before a consuming service or a production upgrade does (Section 6).
- A dedicated `migration-check` job, running `uv run alembic upgrade head`/`alembic check`/downgrade-then-upgrade against an ephemeral PostgreSQL service container, bridges directly to the sibling Alembic course's own CI chapter, now expressed through `uv run` for full reproducibility (Section 7).
- The complete workflow chains `lint-and-typecheck` → (`test-matrix` + `migration-check` in parallel) → `build-image`, with each stage gated on the previous ones succeeding (Section 8).

---

## Knowledge Check

1. Why does this chapter's CI workflow use `uv sync --locked` while Chapter 14's Dockerfile uses `uv sync --frozen` — what different failure does each one guard against?
2. What does `enable-cache: true` in `astral-sh/setup-uv` actually restore, and why does scoping its key to `uv.lock` specifically matter?
3. A teammate proposes installing `ruff` and `mypy` via a plain `pip install ruff mypy` step in CI, arguing it's simpler than involving uv. What specific risk does this reintroduce?
4. Why does ExpenseFlow's migration-check job run against a single, fixed Python version rather than participating in the 3.11/3.12/3.13 matrix?
5. Walk through what `uv python install ${{ matrix.python-version }}` does on a matrix leg, and why `--python` needs to be passed explicitly to both `uv sync` and `uv run` in that same job.
6. In the full workflow in Section 8, why does `build-image` depend on both `test-matrix` and `migration-check` rather than just one of them?
7. Explain, using the Real-World Scenario, why a dependency bump that passes `test-matrix` cleanly can still fail `migration-check` — what does that failure reveal about where the bug actually lives?

---

## Hands-On Exercise

**Goal:** Build a working GitHub Actions workflow for a scratch uv project mirroring ExpenseFlow's pipeline, and deliberately trigger the specific failures this chapter's jobs are designed to catch.

1. In a scratch repository with a uv project (a small FastAPI + SQLAlchemy + Alembic setup is enough, single-package is fine for this exercise), add `.github/workflows/ci.yml` implementing the `lint-and-typecheck` job from Section 8, using `astral-sh/setup-uv` with a pinned version and `enable-cache: true`.
2. Push a commit that hand-edits `pyproject.toml` to add a new dependency, without running `uv lock` afterward. Open a PR and confirm `uv sync --locked` fails, and read its error output carefully to see exactly what it reports as out of date.
3. Run `uv lock` locally, commit the regenerated `uv.lock`, push again, and confirm the same step now passes.
4. Add the `test-matrix` job, and confirm all three Python-version legs run and report results independently (`fail-fast: false`) — try introducing a Python-3.11-only incompatibility (e.g., a `match` statement feature that behaves differently, or a typing construct only valid in later versions) and confirm exactly one leg fails while the others pass.
5. Add the `migration-check` job with a `postgres:16` service container, and confirm `uv run alembic upgrade head` succeeds against it.
6. Deliberately edit a SQLAlchemy model without writing the corresponding migration, push, and confirm `uv run alembic check` fails the build with a clear drift message.
7. Add the `build-image` job with `needs: [test-matrix, migration-check]`, and confirm via the Actions UI that it never becomes eligible to run until both dependency jobs have completed successfully.

---

## Further Reading

- [astral-sh/setup-uv GitHub Action](https://github.com/astral-sh/setup-uv) — full configuration reference for the action used throughout this chapter.
- [uv Guides — GitHub Actions / CI](https://docs.astral.sh/uv/guides/) — Astral's own official CI integration guide, including cache configuration and matrix-testing patterns.
- [uv Reference — CLI](https://docs.astral.sh/uv/reference/) — full flag reference for `uv sync`, `uv python install`, and `uv run`, including `--locked`/`--frozen`/`--python`.
- [This repo's Alembic course, Chapter 15: CI/CD Integration](../../Databases/alembic-course/15-cicd-integration.md) — the full migration-drift, ephemeral-database, and rollback-testing material this chapter's Section 7 builds directly on top of.
- [PyPI Trusted Publishers (OIDC publishing)](https://docs.pypi.org/trusted-publishers/) — relevant to [Chapter 16](./16-publishing-packages.md)'s release-gated publish workflow, a natural extension of this chapter's `build-image` job.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-docker-integration.md">← Previous: Docker Integration</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-publishing-packages.md">Next: Publishing Packages →</a>
</div>
