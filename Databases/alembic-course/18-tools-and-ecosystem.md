# Tools & Ecosystem

[Chapter 17](./17-common-mistakes-and-pitfalls.md) closed the "here's exactly how this goes wrong" arc with seven concrete failure modes. This chapter shifts from *what to avoid* to *what else exists* — the tools and community packages that extend Alembic beyond what it does out of the box, and the broader landscape it sits in. Alembic's core is deliberately narrow: it versions the SQL schema it's told about, using the `op.*` directives covered in Chapter 8. It has no built-in opinion about Postgres views, functions, or triggers; no first-class enum-diffing support; no test framework of its own; and no opinion about how it fits into a container's startup sequence. The ecosystem around Alembic fills exactly these gaps. This chapter surveys the tools ExpenseFlow's team would actually reach for as the project matures, and closes with an honest comparison of Alembic against the migration tools you'd meet in other ecosystems — Django, Flyway, Liquibase — so the concepts you've learned here transfer cleanly if you ever work in one of those instead.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what `alembic-utils` adds on top of core Alembic, and version a Postgres view, function, or trigger the same disciplined way you version a table.
- Describe the enum-diffing gap in core Alembic's autogenerate and how community packages close it.
- Write and run a `pytest-alembic` test suite that verifies migration correctness independent of application-level tests.
- Integrate Alembic into a FastAPI application's lifecycle and a Docker container's entrypoint sequence correctly, without reintroducing Chapter 15's concurrent-migration race.
- Use `sqlacodegen` to reverse-engineer SQLAlchemy models from an existing database, and know when that's the right tool for the job.
- Place Alembic accurately relative to Django migrations, Flyway, and Liquibase in a system-design or tooling discussion.

---

## Prerequisites for This Chapter

This chapter assumes working familiarity with:

- The `op.*` directive catalog and manual migrations (**[Chapter 8: Writing Manual Migrations](./08-writing-manual-migrations.md)**)
- PostgreSQL-specific DDL — ENUM types, extensions, `op.execute()` (**[Chapter 12: PostgreSQL-Specific Features](./12-postgresql-specific-features.md)**)
- `env.py`'s structure and the async-engine handling (**[Chapter 4: env.py & alembic.ini](./04-migration-environment-env-py.md)**)
- CI/CD pipeline design, ephemeral Postgres, and drift detection (**[Chapter 15: CI/CD Integration](./15-cicd-integration.md)**)
- The consolidated best-practices and common-mistakes chapters (**[Chapter 16](./16-best-practices.md)**, **[Chapter 17](./17-common-mistakes-and-pitfalls.md)**)

None of the tools in this chapter are prerequisites for anything earlier in the course — they're additions you reach for once the core workflow is solid, not replacements for it.

A quick map of what's ahead, and the specific gap in core Alembic each tool addresses:

| Tool | Gap it fills | Section |
|---|---|---|
| `alembic-utils` | No versioning for views, functions, or triggers | 1 |
| Enum-diffing packages | Autogenerate's limited detection of Postgres `ENUM` value changes | 2 |
| `pytest-alembic` | No built-in test framework for migration correctness properties | 3 |
| FastAPI/Docker patterns | No opinion on how migrations fit into a container's startup sequence | 4 |
| `sqlacodegen` | No reverse path from an existing database back to SQLAlchemy models | 5 |

---

## 1. Versioning Views, Functions, and Triggers with `alembic-utils`

Core Alembic's autogenerate only knows about the SQLAlchemy `Table` objects registered on `target_metadata` (Chapter 4/7) — it has no concept of a Postgres view, stored function, or trigger, because none of those map to a SQLAlchemy ORM construct. In practice, most non-trivial applications eventually want at least one of these: ExpenseFlow's team, for instance, wants a `monthly_expense_summary` view that pre-aggregates spending per user per month for a dashboard, without duplicating that aggregation logic in application code.

Without a dedicated tool, a view like this ends up as a raw `op.execute("CREATE VIEW ...")` call in a migration — which works, but autogenerate can never detect drift in it, and changing the view means writing a new migration that drops and recreates it by hand, with no diffing support at all.

[`alembic-utils`](https://github.com/olirice/alembic_utils) closes this gap by giving views, functions, and triggers first-class, versioned "entity" objects that plug into Alembic's autogenerate machinery the same way a `Table` does — including diffing an entity's current definition against what's live in the database and generating the `CREATE OR REPLACE`/`DROP` migration code automatically.

```python
# app/db/views.py
from alembic_utils.pg_view import PGView

monthly_expense_summary = PGView(
    schema="public",
    signature="monthly_expense_summary",
    definition="""
        SELECT
            user_id,
            date_trunc('month', expense_date) AS month,
            sum(amount_cents) AS total_cents
        FROM expenses
        GROUP BY user_id, date_trunc('month', expense_date)
    """,
)
```

```python
# alembic/env.py — register the entity alongside target_metadata
from alembic_utils.replaceable_entity import register_entities
from app.db.views import monthly_expense_summary

target_metadata = Base.metadata
register_entities([monthly_expense_summary])
```

With the entity registered, `alembic revision --autogenerate` now detects changes to the view's definition the same way it detects a changed column — editing the `definition` string and re-running autogenerate produces a migration that issues the correct `CREATE OR REPLACE VIEW`. The same pattern extends to `PGFunction` and `PGTrigger` for stored procedures and triggers.

**Caveat, consistent with Chapter 16's review discipline:** treat `alembic-utils`-generated migrations with the same "review before committing" rule as any other autogenerated output — a view definition change that looks trivial in a diff can still change query semantics (a `GROUP BY` column added or removed) in ways worth a second look.

---

## 2. Enum Diffing with Community Packages

Chapter 12 covered native Postgres `ENUM` types and their awkward migration story: core Alembic's autogenerate historically has limited support for detecting changes to an enum's *values* (adding, removing, or reordering members) the way it detects changes to a column's type or nullability. Adding a new value to a Postgres enum also has its own transaction-block caveat (`ALTER TYPE ... ADD VALUE` cannot run inside the same transaction as other statements that use the new value, in older Postgres versions), which is easy to get wrong when writing it by hand.

Community packages in the spirit of `alembic_postgresql_enum` address this specifically: they hook into Alembic's autogenerate comparison process to detect enum value changes (additions, removals, renames) and generate the correct sequence of `ALTER TYPE` statements — including handling the transaction-block restriction correctly — rather than requiring you to hand-write `op.execute("ALTER TYPE ...")` calls and get the sequencing right yourself every time.

```python
# Without enum-diffing support: adding a new category enum value
# requires a hand-written op.execute, easy to get the transaction
# semantics wrong on
def upgrade() -> None:
    op.execute("COMMIT")  # exit the implicit migration transaction first
    op.execute("ALTER TYPE category_enum ADD VALUE 'subscriptions'")
```

With an enum-diffing package registered in `env.py`, this becomes an ordinary autogenerate detection: changing the Python `Enum` definition and re-running `--autogenerate` produces the correct migration automatically, the same way adding a table column does.

```python
# app/models/expense.py — before: a plain string column (Chapter 2-6 era)
category: Mapped[str] = mapped_column(sa.String, nullable=False)

# app/models/expense.py — after: a native Postgres ENUM (Chapter 12)
class CategoryEnum(str, enum.Enum):
    food = "food"
    transport = "transport"
    utilities = "utilities"
    subscriptions = "subscriptions"  # new value added later

category: Mapped[CategoryEnum] = mapped_column(
    sa.Enum(CategoryEnum, name="category_enum"), nullable=False
)
```

With an enum-diffing package wired into `env.py`'s comparison hooks, adding `subscriptions` to `CategoryEnum` and re-running `--autogenerate` generates the correct `ALTER TYPE category_enum ADD VALUE 'subscriptions'` migration — including the transaction-block handling — without anyone needing to remember the raw SQL syntax or its caveats. As with `alembic-utils`, evaluate any such package's maintenance status and pin its version deliberately (Chapter 16) before depending on it in a production migration path — this corner of the ecosystem moves independently of Alembic's own release cadence.

---

## 3. Testing Migrations with `pytest-alembic`

Chapter 15's CI pipeline runs `alembic upgrade head` and a downgrade check as shell steps — sufficient for the basics, but not integrated with the rest of the test suite, and not expressive enough to test properties like "every migration has a non-empty downgrade" or "the migration history has no more than one head" as first-class, individually reported test cases.

[`pytest-alembic`](https://pytest-alembic.readthedocs.io/) provides a set of ready-made pytest tests covering exactly these properties, plus a fixture-based way to write your own migration-specific tests (e.g., "after this specific migration runs, does the backfilled `currency` column actually contain no NULLs?").

```python
# conftest.py
import pytest
from pytest_alembic import runner
from pytest_alembic.config import Config

@pytest.fixture
def alembic_config():
    return Config(config_options={"file": "alembic.ini"})

@pytest.fixture
def alembic_engine():
    from sqlalchemy import create_engine
    return create_engine("postgresql+psycopg2://expenseflow:pw@localhost/expenseflow_test")
```

```bash
# Runs pytest-alembic's built-in suite: single head, upgrade/downgrade
# round-trips cleanly, no missing down_revision, model matches migrations
pytest --test-alembic
```

```python
# A custom test using pytest-alembic's runner fixture to verify a
# specific data migration's actual effect, not just that it ran without error
def test_currency_backfill_leaves_no_nulls(alembic_runner, alembic_engine):
    alembic_runner.migrate_up_to("c7f1a2b3d4e5")
    with alembic_engine.connect() as conn:
        result = conn.execute(sa.text("SELECT count(*) FROM expenses WHERE currency IS NULL"))
        assert result.scalar() == 0
```

This directly strengthens two of Chapter 16's practices: "test the upgrade/downgrade cycle" and "test against realistic conditions" both become concrete, individually-failing pytest cases instead of a single pass/fail shell step — a broken downgrade now names itself in the test report instead of showing up as one generic pipeline failure.

---

## 4. Integrating Alembic with FastAPI's Lifecycle and Docker Entrypoints

Chapter 15 established the principle: migrations run as a single, dedicated release step before the application rolls out, never as a race between concurrently starting app containers (Chapter 17, Mistake #5, covers exactly what goes wrong if you don't). This section makes that concrete for a containerized FastAPI deployment.

**What not to do** — running migrations inside FastAPI's own startup event, where every replica does it on every restart:

```python
# Anti-pattern: every FastAPI replica tries to run migrations on its own
# startup — races under multiple replicas, silently reintroduces
# Chapter 17's Mistake #5 in reverse (code starting before migration finishes)
@app.on_event("startup")
async def run_migrations():
    from alembic.config import Config
    from alembic import command
    command.upgrade(Config("alembic.ini"), "head")
```

This "works" with exactly one replica and breaks the moment you scale to two, because both replicas attempt `alembic upgrade head` concurrently — Alembic takes a lock on `alembic_version` during the upgrade, so one replica typically wins and the other either waits or errors, and which one serves traffic first becomes non-deterministic.

**The correct pattern** — a Docker entrypoint script or a dedicated Kubernetes Job/init container runs the migration exactly once, before any application replica starts:

```bash
#!/bin/sh
# docker-entrypoint.sh — runs once, as a separate step, before uvicorn starts
set -e
echo "Running migrations..."
alembic upgrade head
echo "Migrations complete, starting application..."
exec uvicorn app.main:app --host 0.0.0.0 --port 8000
```

```yaml
# docker-compose.yml excerpt: a dedicated one-shot migration service
# that the app service depends on completing successfully
services:
  migrate:
    build: .
    command: alembic upgrade head
    depends_on:
      - postgres

  app:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000
    depends_on:
      migrate:
        condition: service_completed_successfully
```

In a Kubernetes deployment, the equivalent is a `Job` (or an init container on the first replica only, though a dedicated `Job` is the safer default) that runs `alembic upgrade head` as part of the release pipeline, gating the deployment's rollout — the same shape as Chapter 15's CI job, extended into the actual production release process.

```yaml
# Kubernetes: a Job that must complete successfully before the
# Deployment's rollout proceeds, mirroring the docker-compose pattern above
apiVersion: batch/v1
kind: Job
metadata:
  name: expenseflow-migrate
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: expenseflow:latest
          command: ["alembic", "upgrade", "head"]
      restartPolicy: Never
  backoffLimit: 2
```

A release pipeline (Argo CD, a Helm hook, or a plain CI/CD step) applies this `Job` and waits for it to report `Complete` before rolling the `Deployment` forward — the same gating principle as Chapter 15's CI job, just applied to the production release instead of the test pipeline. Avoid using a readiness probe on the application pods themselves as a substitute for this gate: a probe that merely checks "can I connect to Postgres" says nothing about whether the schema the running code expects actually exists yet.

---

## 5. Reverse-Engineering Models with `sqlacodegen`

Every prior chapter assumed ExpenseFlow's SQLAlchemy models come first and migrations follow. The reverse situation happens too — most commonly when adopting Alembic against a database that already exists, with no SQLAlchemy models yet (a legacy database, or a database another team owns and you're integrating with read-only). Writing `Mapped[...]`/`mapped_column(...)` declarations by hand for a schema with forty existing tables is tedious and error-prone in exactly the places that matter (getting a `nullable` or a foreign key wrong).

`sqlacodegen` connects to a live database and generates SQLAlchemy model code from its actual schema:

```bash
pip install sqlacodegen

sqlacodegen postgresql://expenseflow:pw@localhost/expenseflow_legacy \
  --outfile app/models/legacy_generated.py
```

The output is a reasonable starting point, not a finished, idiomatic codebase — review it the same way you'd review an autogenerated migration (Chapter 16): confirm relationships came out correctly, check that column types map to what you actually want in SQLAlchemy 2.0 style (`sqlacodegen`'s output style has historically lagged behind the newest `Mapped[...]` declarative syntax, so expect to hand-adjust it), and never treat generated model code as exempt from the same code review your hand-written models get.

Once models exist (generated or hand-written) for an already-existing database, the next step is Chapter 9's `alembic stamp head` — telling Alembic this database is already at the current schema state, so the first real migration you write going forward is a genuine, incremental change rather than an attempt to recreate the whole legacy schema from scratch.

---

## 6. Alembic in Context: How Other Ecosystems Solve the Same Problem

Alembic is SQLAlchemy's migration tool, but "versioned, incremental schema change with an upgrade/downgrade pair" is a problem every serious web framework and database tooling ecosystem has solved, usually in a very similar shape. Recognizing the parallels (and the differences) is useful both for onboarding engineers who come from another stack, and for system-design conversations where the underlying database, not the framework, is the actual point of comparison.

| Tool | Ecosystem | Migration authoring | Autogenerate | Down/rollback | Notes |
|---|---|---|---|---|---|
| **Alembic** | Python / SQLAlchemy | Python scripts (`upgrade()`/`downgrade()`) | Yes, DB-vs-metadata diff (Chapter 7) | Yes, per-revision `downgrade()` | Framework-agnostic; used by FastAPI, Flask, and any SQLAlchemy project |
| **Django Migrations** | Django (Python) | Auto-generated Python migration files from model changes, editable | Yes, and more automatic than Alembic's — tightly coupled to Django's own ORM | Yes, Django can compute reverse operations for most auto-generated migrations | Tightly integrated with Django's ORM specifically; less flexible outside Django |
| **Flyway** | Java/JVM, polyglot (works with any language via CLI) | Plain versioned `.sql` files (`V1__init.sql`, `V2__add_categories.sql`) | No — SQL is hand-written, not diffed from an ORM model | Historically forward-only by convention (undo scripts are a paid/limited feature); teams write compensating forward migrations instead | No ORM coupling at all; version history tracked in a `flyway_schema_history` table, conceptually identical to `alembic_version` |
| **Liquibase** | Java/JVM, polyglot | Changesets in XML, YAML, JSON, or SQL, tracked by ID+author+checksum | No — changesets are hand-authored, though Liquibase can generate a changelog from an existing DB (`generateChangeLog`, comparable to `sqlacodegen`'s role) | Yes, `rollback` can be defined per changeset or auto-computed for some operation types | Checksums each changeset and refuses to silently re-run an edited one — a structural defense against Chapter 17's Mistake #3 that Alembic doesn't have built in |

The conceptual core is identical everywhere: a linear (or branching-and-mergeable) history of small, ordered, ideally-reversible schema changes, tracked by a small state table in the database itself (`alembic_version`, `django_migrations`, `flyway_schema_history`, `DATABASECHANGELOG`). What differs is how much of the DDL is hand-written versus diffed from application code, and how strictly each tool defends against a changed-after-the-fact migration file — worth knowing if you ever need to explain Alembic's model to someone who's only ever used one of the others, or vice versa.

---

## 7. IDE and Editor Tips

A handful of small editor habits pay off disproportionately once a migrations folder has fifty-plus files in it:

- **Enable Python syntax highlighting and linting for `script.py.mako`-generated files specifically** — most editors correctly treat generated `.py` files under `alembic/versions/` as ordinary Python once generated, but the `.mako` template itself (Chapter 4) benefits from Mako-aware syntax highlighting if your editor supports it, since it's a mixed Python/template-directive file.
- **Don't expect "go to definition" to follow a `down_revision` chain** — revision IDs are just string literals, not import references, so no editor's navigation will jump from one revision file to its parent automatically. Lean on `alembic history` and `alembic show <rev>` (Chapter 5) from an integrated terminal instead.
- **Use a project-level snippet or template for new migration files** that already includes your team's docstring conventions (ticket reference, author, a one-line "why" comment) so every migration is self-documenting from the moment `alembic revision` creates it.
- **Keep `alembic/versions/` sorted by filename with the revision ID visible**, and configure `file_template` (Chapter 4) to prefix filenames with a date or sequence number (`%(rev)s_%(slug)s` vs. `%(year)d%(month).2d%(day).2d_%(rev)s_%(slug)s`) so a directory listing roughly approximates chronological order — pure revision-ID filenames sort effectively at random, which makes skimming the folder unnecessarily hard.
- **Add a pre-commit hook or CI lint step that runs `alembic heads` and fails on more than one result**, turning Chapter 17's Mistake #2 into something caught locally before a PR is even opened, not just in CI after the fact.

---

## Real-World Scenario

**Setup:** Six months after ExpenseFlow's zero-downtime rename (Chapter 14) and CI pipeline (Chapter 15) are both running smoothly, the team takes on two new pieces of work: a finance-team request for a pre-aggregated monthly spending view for their BI tool, and an acquisition that brings in a second, older expense-tracking database with no SQLAlchemy models at all, which needs to be merged into ExpenseFlow's stack.

**The view request.** Rather than hand-writing `op.execute("CREATE VIEW ...")` and losing all future diffing ability, the team adopts `alembic-utils`, defines `monthly_expense_summary` as a `PGView` in `app/db/views.py`, and registers it in `env.py` alongside `target_metadata`. When the finance team later asks for the view to also break down spending by category, the fix is a one-line change to the view's `definition` string and a routine `--autogenerate` run — the same reviewed-diff workflow as any table change, instead of a hand-written `DROP VIEW`/`CREATE VIEW` pair.

**The acquisition database.** The acquired database has forty-odd tables and no ORM layer at all. The team runs `sqlacodegen` against a read replica to get a first-draft set of SQLAlchemy models, reviews the output carefully (two foreign keys came out with the wrong `ondelete` behavior and are corrected by hand), and once the models are confirmed accurate, runs `alembic stamp head` against the acquired database — recording that it's already at the current schema state without attempting to replay its entire history as a fresh set of migrations. The first genuine migration written afterward is a small, real change: adding ExpenseFlow's `category_id` foreign key convention to the acquired schema, folding it properly into the unified data model.

**Testing both.** The team adds a `pytest-alembic` suite to CI (extending Chapter 15's existing pipeline) that verifies a single head, a clean upgrade/downgrade round trip, and a custom test confirming the new view actually returns the right aggregation shape against a small fixture dataset — catching, during a later refactor, that a column rename in `expenses` had silently broken the view's `GROUP BY` clause before it ever reached production.

**Outcome:** Two different ecosystem tools solved two different gaps in core Alembic — `alembic-utils` for a database object with no ORM equivalent, `sqlacodegen` plus `alembic stamp` for adopting Alembic against an existing, model-less database — and `pytest-alembic` gave both a testing safety net that a simple `alembic upgrade head` CI step wouldn't have provided on its own.

---

## Best Practices

- **Reach for `alembic-utils` the moment you write your second hand-rolled `op.execute("CREATE VIEW ...")`** — the first one might be a one-off, but a database object you expect to change deserves the same diffing support a table gets.
- **Pin community package versions (`alembic-utils`, enum-diffing packages, `pytest-alembic`) as deliberately as Alembic and SQLAlchemy themselves** — this corner of the ecosystem moves independently and can change autogenerate's behavior underneath you.
- **Run migrations as a dedicated Docker entrypoint step or CI/CD job, never inside a FastAPI `startup` event handler** — the startup-event pattern races the moment you scale past one replica.
- **Treat `sqlacodegen` output as a reviewable first draft**, not a finished set of models — check relationships, nullability, and type mappings by hand before trusting it.
- **Add `pytest-alembic`'s built-in single-head and round-trip checks to CI early**, even before writing custom migration tests — they catch Chapter 17's most common structural mistakes for free.
- **Know Alembic's place in the wider landscape** (Section 6's comparison table) well enough to translate the concepts here to a Django, Flyway, or Liquibase context if a project ever calls for it.

---

## Common Mistakes

- **Hand-writing `op.execute()` for a Postgres view or function that changes regularly**, instead of adopting `alembic-utils`, and losing all autogenerate diffing ability for that object as a result.
- **Depending on an unmaintained or unpinned community package** (an enum-diffing helper, a niche `alembic-utils`-adjacent tool) in a production migration path without checking its maintenance status or pinning its version.
- **Running migrations inside a FastAPI startup event handler in a multi-replica deployment**, silently reintroducing the concurrent-migration race Chapter 15 and Chapter 17 (Mistake #5) both warn against.
- **Treating `sqlacodegen`'s output as production-ready without review**, shipping incorrect `ondelete` behavior or nullable flags straight from generated code.
- **Assuming `pytest-alembic`'s built-in tests replace application-level integration testing** — it verifies migration mechanics (single head, clean round trip), not that your application logic behaves correctly against the resulting schema.
- **Confusing Flyway/Liquibase's forward-only-by-convention culture with Alembic's downgrade-per-revision model** when discussing migration strategy across teams that use different tools, and assuming practices transfer 1:1 when the underlying guarantees actually differ.

---

## Summary

- **`alembic-utils`** (Section 1) gives Postgres views, functions, and triggers the same versioned, diffable treatment core Alembic gives tables.
- **Enum-diffing community packages** (Section 2) close autogenerate's gap around Postgres `ENUM` value changes and their transaction-block caveat.
- **`pytest-alembic`** (Section 3) turns migration correctness properties (single head, clean upgrade/downgrade, custom data assertions) into individually reported pytest tests.
- **FastAPI/Docker integration** (Section 4) means running migrations as a single dedicated entrypoint step or release job — never inside a per-replica application startup hook.
- **`sqlacodegen`** (Section 5) reverse-engineers SQLAlchemy models from an existing database, useful when adopting Alembic against a schema that predates any ORM layer, paired with `alembic stamp head` (Chapter 9).
- **Alembic in context** (Section 6): the same versioned-migration-with-a-history-table shape recurs across Django migrations, Flyway, and Liquibase, differing mainly in how much DDL is hand-written vs. diffed, and how strictly each tool defends against an edited-after-the-fact migration.
- **IDE/editor habits** (Section 7): Mako-aware highlighting, a documented migration template, sane `file_template` ordering, and a local `alembic heads` check all pay off as the migrations folder grows.
- The **Real-World Scenario** showed `alembic-utils`, `sqlacodegen`, and `pytest-alembic` each solving a distinct gap in the same ExpenseFlow project within one work cycle.

---

## Knowledge Check

1. What specifically does `alembic-utils` add on top of core Alembic's autogenerate, and why can't a plain `op.execute("CREATE VIEW ...")` get the same diffing behavior?
2. Why does adding a value to a Postgres `ENUM` type have a transaction-block caveat, and what does an enum-diffing community package handle on your behalf?
3. Name two properties `pytest-alembic`'s built-in test suite checks that a plain `alembic upgrade head` CI shell step does not verify individually.
4. Explain what's wrong with running Alembic migrations inside a FastAPI `@app.on_event("startup")` handler once an application is deployed with more than one replica.
5. You've been handed a legacy database with no SQLAlchemy models. What two tools/commands, used together, get you to a working, versioned Alembic setup against it without replaying its entire history as new migrations?
6. How does Liquibase's changeset checksum mechanism address a failure mode that core Alembic has no built-in defense against? Which chapter covered that failure mode?
7. A colleague who's only used Flyway joins the team and asks why Alembic migrations have a `downgrade()` function at all, since their previous team treated migrations as forward-only. How would you explain the difference in philosophy?

---

## Hands-On Exercise

1. Install `pytest-alembic` and `alembic-utils` into ExpenseFlow's virtual environment:

```bash
pip install pytest-alembic alembic-utils
```

2. Define a small Postgres view using `alembic-utils`, modeled on ExpenseFlow's schema:

```python
# app/db/views.py
from alembic_utils.pg_view import PGView

recent_expenses_view = PGView(
    schema="public",
    signature="recent_expenses",
    definition="""
        SELECT id, user_id, amount_cents, expense_date
        FROM expenses
        WHERE expense_date >= current_date - interval '30 days'
    """,
)
```

3. Register it in `alembic/env.py` alongside `target_metadata`, then generate and review the migration:

```bash
alembic revision --autogenerate -m "add recent_expenses view"
```

Read the generated file and confirm it contains the correct `CREATE VIEW` statement via `alembic_utils`'s operation classes, not a raw `op.execute()`.

4. Apply it, then add a `conftest.py` wiring up `pytest-alembic`'s fixtures against a local test database, and run the built-in suite:

```bash
alembic upgrade head
pytest --test-alembic
```

5. Change the view's `definition` to add a `category_id` column, re-run `--autogenerate`, and confirm a new migration is generated containing a `CREATE OR REPLACE VIEW` reflecting the change.

6. Write one custom `pytest-alembic` test that asserts the view returns zero rows against a freshly migrated, empty database — confirming the view's mechanics work independent of any seeded data.

---

## Further Reading

- [`alembic-utils` GitHub Repository](https://github.com/olirice/alembic_utils) — full reference for `PGView`, `PGFunction`, `PGTrigger`, and registering entities in `env.py`.
- [`pytest-alembic` documentation](https://pytest-alembic.readthedocs.io/) — the built-in test suite, custom test authoring, and fixture configuration referenced in Section 3.
- [Alembic Official Documentation](https://alembic.sqlalchemy.org/en/latest/) — the umbrella reference this whole ecosystem extends.
- [Alembic GitHub Repository](https://github.com/sqlalchemy/alembic) — issues and discussions are a good place to gauge the maintenance status of Alembic itself and any dependent tooling.
- [FastAPI SQL Databases guide](https://fastapi.tiangolo.com/tutorial/sql-databases/) — the application-lifecycle context behind Section 4's entrypoint/startup discussion.

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./17-common-mistakes-and-pitfalls.md">← Previous: Common Mistakes & Pitfalls</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./19-capstone-projects.md">Next: Capstone Projects →</a>
</div>
