# CI/CD Integration

Chapter 14 built the discipline of expand/contract, safe-operation classification, and lock-aware DDL by hand, deploy by deploy, trusting engineers to follow the sequence correctly every time. That trust doesn't scale — the entire point of a CI/CD pipeline is to make the safe path the automatic path, and to catch the unsafe one before a human has to. This chapter takes everything from Chapters 1 through 14 and asks: what should actually run in a pipeline, automatically, on every pull request and every merge, so that a migration reaches production only after proving it applies cleanly, matches the models it's supposed to represent, and can be reversed if something goes wrong? ExpenseFlow's GitHub Actions pipeline is the concrete answer this chapter builds, piece by piece, ending in a complete workflow file you could adapt directly.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure an ephemeral PostgreSQL instance in CI (via GitHub Actions services or testcontainers) and run `alembic upgrade head` against it as part of a pipeline.
- Implement a migration/model drift check that fails the build when models and migrations have diverged.
- Explain why running migrations as part of container startup is dangerous for multi-instance deployments, and describe the alternative of a dedicated release step.
- Write a CI job that proves downgrade paths work by upgrading, downgrading, and upgrading again.
- Assemble a complete GitHub Actions workflow for ExpenseFlow covering migration validation, drift detection, tests, and staged deployment.

---

## Prerequisites for This Chapter

- Running `alembic upgrade`/`downgrade` and understanding why downgrade paths matter — [Chapter 6: Upgrade & Downgrade](./06-upgrade-and-downgrade.md).
- Autogenerate's comparison of live database state against `target_metadata` — [Chapter 7: Autogenerate Migrations](./07-autogenerate-migrations.md), since the drift check in this chapter (Section 3) is built directly on that same comparison machinery.
- SQLite vs. ephemeral PostgreSQL trade-offs for testing migrations — [Chapter 13: SQLite & Batch Migrations](./13-sqlite-batch-migrations.md), whose conclusion (favor real PostgreSQL for migration validation) this chapter now implements in CI.
- The expand/contract pattern and why deploy timing matters — [Chapter 14: Zero-Downtime Migrations & Production Deployment](./14-zero-downtime-migrations.md), since Section 4 of this chapter is about not undoing that discipline at the infrastructure level.

---

## 1. What CI Actually Needs to Verify

Before writing any YAML, it's worth being explicit about what a migration-aware CI pipeline is trying to prove, because it's easy to build a pipeline that runs green while still missing the failure modes that actually cause incidents. There are four distinct claims a good pipeline should verify, each independent of the others:

1. **The migration chain applies cleanly against a real PostgreSQL instance**, from whatever a fresh database looks like all the way to `head` — not just against SQLite (Chapter 13), and not just "it applied on someone's laptop once."
2. **The migration chain's end state matches what the SQLAlchemy models currently describe** — no drift where a developer edited a model and forgot to generate/write the corresponding migration, or vice versa.
3. **The application's test suite passes against a database that actually went through the real migration chain**, not a database built by `Base.metadata.create_all()`, which bypasses migrations entirely and can hide problems only the migration path itself would expose.
4. **Every migration's `downgrade()` actually works**, not just its `upgrade()` — a downgrade path that's never been executed is, in practice, unverified code, and Chapter 6 already warned this is exactly the kind of thing teams neglect.

The remaining sections build a CI job answering each of these in turn, and Section 6 assembles them into one workflow.

---

## 2. Ephemeral PostgreSQL in CI

### 2.1 GitHub Actions service containers

The simplest way to get a real, disposable PostgreSQL instance in a GitHub Actions job is a **service container** — GitHub Actions starts it alongside your job's main container, on the same Docker network, and tears it down automatically when the job finishes:

```yaml
jobs:
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
```

The `options` health check matters more than it looks — without it, your job's steps might start running against a container that's still initializing, and the very first `alembic upgrade head` in the pipeline fails with a connection-refused error that has nothing to do with your migrations and everything to do with a race the health check exists specifically to prevent.

### 2.2 `testcontainers-python` as an alternative

Some teams prefer starting the database from inside the test process itself, using [`testcontainers-python`](https://testcontainers-python.readthedocs.io/), which gives finer control (a fresh container per test module, programmatic access to the exact connection URL) at the cost of needing Docker-in-Docker or a Docker socket available to the CI runner:

```python
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16") as postgres:
        yield postgres


@pytest.fixture(scope="session")
def database_url(postgres_container):
    return postgres_container.get_connection_url().replace("psycopg2", "asyncpg")
```

Both approaches are valid; GitHub Actions service containers are the lower-friction default for a straightforward pipeline like ExpenseFlow's, while `testcontainers` earns its keep when tests need multiple isolated database instances within a single job, or when a team wants their local `pytest` run and their CI run to provision the database identically.

### 2.3 Running the migration chain itself

With the service container up, the actual verification step is close to what a developer runs locally:

```yaml
      - name: Run Alembic migrations against ephemeral Postgres
        env:
          DATABASE_URL: postgresql+psycopg://expenseflow:expenseflow@localhost:5432/expenseflow_test
        run: |
          alembic upgrade head
```

If this step fails, it means the migration chain itself is broken against real PostgreSQL — a syntax error, a missing dependency between revisions, an operation that only ever worked by accident against SQLite. Catching this on every PR, against a database that's thrown away immediately afterward, is strictly cheaper than catching it during a production deploy.

---

## 3. Detecting Migration/Model Drift

### 3.1 What drift means, concretely

Drift is what happens when the SQLAlchemy models (`target_metadata`) and the actual database schema produced by running every migration to `head` no longer agree — most commonly because a developer changed a model (added a column, changed a type) and either forgot to write the corresponding migration, or wrote one that doesn't fully capture the change. Left undetected, drift means autogenerate's comparison (Chapter 7) is silently wrong going forward, and worse, it means the schema your tests ran against never actually matched what your ORM thinks it's talking to.

### 3.2 `alembic check`

Modern Alembic versions (1.9+) ship a purpose-built command for exactly this: `alembic check` runs the same comparison autogenerate uses internally, but instead of generating a new migration script, it simply reports whether any difference exists between the current database (after applying every existing migration) and `target_metadata`, exiting non-zero if there is one:

```yaml
      - name: Check for migration/model drift
        env:
          DATABASE_URL: postgresql+psycopg://expenseflow:expenseflow@localhost:5432/expenseflow_test
        run: |
          alembic upgrade head
          alembic check
```

A failing `alembic check` means exactly one thing: someone changed a model without writing (or fully writing) the migration for it. The fix is always to go write the missing migration — never to suppress or ignore the check, since ignoring it just moves the discovery of the mismatch from "a PR check today" to "a production incident once the missing schema change is actually needed."

### 3.3 The manual fallback

For Alembic versions predating `alembic check`, or for teams wanting the same idea with more visibility into what's different, the manual equivalent is to run `--autogenerate` against the post-migration database and assert nothing meaningful comes out:

```bash
alembic upgrade head
alembic revision --autogenerate -m "drift-check-scratch" --rev-id drift_check_scratch
# Inspect the generated file: it should contain an empty upgrade()/downgrade() pair.
# A non-trivial diff here means drift exists.
git diff --exit-code alembic/versions/*drift_check_scratch*.py || {
  echo "Drift detected between models and migrations"; exit 1;
}
rm -f alembic/versions/*drift_check_scratch*.py
```

This is more brittle than `alembic check` (it depends on parsing a generated file rather than a structured pass/fail result) and is worth replacing with `alembic check` if your Alembic version supports it — it's included here mainly so the underlying mechanism (autogenerate's comparison, reused for verification rather than generation) is clear rather than feeling like a different tool entirely.

---

## 4. Migrations as a Release Step vs. Container Startup

### 4.1 The tempting pattern: migrate on startup

A common pattern, especially in smaller projects, is to run `alembic upgrade head` directly inside the application container's entrypoint, before starting the actual server process:

```bash
#!/bin/sh
# entrypoint.sh
set -e
alembic upgrade head
exec uvicorn app.main:app --host 0.0.0.0 --port 8000
```

This is appealingly simple — one container, one command, migrations always applied before the app starts serving. It works fine for a single-instance deployment or a low-traffic side project. It becomes actively dangerous the moment you run more than one instance of the application simultaneously, which is the normal case for anything running a rolling deploy across multiple replicas (exactly ExpenseFlow's production setup).

### 4.2 Why concurrent startup migrations are dangerous

During a rolling deploy, Kubernetes (or ECS, or any comparable orchestrator) doesn't replace instances atomically — it starts new-version instances while old-version ones are still running, then terminates the old ones once the new ones report healthy. If migrations run inside each container's startup, **every new instance that starts during the rollout attempts to run the migration chain independently and concurrently**:

```mermaid
sequenceDiagram
    participant P1 as Pod 1 (new version, starting)
    participant P2 as Pod 2 (new version, starting)
    participant P3 as Pod 3 (new version, starting)
    participant DB as Database

    par All three pods start roughly simultaneously
        P1->>DB: alembic upgrade head
        P2->>DB: alembic upgrade head
        P3->>DB: alembic upgrade head
    end
    Note over DB: Multiple concurrent DDL transactions against<br/>the same tables and the alembic_version row —<br/>contention, wasted work, and a window where some<br/>pods serve traffic against a partially-migrated schema
```

Several concrete problems fall out of this, beyond the obvious wastefulness of running the same migration three times:

- **Lock contention between the concurrent migration attempts themselves.** Two instances racing to run the same `ALTER TABLE` can deadlock or serialize behind each other, adding unpredictable startup latency — directly undermining the very rolling-deploy speed the orchestrator is trying to give you.
- **A window where already-started, already-healthy instances serve traffic against a schema mid-migration.** If the migration chain has more than one revision to apply, an instance that finished migrating and started serving traffic might be running against a schema state that a *slower* concurrent migration attempt on another instance hasn't reached yet, or is actively altering — precisely the coordination hazard Chapter 14 spent an entire chapter teaching you to avoid deliberately, now reintroduced accidentally by infrastructure design.
- **No single, observable point of truth for "did the migration succeed."** With one migration attempt per pod, a failure on one pod (but not others) is easy to miss in aggregate health dashboards, and retries are entangled with the pod's own restart/backoff behavior rather than being a distinct, inspectable step.
- **`alembic_version` contention.** Alembic's own bookkeeping row update at the end of a migration run is itself a write to a shared table; concurrent instances contending for that update add yet another source of timing-dependent behavior at exactly the moment you want deployments to be predictable.

### 4.3 The safer pattern: a dedicated migration step

The fix is architectural: decouple "run the migration" from "start an application replica" entirely, so the migration runs **exactly once**, before any new application replica starts, as its own distinct step in the deployment pipeline — a Kubernetes `Job`, an ECS one-off task, or (for platforms without a native Job primitive) a dedicated CI/CD pipeline step that runs `alembic upgrade head` and only proceeds to the deployment stage if it exits successfully:

```mermaid
flowchart LR
    A["CI/CD pipeline:\nbuild + test pass"] --> B["Release step:\nrun alembic upgrade head\nONCE, against production DB"]
    B -->|success| C["Deploy new app replicas\n(rolling, N instances)\nno migration logic inside them"]
    B -->|failure| D["Pipeline halts —\nno new replicas deployed"]
```

This single change removes every problem in Section 4.2 at once: there is exactly one migration attempt, its success or failure is a first-class, individually observable pipeline step, and no application replica ever starts until the schema it's about to run against is already fully settled — which is precisely the ordering expand/contract (Chapter 14) assumes when it says "the migrate phase's app release can safely assume the expand migration has already been applied everywhere."

### 4.4 A GitHub Actions release-step example

```yaml
  deploy-migration:
    needs: [migration-check, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - name: Run migration against production database
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: alembic upgrade head

  deploy-app:
    needs: [deploy-migration]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Roll out new application replicas
        run: echo "trigger rolling deploy of app containers (kubectl/ecs/etc.)"
```

The `needs: [deploy-migration]` dependency on the `deploy-app` job is the entire mechanism enforcing the ordering: no application replica rollout starts until the migration job has completed successfully, and if it fails, the workflow simply stops there — no partial, racing rollout ever begins.

---

## 5. Rollback Testing in CI

### 5.1 Why this matters, briefly revisited

Chapter 6 introduced the discomforting fact that most teams write `downgrade()` functions and never actually run them, which means a downgrade path is, in practice, unverified code that happens to look plausible. CI is the natural place to close that gap cheaply, since the database being tested against is disposable — there's no cost to actually exercising a downgrade the way there would be against production.

### 5.2 The upgrade → downgrade → upgrade cycle

```yaml
      - name: Verify downgrade paths work
        env:
          DATABASE_URL: postgresql+psycopg://expenseflow:expenseflow@localhost:5432/expenseflow_test
        run: |
          alembic upgrade head
          alembic downgrade base
          alembic upgrade head
```

This proves two things at once: every migration's `downgrade()` executes without error all the way back to an empty database, and — just as importantly — the *second* `upgrade head` proves the `upgrade()` functions are themselves idempotent-in-sequence and don't depend on some artifact left behind by a downgrade that didn't fully clean up (a stray index, a leftover enum type, a sequence that wasn't reset). Teams with a long migration history sometimes limit this to the most recent N revisions (`alembic downgrade -5` then `alembic upgrade head`) rather than all the way to `base`, purely to keep the CI job's runtime reasonable — the principle matters more than testing literally every historical revision on every single PR.

### 5.3 `pytest-alembic` as a more structured alternative

[`pytest-alembic`](https://pytest-alembic.readthedocs.io/) packages this idea (and several related checks — that every migration is reachable, that there's only a single head, that upgrade/downgrade round-trips cleanly) as a set of ready-made `pytest` tests you can drop into a project with a small amount of fixture wiring, rather than hand-rolling the shell-script version above. For a team wanting the same guarantees expressed as ordinary test functions (with the reporting, parallelization, and failure output `pytest` already gives you), `pytest-alembic` is worth adopting directly — [Chapter 18](./18-tools-and-ecosystem.md) covers it, and the wider migration tooling ecosystem, in full.

---

## 6. A Full GitHub Actions Workflow for ExpenseFlow

Putting Sections 2 through 5 together, here is the complete pipeline ExpenseFlow runs on every pull request and on merges to `main`:

```yaml
name: ExpenseFlow CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
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
      DATABASE_URL: postgresql+psycopg://expenseflow:expenseflow@localhost:5432/expenseflow_test
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run migrations against ephemeral Postgres
        run: alembic upgrade head

      - name: Check for model/migration drift
        run: alembic check

      - name: Verify downgrade paths
        run: |
          alembic downgrade base
          alembic upgrade head

  test:
    needs: [migration-check]
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
      DATABASE_URL: postgresql+psycopg://expenseflow:expenseflow@localhost:5432/expenseflow_test
      TEST_DATABASE_URL: sqlite+aiosqlite:///:memory:
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Migrate the Postgres test database
        run: alembic upgrade head

      - name: Run test suite
        run: pytest -v --cov=app

  deploy-migration:
    needs: [migration-check, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run migration against production database
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: alembic upgrade head

  deploy-app:
    needs: [deploy-migration]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Trigger rolling deployment of application containers
        run: echo "kubectl rollout restart deployment/expenseflow-api"
```

Reading it top to bottom: `migration-check` proves the chain runs against real PostgreSQL, matches the models, and rolls back cleanly, all against a throwaway database; `test` proves the application itself behaves correctly on top of a genuinely migrated schema; and only after both pass does `deploy-migration` run the migration exactly once against the real production database, gating `deploy-app` so that no application replica rollout begins until the schema it depends on is already fully in place — the CI/CD-level enforcement of the ordering Section 4.3 argued for architecturally.

---

## Real-World Scenario

A new engineer on ExpenseFlow's team opens a PR that adds a `monthly_budgets.rollover_enabled` boolean column directly to the SQLAlchemy model, intending to write the migration "in a follow-up commit" later that day, and forgets. The `test` job passes cleanly — the test suite's SQLite fixture builds tables straight from `Base.metadata.create_all()` (Chapter 13, Section 1), which reflects the model's new column with no migration required at all, so nothing in the ordinary test run notices anything is missing. Without the `migration-check` job's `alembic check` step, this PR would merge, and the very next production migration run would leave the production database's `monthly_budgets` table missing a column the application code now assumes exists — a drift bug that would only surface the moment some code path actually touched `rollover_enabled`, potentially days later and in production rather than in review.

Instead, `migration-check`'s `alembic check` step fails the PR immediately, with a clear message identifying exactly the missing column, before the `test` job even needs to run (the `needs: [migration-check]` dependency stops the pipeline there). The engineer writes the missing migration, pushes an update, and the same job now passes — a fifteen-minute delay in review instead of a production incident discovered by an on-call engineer at an inconvenient hour. This is precisely the class of bug Section 3's drift check exists to catch mechanically, rather than relying on code review to notice a missing file in a diff.

---

## Best Practices

- Run the full migration chain against a real, ephemeral PostgreSQL instance in CI, not just SQLite — Chapter 13's conclusion made concrete as an actual pipeline job (Section 2).
- Add a drift check (`alembic check` or its manual equivalent) as a required, blocking CI step on every pull request, not an occasional manual audit (Section 3).
- Never run `alembic upgrade head` inside an application container's startup entrypoint for any multi-instance deployment — use a dedicated release step or Job that runs exactly once before any new replica starts (Section 4).
- Prove downgrade paths actually work in CI with an upgrade → downgrade → upgrade cycle, rather than trusting `downgrade()` functions that have never been executed (Section 5).
- Gate application deployment on migration success at the pipeline level (`needs:` in GitHub Actions, or the equivalent dependency mechanism in your CI system), so a failed migration cannot be followed by a rollout of code that assumes it succeeded (Section 4.4).
- Keep the migration-verification jobs fast by scoping downgrade testing to a reasonable window (recent N revisions) once the migration history grows long, rather than replaying the entire history on every PR.

---

## Common Mistakes

- Relying on a SQLite-only CI job as proof that migrations are production-safe, missing everything Chapter 13's trade-off table already flagged as SQLite's blind spots (Section 2).
- Testing against a database built with `Base.metadata.create_all()` instead of the real migration chain, which hides exactly the kind of model/migration drift Section 3's check exists to catch (Real-World Scenario).
- Baking `alembic upgrade head` into a Docker container's entrypoint and discovering only after scaling past one replica that concurrent migration attempts race, contend for locks, and create a window of partially-migrated schema visible to already-started instances (Section 4.2).
- Writing `downgrade()` functions and never running them anywhere, including in CI — leaving a rollback path that looks complete in code review but has never actually executed (Section 5.1).
- Treating a failing drift check as something to work around (deleting the check, marking the job non-blocking) rather than as a signal that a migration is genuinely missing.
- Deploying application code and running its corresponding migration in the same, undifferentiated step, rather than as two separately gated pipeline stages — reintroducing the exact ordering hazard the expand/contract pattern (Chapter 14) depends on not happening.

---

## Summary

- A migration-aware CI pipeline should verify four independent things: the chain applies against real PostgreSQL, models and migrations agree, the test suite passes against a genuinely migrated schema, and downgrade paths actually work (Section 1).
- GitHub Actions service containers (or `testcontainers-python`) provide a real, disposable PostgreSQL instance for CI, addressing Chapter 13's conclusion that SQLite alone isn't a faithful migration test target (Section 2).
- `alembic check` (or its manual autogenerate-diff equivalent) catches model/migration drift automatically and should be a required, blocking check (Section 3).
- Running migrations inside application container startup is dangerous once more than one replica exists — concurrent migration attempts race, contend for locks, and can expose already-started instances to a partially-migrated schema; a dedicated, single-run release step avoids all of this (Section 4).
- An upgrade → downgrade → upgrade cycle in CI is the cheapest way to prove downgrade paths genuinely work, rather than trusting untested code (Section 5).
- ExpenseFlow's full pipeline chains `migration-check` → `test` → `deploy-migration` → `deploy-app`, with each stage gated on the previous one succeeding, enforcing migration-before-rollout ordering at the infrastructure level (Section 6).

---

## Knowledge Check

1. Name the four distinct things a migration-aware CI pipeline should verify, and explain why testing the migration chain against SQLite alone would leave at least one of them unverified.
2. What does `alembic check` actually compare, and what specific kind of bug does it catch that a normal test suite run (using `Base.metadata.create_all()`) would miss?
3. Explain, step by step, what goes wrong when three application replicas each run `alembic upgrade head` inside their own container startup during a simultaneous rolling deploy.
4. Why does an upgrade → downgrade → upgrade cycle prove more than an upgrade → downgrade cycle alone?
5. In the full GitHub Actions workflow in Section 6, why does `deploy-app` depend on `deploy-migration` rather than running in parallel with it?
6. A teammate proposes removing the drift check because "it's redundant with tests, and it's slowing down our PRs." How would you respond, using the Real-World Scenario as a concrete counter-example?
7. What operational alternative to a container-startup migration would you propose for a team currently doing that, and what specific class of incident would it prevent?

---

## Hands-On Exercise

**Goal:** Build a working GitHub Actions workflow for a scratch project that mirrors ExpenseFlow's pipeline, and deliberately trigger both the drift check and the container-startup race to see the failure modes this chapter describes.

1. In a scratch repository with Alembic and a couple of SQLAlchemy models, add a GitHub Actions workflow file at `.github/workflows/ci.yml` implementing the `migration-check` job from Section 6, using a `postgres:16` service container.
2. Push a commit that adds a new nullable column to a model without writing the corresponding migration. Open a PR and confirm the `alembic check` step fails, and read its output carefully to see exactly what it reports as missing.
3. Write the missing migration, push again, and confirm the same job now passes.
4. Add the `test` job from Section 6, wiring `needs: [migration-check]`, and confirm it only runs after `migration-check` succeeds — try temporarily breaking `migration-check` and observe that `test` never starts.
5. Locally, simulate the container-startup race from Section 4.2: open two terminal sessions against the same fresh PostgreSQL database, and run `alembic upgrade head` in both at nearly the same moment (script it with `&` to background one of them). Observe what happens — depending on timing, one instance may complete first while the other encounters a lock wait or a "duplicate object" error on a `CREATE` statement, reproducing the exact contention Section 4.2 describes.
6. Add the `deploy-migration`/`deploy-app` job pair from Section 6 (pointing at a throwaway target instead of real production for this exercise), and confirm via the Actions UI that `deploy-app` only becomes eligible to run once `deploy-migration` has completed successfully.

---

## Further Reading

- [Alembic Official Documentation](https://alembic.sqlalchemy.org/en/latest/) — the `alembic check` command and general CLI reference used throughout this chapter.
- [Alembic Cookbook](https://alembic.sqlalchemy.org/en/latest/cookbook.html) — includes CI-oriented recipes referenced in Sections 2 and 5.
- [`pytest-alembic`](https://pytest-alembic.readthedocs.io/) — structured `pytest`-based migration testing covering round-trip and drift checks, previewed in Section 5.3 and covered fully in Chapter 18.
- [Alembic GitHub Repository](https://github.com/sqlalchemy/alembic) — for tracking `alembic check` behavior across versions and reviewing real-world CI configurations in the issue tracker.
- [FastAPI SQL Databases Guide](https://fastapi.tiangolo.com/tutorial/sql-databases/) — background on the application/database wiring this chapter's pipeline is built to validate end to end.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./14-zero-downtime-migrations.md">← Previous: Zero-Downtime Migrations & Production Deployment</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./16-best-practices.md">Next: Best Practices →</a>
</div>
