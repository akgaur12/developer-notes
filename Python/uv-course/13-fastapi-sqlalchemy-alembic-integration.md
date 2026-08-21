# Integrating FastAPI, SQLAlchemy & Alembic

[Chapter 12: Workspaces & Monorepos](./12-workspaces-and-monorepos.md) left ExpenseFlow restructured as a two-member uv workspace — `packages/api`, the FastAPI application, and `packages/shared`, a small internal library of Pydantic schemas and constants used by both the API and a forthcoming background-worker service — sharing one `uv.lock` and one resolution. Every chapter before this one taught a piece of uv in isolation: adding dependencies, locking them, running code, managing dev tools, structuring a workspace. This chapter doesn't introduce a new uv feature at all. Instead, it assembles everything so far into the loop Priya and Marcus actually run, dozens of times a day, while building ExpenseFlow — starting the API, generating and applying migrations, and running the test suite — and it draws a firm line around what uv is actually responsible for in that loop versus what belongs to FastAPI, SQLAlchemy, Alembic, and `pydantic-settings` instead. Getting that boundary right matters more than it sounds: a surprising number of real "uv" confusions turn out to be config-loading confusions wearing a uv costume.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Run ExpenseFlow's FastAPI dev server, Alembic commands, and test suite entirely through `uv run`, in a workspace where the runnable code lives in `packages/api`.
- Explain precisely which parts of ExpenseFlow's configuration uv manages (the Python interpreter, the dependency set, the environment) and which parts it deliberately does not (runtime configuration values like `DATABASE_URL`, which `pydantic-settings` reads from `.env`).
- Wire `pydantic-settings` to load `DATABASE_URL` and other runtime configuration from a `.env` file, and explain why `uv run` does not read or interpret that file itself.
- Read and reproduce a complete, workspace-aware `pyproject.toml` for ExpenseFlow's `packages/api` member, reflecting every dependency added since Chapter 7.
- Run `uv run pytest` against a real, migrated test database rather than one built by `Base.metadata.create_all()`, and explain why that distinction matters.
- Assemble a one-command developer loop (lint, type-check, test) appropriate for day-to-day ExpenseFlow work, building on Chapter 10's dev-dependency setup.

---

## Prerequisites for This Chapter

- The mechanics of `uv run` — how it ensures the environment matches the lockfile before executing anything — from [Chapter 9: Running Code with `uv run`](./09-running-code-with-uv-run.md).
- Dev dependencies (`pytest`, `ruff`, `mypy`) added via `uv add --dev`, from [Chapter 10: Development Dependencies & Tooling](./10-development-dependencies-and-tooling.md).
- ExpenseFlow's `packages/api` + `packages/shared` workspace layout and how `uv sync` resolves the whole workspace at once, from [Chapter 12: Workspaces & Monorepos](./12-workspaces-and-monorepos.md).
- Basic familiarity with FastAPI, SQLAlchemy 2.0's `Mapped[...]`/`mapped_column(...)` style, and Alembic's `upgrade`/`downgrade`/`revision --autogenerate` commands is assumed but not re-taught here — for the schema and migration side of ExpenseFlow in full depth, see this repo's sibling [Alembic & Database Schema Migrations course](../../Databases/alembic-course/00-index.md), which this chapter references throughout rather than repeats.

---

## 1. The ExpenseFlow Developer Loop, End to End

Every uv-specific skill from Chapters 1–12 exists to make one thing effortless: a developer sitting down to work on ExpenseFlow should be able to run a handful of `uv run` commands and trust that whatever runs is using exactly the right interpreter and exactly the right, locked dependency set — no manual `source .venv/bin/activate`, no "did I `pip install` the latest requirements" uncertainty, no drift between what Priya's laptop has installed and what Marcus's has. The day-to-day loop looks like this:

```mermaid
flowchart TD
    A["uv run uvicorn app.main:app --reload\n(start the dev server)"] --> B["Edit a SQLAlchemy model\nin packages/api/src/app/models.py"]
    B --> C["uv run alembic revision --autogenerate -m '...'\n(generate a migration)"]
    C --> D["Review the generated migration file by hand"]
    D --> E["uv run alembic upgrade head\n(apply it to the dev database)"]
    E --> F["uv run pytest\n(run the test suite against a migrated test database)"]
    F --> G["uv run ruff check && uv run mypy\n(lint + type-check before committing)"]
    G -->|loop back| B
```

Nothing in this diagram is uv inventing new behavior — `uvicorn`, `alembic`, and `pytest` are ordinary third-party tools doing exactly what they'd do if invoked directly. What `uv run` contributes, in front of every single one of these commands, is the guarantee from Chapter 9: before executing the command, uv checks the project's environment against `uv.lock` and brings it into sync if anything has drifted, then executes the command with that environment's interpreter — automatically, every time, without a separate "did you remember to sync" step a developer has to remember on their own.

---

## 2. Running the FastAPI Dev Server Through `uv run`

### 2.1 The command, and why it's `app.main:app`

ExpenseFlow's `packages/api` member deliberately names its importable package `app` — not `expenseflow_api` — specifically so the day-to-day command every teammate types is the short, immediately recognizable one FastAPI's own tutorials use:

```bash
cd packages/api
uv run uvicorn app.main:app --reload
```

The *distribution* name (what shows up in `uv.lock`, what `packages/shared` depends on) is still `expenseflow-api` — Python has always allowed a package's installable/distribution name to differ from the importable module name inside it, and ExpenseFlow's team uses that gap deliberately here, trading a little bit of "this is unusual" for a command every new hire recognizes on sight from any FastAPI getting-started guide.

`--reload` tells uvicorn to watch the source tree and restart the server on every file save — this is a uvicorn feature, not a uv one, and it works identically whether uv or a manually activated virtualenv is running the process underneath.

### 2.2 Running from the workspace root vs. from `packages/api`

Because ExpenseFlow is now a uv workspace (Chapter 12), there are two equally valid places to issue this command from:

```bash
# From inside packages/api directly — uv discovers the nearest pyproject.toml,
# recognizes it as a workspace member, and uses the shared workspace environment.
cd packages/api
uv run uvicorn app.main:app --reload

# From the workspace root — the --package flag tells uv which member's
# context to use for this invocation, without needing to cd anywhere.
uv run --package expenseflow-api uvicorn app.main:app --reload
```

Both commands execute against the exact same `.venv` — a uv workspace shares one environment and one lockfile across all its members, so there's no risk of the two invocations quietly using different dependency versions. Priya prefers `cd packages/api` for anything she's actively working on; Marcus, who more often runs one-off commands against a member he isn't currently `cd`'d into (chasing down a CI failure, for instance), uses `--package` from the root. Both are correct; pick whichever matches how your terminal session is already organized.

### 2.3 What `uv run` is doing underneath, one more time

It's worth restating Chapter 9's point in this specific context, because it's easy to let the FastAPI-specific command obscure it: `uv run uvicorn app.main:app --reload` is not "uv starts uvicorn." It is "uv resolves whether the workspace environment currently matches `uv.lock`, syncs it if not, then hands off execution to `uvicorn`, which is just an ordinary console-script entry point installed into that environment." If a teammate just pulled a branch that added a new dependency to `packages/api/pyproject.toml` and regenerated `uv.lock`, the very next `uv run uvicorn ...` transparently installs that new dependency before starting the server — no separate `uv sync` step required, and no stale-environment surprise either.

---

## 3. The Alembic Loop Through `uv run`

### 3.1 Generating a migration

When Priya adds a new column to a SQLAlchemy model in `packages/api/src/app/models.py`, the next step is exactly what the sibling Alembic course teaches in depth — autogenerate a migration by diffing the models against the live database:

```bash
cd packages/api
uv run alembic revision --autogenerate -m "add rollover_enabled to monthly_budgets"
```

`uv run` here does the same job it always does: confirm the environment (which includes `alembic` itself, `sqlalchemy`, and the async database driver `asyncpg`) matches `uv.lock`, then execute `alembic`'s own `revision --autogenerate` command. Everything about *how* autogenerate compares models to the database, what it can and can't detect automatically, and how to review the generated file by hand before trusting it is covered thoroughly in [the Alembic course's Chapter 7, Autogenerate Migrations](../../Databases/alembic-course/07-autogenerate-migrations.md) — this chapter is only concerned with the fact that the command runs through `uv run` rather than a bare, ambient `alembic`.

### 3.2 Applying migrations

```bash
uv run alembic upgrade head
```

Same story: uv ensures the right environment, then Alembic does what Alembic does — applies every not-yet-applied revision, in order, up to `head`. In development, Priya and Marcus run this directly against a local PostgreSQL instance (usually one running in Docker Compose, a detail [Chapter 14](./14-docker-integration.md) picks up); in CI, the same command runs against an ephemeral database, a topic [Chapter 15](./15-cicd-integration.md) covers as a dedicated pipeline job.

### 3.3 Why this is a two-tool handshake, not a uv feature

It's worth being explicit that uv has no special "Alembic integration" — there is no `uv migrate` command, and there doesn't need to be. Alembic already has a complete, mature CLI; uv's entire job is making sure that CLI, whenever invoked via `uv run`, is running against the exact interpreter and dependency set the project's `uv.lock` describes. This is a deliberate design choice on Astral's part, consistent with everything Chapter 2 said about uv building on standards rather than reinventing tool-specific behavior — uv doesn't need to understand what Alembic does in order to correctly run it.

---

## 4. The Configuration Boundary: What uv Manages, What It Doesn't

### 4.1 The scope line, drawn precisely

This is the single most important conceptual point in this chapter, and it resolves a confusion that trips up nearly every team adopting uv for the first time: **uv manages the Python interpreter, the dependency set, and the virtual environment that command runs in. It has no concept of, and no involvement in, your application's runtime configuration** — database URLs, API keys, feature flags, environment names (`development`/`staging`/`production`). Those are the application's concern, not the package manager's, and ExpenseFlow delegates them entirely to `pydantic-settings`.

```mermaid
flowchart LR
    subgraph uv["uv's scope"]
        A["Python interpreter\n(3.13, pinned via .python-version)"]
        B["Dependency set\n(pyproject.toml + uv.lock)"]
        C["Virtual environment\n(.venv, kept in sync by uv run)"]
    end
    subgraph app["Application's scope"]
        D["DATABASE_URL, SECRET_KEY,\nfeature flags, etc."]
        E[".env file"]
        F["pydantic-settings\nSettings class"]
    end
    E --> F --> D
    uv -.->|"uv run starts the process,\nthen gets out of the way"| app
```

`uv run` starts the process with the correct interpreter and dependencies already in place, and then it is simply gone — the process it launched is an ordinary Python process, and everything that process does to figure out its own runtime configuration happens entirely outside of anything uv is aware of.

### 4.2 `pydantic-settings` and `.env`

ExpenseFlow's `packages/api/src/app/config.py` defines its settings model using `pydantic-settings` (added as a dependency back in Chapter 7):

```python
# packages/api/src/app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    database_url: str
    environment: str = "development"
    feature_read_notes_column: bool = False


settings = Settings()
```

And a `.env` file, sitting in `packages/api/` alongside `pyproject.toml`, that is **never** committed to version control (only `.env.example`, a template with placeholder values, is):

```bash
# packages/api/.env  (git-ignored — see .gitignore)
DATABASE_URL=postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow_dev
ENVIRONMENT=development
FEATURE_READ_NOTES_COLUMN=false
```

When `uv run uvicorn app.main:app --reload` starts the application, `Settings()` is instantiated somewhere during import (typically as a module-level singleton, `settings`, imported wherever configuration is needed), and *at that point* — entirely inside the running Python process, with no involvement from uv whatsoever — `pydantic-settings` reads `.env`, applies any real OS environment variables that override it, validates the values against the declared field types, and produces the `Settings` instance the rest of the application uses. If `DATABASE_URL` is missing from both `.env` and the real environment, `pydantic-settings` raises a validation error immediately — a clear, fast failure that has nothing to do with uv and everything to do with the settings model itself.

### 4.3 Why this distinction matters in practice

Two very different categories of "it doesn't work" bug get regularly conflated by teams new to uv, and it's worth naming both explicitly so you never lose time down the wrong one:

| Symptom | Actual cause | Fix |
|---|---|---|
| `uv run uvicorn app.main:app` fails with `ModuleNotFoundError: No module named 'fastapi'` | uv's scope — the dependency isn't in `uv.lock`/the environment | `uv add fastapi`, or `uv sync` if it's already declared but not installed |
| `uv run uvicorn app.main:app` starts, but the app immediately raises a `pydantic` `ValidationError` about `database_url` | `pydantic-settings`'s scope — no `.env` file present, or `DATABASE_URL` missing from it | Create/fix `packages/api/.env`; this has nothing to do with `uv sync` or the lockfile |
| The app connects to the wrong database in CI | `pydantic-settings`'s scope — CI sets a real `DATABASE_URL` environment variable that overrides whatever a stray `.env` might contain | Confirm CI's environment variables, not uv's cache or lockfile |
| Two teammates get different dependency-resolution errors on the same code | uv's scope — an un-pinned resolution difference (Chapter 8) | `uv sync --locked` to catch it, `uv lock` to re-resolve deliberately |

Marcus lost the better part of an afternoon early in ExpenseFlow's life chasing what he assumed was a uv lockfile problem, running `uv lock --upgrade` and `uv sync` repeatedly, before Priya pointed out the actual error message: a `pydantic.ValidationError` complaining `database_url: Field required` — his local `.env` file had been accidentally deleted by an overzealous `git clean -fdx`. uv had done its job perfectly the entire time; the problem was one directory layer up, in a file uv has never heard of.

---

## 5. The Test Loop: `uv run pytest` Against a Real Test Database

### 5.1 Why the test database matters

Chapter 10 established `pytest` as a dev dependency, run via `uv run pytest`. For ExpenseFlow specifically, the test suite exercises real SQLAlchemy queries against a real PostgreSQL database — not an in-memory SQLite fixture — because (as the sibling Alembic course's Chapter 13 covers in depth) SQLite's behavior diverges from PostgreSQL's in ways that matter for this project (constraint enforcement, `JSONB` columns, concurrent-migration semantics). ExpenseFlow's `conftest.py`, at `packages/api/tests/conftest.py`, provisions a dedicated test database and runs the real Alembic migration chain against it before any test executes:

```python
# packages/api/tests/conftest.py
import asyncio

import pytest
from alembic import command
from alembic.config import Config
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from app.config import settings


@pytest.fixture(scope="session", autouse=True)
def apply_migrations() -> None:
    """Run the real Alembic migration chain against the test database once per session."""
    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", settings.database_url)
    command.upgrade(alembic_cfg, "head")


@pytest.fixture
async def db_session():
    engine = create_async_engine(settings.database_url)
    session_factory = async_sessionmaker(engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
        await session.rollback()
    await engine.dispose()
```

### 5.2 Pointing tests at a *different* database than dev

The test suite must never run against the same database Priya or Marcus is using for manual dev-server testing — a test that leaves data behind (or, worse, a test run mid-development that truncates a table a developer was relying on) is exactly the kind of cross-contamination a dedicated test database avoids entirely. ExpenseFlow handles this the same way it handles every other runtime value — through `pydantic-settings`, driven by which `.env`-equivalent is active for the process:

```bash
# Running the dev server: reads packages/api/.env (DATABASE_URL points at expenseflow_dev)
uv run uvicorn app.main:app --reload

# Running tests: a separate environment variable, set only for the test invocation,
# overrides DATABASE_URL to point at a dedicated expenseflow_test database
DATABASE_URL=postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow_test \
  uv run pytest -v
```

Because `pydantic-settings` gives real OS environment variables precedence over `.env` file contents by default, this override works without editing any file — exactly the kind of runtime-configuration flexibility that lives entirely outside uv's concerns, and exactly why [Chapter 15](./15-cicd-integration.md)'s CI pipeline can point the same test suite at a freshly created, ephemeral database with nothing more than an environment variable set at the job level.

### 5.3 Running the suite

```bash
cd packages/api
uv run pytest -v --cov=app
```

As always, `uv run` first confirms the environment (which includes `pytest`, `pytest-asyncio`, and `pytest-cov` — all dev dependencies, never shipped to production, a point [Chapter 14](./14-docker-integration.md) returns to directly) matches `uv.lock`, then hands off to `pytest` itself.

---

## 6. The Full ExpenseFlow `pyproject.toml`, Assembled

### 6.1 The workspace root

Since Chapter 12, ExpenseFlow's root `pyproject.toml` is a **virtual workspace root** — it declares the workspace members but is not itself an installable project:

```toml
# expenseflow/pyproject.toml (workspace root)
[tool.uv.workspace]
members = ["packages/*"]

[tool.uv]
package = false
```

### 6.2 `packages/api/pyproject.toml`, in full

This is every dependency ExpenseFlow's API member has accumulated since Chapter 7 — runtime dependencies added via `uv add`, the dev group added via `uv add --dev` in Chapter 10, and the `expenseflow-shared` workspace dependency added in Chapter 12 — shown exactly as it stands entering this chapter:

```toml
# packages/api/pyproject.toml
[project]
name = "expenseflow-api"
version = "0.1.0"
description = "ExpenseFlow's FastAPI application"
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.115.6,<0.116",
    "uvicorn[standard]>=0.32.1,<0.33",
    "sqlalchemy>=2.0.36,<2.1",
    "alembic>=1.14.0,<2",
    "asyncpg>=0.30.0,<0.31",
    "pydantic-settings>=2.6.1,<3",
    "expenseflow-shared",
]

[dependency-groups]
dev = [
    "pytest>=8.3.4,<9",
    "pytest-asyncio>=0.24.0,<0.25",
    "pytest-cov>=6.0.0,<7",
    "ruff>=0.8.2,<0.9",
    "mypy>=1.13.0,<2",
    "pre-commit>=4.0.1,<5",
]

[tool.uv.sources]
expenseflow-shared = { workspace = true }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/app"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py313"

[tool.mypy]
python_version = "3.13"
strict = true
```

Reading this end to end tells the whole dependency-management story of Chapters 7–12 in one file: `[project.dependencies]` is exactly the runtime set ExpenseFlow needed to become a working FastAPI + SQLAlchemy + Alembic + PostgreSQL application; `httpx`, briefly added and then removed back in Chapter 7's worked example, is conspicuously absent, exactly as that chapter left it; `[dependency-groups.dev]` is the tooling from Chapter 10, kept strictly separate from the runtime set for the reason [Chapter 14](./14-docker-integration.md) makes concrete (dev dependencies never enter the production image); and `[tool.uv.sources]` with `expenseflow-shared = { workspace = true }` is Chapter 12's path-dependency mechanism, telling uv to resolve `expenseflow-shared` from the sibling `packages/shared` workspace member rather than from PyPI.

---

## 7. The One-Command Dev Loop

Chapter 10 previewed the idea that a `Makefile` or `justfile` wrapping the day-to-day commands is a convenience, not a requirement — every command it wraps is still just `uv run` underneath. ExpenseFlow's team settled on a small `Makefile` at `packages/api/Makefile`:

```makefile
.PHONY: dev migrate test check

dev:
	uv run uvicorn app.main:app --reload

migrate:
	uv run alembic upgrade head

test:
	uv run pytest -v --cov=app

check:
	uv run ruff check .
	uv run mypy .
	uv run pytest
```

`make check` before every commit is the closest thing ExpenseFlow's team has to a formal pre-push ritual — and, as Chapter 10 also covered, the exact same three commands are wired into `pre-commit`, so the check runs automatically on `git commit` even if someone forgets to run `make check` by hand. Nothing here is uv-specific ceremony; it's uv, quietly doing its one job — guaranteeing the environment underneath each of these commands is correct — well enough that the Makefile only needs to think about *what* to run, never *how* to make sure it runs against the right dependencies.

---

## Real-World Scenario

A new engineer, Diego, joins ExpenseFlow's team and clones the repository for the first time. He runs `uv sync` at the workspace root — which, per Chapter 12, resolves and installs both `packages/api` and `packages/shared` into one shared `.venv` — and then, following the README, runs `cd packages/api && uv run uvicorn app.main:app --reload`. The server starts, but the very first request to `/expenses` returns a 500 error. Diego's first instinct, having just spent a week on the earlier chapters of this course, is to suspect a dependency problem — he re-runs `uv sync --locked` to confirm the lockfile is being honored exactly, and it passes cleanly with no changes.

The actual traceback, which he'd glossed over the first time, is a `pydantic_core.ValidationError: 1 validation error for Settings — database_url: Field required`. There's no `.env` file in his fresh clone at all, because — correctly — it's git-ignored, and the README's setup instructions, written before Chapter 4's addition, never mentioned copying `.env.example` to `.env`. Once he does that and fills in a real `DATABASE_URL` pointing at his local PostgreSQL instance, the server starts cleanly. Priya, reviewing his onboarding notes afterward, adds one line to the README — "copy `.env.example` to `.env` before running `uv run uvicorn ...`" — and a short comment to this effect at the top of `config.py` itself, precisely because this chapter's scope boundary (Section 4) is easy to internalize in the abstract and still trip over the first time it's a blank `.env` file rather than a concept in a course.

---

## Best Practices

- Run every ExpenseFlow command — the dev server, Alembic, the test suite — through `uv run`, never through an ambient, globally-installed `uvicorn`/`alembic`/`pytest` that might silently be a different version than what `uv.lock` pins.
- Keep `.env` out of version control entirely; commit only `.env.example` with placeholder values, and document the copy-and-fill-in step explicitly in the project README.
- Point the test suite at a dedicated test database via an environment variable override, never at the same database a developer is using for manual testing.
- Run the real Alembic migration chain against the test database in a session-scoped fixture, rather than building test tables with `Base.metadata.create_all()`, so tests exercise the same schema-creation path production actually uses.
- Keep dev dependencies (`pytest`, `ruff`, `mypy`, `pre-commit`) in `[dependency-groups.dev]`, never in `[project.dependencies]` — this distinction is what makes Chapter 14's production image lean and dev-tool-free.
- When a `uv run <command>` fails, check first whether the failure is a missing/misconfigured dependency (uv's scope) or a missing/misconfigured runtime setting (`pydantic-settings`'s scope) before assuming either one — the error message almost always tells you which.

---

## Common Mistakes

- Assuming a `pydantic.ValidationError` about a missing setting is a uv/dependency problem, and reaching for `uv sync`/`uv lock --upgrade` when the actual fix is creating or correcting a `.env` file.
- Running the test suite against the same database used for manual dev-server testing, causing tests to leave behind or destroy data a developer was relying on.
- Building test fixtures with `Base.metadata.create_all()` instead of running the real Alembic chain, hiding model/migration drift that would only surface later, against a database that went through real migrations (the exact bug [Chapter 15](./15-cicd-integration.md)'s drift check exists to catch in CI).
- Forgetting that `uv run` from inside `packages/api` and `uv run --package expenseflow-api` from the workspace root use the *same* shared environment — treating them as somehow independently syncable and being confused when a change made from one location doesn't seem to "apply" from the other (it always does; there's only one `.venv`).
- Committing a real `.env` file with a developer's local database credentials to version control, either directly or by forgetting to add it to `.gitignore` on a fresh clone.
- Mixing runtime configuration into `pyproject.toml` (for example, hardcoding a `DATABASE_URL` default there) instead of `pydantic-settings`/`.env` — `pyproject.toml` is uv's territory (dependencies, build metadata), not the application's.

---

## Summary

- ExpenseFlow's day-to-day developer loop — dev server, Alembic migrations, test suite — runs entirely through `uv run`, which guarantees the environment matches `uv.lock` before handing off to the underlying tool each time (Section 1).
- The FastAPI dev server runs as `uv run uvicorn app.main:app --reload`, either from inside `packages/api` or from the workspace root with `--package expenseflow-api`, both against the same shared workspace environment (Section 2).
- Alembic's `revision --autogenerate` and `upgrade head` run through `uv run` exactly like any other command — uv has no Alembic-specific behavior, and the migration mechanics themselves are the sibling Alembic course's concern, not this one's (Section 3).
- uv's scope is the interpreter, the dependency set, and the environment; runtime configuration like `DATABASE_URL` is `pydantic-settings`'s job, reading from a `.env` file uv never touches — most "uv isn't working" confusions in practice turn out to be misconfigured `.env` files instead (Section 4).
- The test suite runs via `uv run pytest` against a dedicated test database, migrated with the real Alembic chain rather than `Base.metadata.create_all()` (Section 5).
- `packages/api/pyproject.toml`, shown in full, reflects every dependency added since Chapter 7 — runtime dependencies, the dev group, and the `expenseflow-shared` workspace source (Section 6).
- A `Makefile` (or equivalent) wrapping `uv run` commands is a convenience layer, not a new capability — every line in it is still just `uv run` underneath (Section 7).

---

## Knowledge Check

1. What exactly does `uv run` guarantee before executing `uvicorn`, `alembic`, or `pytest`, and what does it explicitly *not* guarantee or manage?
2. A teammate's `uv run uvicorn app.main:app` fails with a `pydantic.ValidationError` about a missing `database_url` field. Is this a uv problem? What should they check first?
3. Why does ExpenseFlow's importable package get named `app` rather than `expenseflow_api`, and what's the difference between that name and the project's distribution name in `pyproject.toml`?
4. Explain the difference between `uv run uvicorn ...` executed from inside `packages/api` versus `uv run --package expenseflow-api uvicorn ...` executed from the workspace root. Are they using different environments?
5. Why does ExpenseFlow's test suite run the real Alembic migration chain against the test database instead of using `Base.metadata.create_all()`?
6. Where does `pydantic-settings` get `DATABASE_URL` from, and what takes precedence if both a `.env` file and a real OS environment variable set it to different values?
7. Which dependency group would `httpx` belong to if ExpenseFlow's team decided to reintroduce it purely as a one-off script dependency (à la Chapter 9's `backfill_currency.py`), rather than a project dependency?

---

## Hands-On Exercise

**Goal:** Reproduce ExpenseFlow's dev loop end to end locally, including deliberately triggering and diagnosing the configuration-boundary confusion this chapter describes.

1. Starting from a uv workspace matching Chapter 12's layout (`packages/api` + `packages/shared`), add `packages/api/.env.example` with placeholder values for `DATABASE_URL`, `ENVIRONMENT`, and `FEATURE_READ_NOTES_COLUMN`, and add `.env` to `.gitignore`.
2. Copy `.env.example` to `.env`, filling in a real connection string for a local PostgreSQL instance (or one running in a container).
3. Run `uv run uvicorn app.main:app --reload` from inside `packages/api` and confirm the server starts and `/docs` loads in a browser.
4. Temporarily rename or delete `.env` and re-run the same command. Confirm you get a `pydantic` validation error, not a uv/dependency error — read the traceback closely enough to see exactly which field is reported missing.
5. Restore `.env`, then add a new nullable column to a model in `app/models.py`, run `uv run alembic revision --autogenerate -m "add test column"`, review the generated file, and run `uv run alembic upgrade head`.
6. Set `DATABASE_URL` as a real environment variable pointing at a separate test database, and run `uv run pytest -v` — confirm the migration fixture in `conftest.py` applies the full chain (including the column you just added) before any test runs.
7. Write a small `Makefile` (or `justfile`) wrapping `dev`, `migrate`, `test`, and `check` targets as shown in Section 7, and confirm `make check` runs ruff, mypy, and pytest in sequence.

---

## Further Reading

- [uv Guides — Working with FastAPI-style projects](https://docs.astral.sh/uv/guides/) — the Guides section covers project workflows this chapter builds on directly.
- [uv Concepts — Projects](https://docs.astral.sh/uv/concepts/) — the underlying model behind `uv run`'s environment-sync behavior used throughout this chapter.
- [FastAPI SQL Databases Guide](https://fastapi.tiangolo.com/tutorial/sql-databases/) — the application-side wiring this chapter's `uv run` commands are driving.
- [pydantic-settings Documentation](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) — the full reference for `.env` loading, precedence rules, and validation behavior discussed in Section 4.
- [This repo's Alembic course](../../Databases/alembic-course/00-index.md) — the schema/migration mechanics behind every `alembic` command run in this chapter.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./12-workspaces-and-monorepos.md">← Previous: Workspaces & Monorepos</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./14-docker-integration.md">Next: Docker Integration →</a>
</div>
