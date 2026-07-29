# The Migration Environment: env.py & alembic.ini

[Chapter 3](./03-architecture-and-internals.md) opened the box on Alembic's internals — the `MigrationContext`, the `ScriptDirectory`, the `Operations` class behind the `op.*` calls, and the offline/online split that lets the same migration script either execute directly against a live connection or emit reviewable SQL text. One piece of that picture was deliberately left abstract: *how* `env.py` actually gets wired to a real application's SQLAlchemy models and a real database URL. This chapter makes that concrete. We're going to run `alembic init` against ExpenseFlow's FastAPI + SQLAlchemy 2.0 + PostgreSQL codebase, read every line of the files it generates, and then modify those files so Alembic knows about ExpenseFlow's `users` and `expenses` tables and reads its connection string from application configuration instead of a hardcoded string in a config file. By the end, you'll have a working migration environment ready for the first real revision, which is where [Chapter 5](./05-revisions-and-version-history.md) picks up.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Run `alembic init` to scaffold a migration environment, and explain the purpose of every file it generates.
- Identify which `alembic.ini` settings matter in practice (`script_location`, `sqlalchemy.url`, `file_template`, `prepend_sys_path`) and which are safe to leave at their defaults.
- Read `env.py` line by line and explain the roles of `target_metadata`, `run_migrations_offline()`, `run_migrations_online()`, `context.configure()`, and `context.run_migrations()`.
- Rewire `env.py` so the database URL comes from environment variables / application settings rather than being hardcoded in `alembic.ini`.
- Register a SQLAlchemy `Base.metadata` containing all of an app's models with Alembic's `target_metadata`.
- Explain the `script.py.mako` template and how it produces every future migration file.
- Diagnose and resolve the async-engine-vs-sync-migration mismatch that trips up nearly every FastAPI + `asyncpg` project the first time they add Alembic.

---

## Prerequisites for This Chapter

This chapter assumes you've read:

- [Chapter 2: Core Concepts](./02-core-concepts.md) — you should already be comfortable with the terms *revision*, *upgrade*, *downgrade*, *head*, and *migration graph*.
- [Chapter 3: Architecture & Internals](./03-architecture-and-internals.md) — you should know what `context.configure()` and `context.run_migrations()` do conceptually, and the difference between offline (`--sql`) and online execution modes. This chapter puts those two functions into a real, runnable file.

You'll also need a working ExpenseFlow-style project skeleton: a FastAPI app using SQLAlchemy 2.0's `Mapped[...]` / `mapped_column(...)` declarative style, an async engine talking to PostgreSQL via `asyncpg`, and application settings loaded from environment variables (a `pydantic-settings` `BaseSettings` subclass, or equivalent). If you don't have that yet, the structure below is written so you can copy it directly.

---

## 1. ExpenseFlow's Starting Point

Before running `alembic init`, let's fix exactly what ExpenseFlow looks like at this stage of the course. It has two tables — `users` and `expenses` — defined as SQLAlchemy 2.0 models, no migrations yet, and a database that (so far) only exists because someone ran `Base.metadata.create_all()` once by hand or in a test fixture. That's precisely the "life without migrations" starting point Chapter 1 described, and it's about to end.

```
expenseflow/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── core/
│   │   ├── __init__.py
│   │   └── config.py          # Settings — reads DATABASE_URL from environment
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py            # declarative Base
│   │   └── session.py         # async engine + AsyncSession factory
│   └── models/
│       ├── __init__.py        # imports every model so Base.metadata is complete
│       ├── user.py
│       └── expense.py
├── requirements.txt
└── .env
```

`app/core/config.py`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # The app talks to Postgres asynchronously, via asyncpg.
    database_url: str = "postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow"

    # A separate, synchronous URL, used ONLY by Alembic migrations (Section 5).
    # If unset, env.py derives one from database_url.
    database_url_sync: str | None = None


settings = Settings()
```

`app/db/base.py`:

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    """Shared declarative base for every ExpenseFlow model."""
```

`app/db/session.py`:

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.core.config import settings

engine = create_async_engine(settings.database_url, echo=False, future=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

`app/models/user.py`:

```python
from datetime import datetime

from sqlalchemy import DateTime, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.db.base import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
```

`app/models/expense.py`:

```python
from datetime import date, datetime

from sqlalchemy import Date, DateTime, ForeignKey, Integer, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.db.base import Base


class Expense(Base):
    __tablename__ = "expenses"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id", ondelete="CASCADE"), nullable=False, index=True
    )
    amount_cents: Mapped[int] = mapped_column(Integer, nullable=False)
    currency: Mapped[str] = mapped_column(String(3), nullable=False, default="USD")
    description: Mapped[str | None] = mapped_column(String(500), nullable=True)
    expense_date: Mapped[date] = mapped_column(Date, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
```

`app/models/__init__.py`:

```python
# Importing every model here ensures Base.metadata is fully populated the
# moment anything imports `app.models` — including Alembic's env.py.
from app.models.expense import Expense
from app.models.user import User

__all__ = ["User", "Expense"]
```

That last file matters more than it looks. Chapter 7 will spend real time on the failure mode where a model exists in the codebase but was never imported anywhere Alembic's `env.py` reaches, so `Base.metadata` never learns about it and autogenerate silently ignores an entire table. Get in the habit now of treating `app/models/__init__.py` as the single place that guarantees every model is registered.

---

## 2. Installing and Initializing Alembic

With `alembic` installed (`pip install alembic`, already covered in Chapter 1, alongside `sqlalchemy` and an async Postgres driver like `asyncpg`), scaffold the migration environment from the project root:

```bash
cd expenseflow
alembic init alembic
```

The single argument, `alembic`, is the directory name Alembic will create for the migration environment — it's a convention, not a requirement. Some teams call it `migrations` instead (`alembic init migrations`); either works, as long as `alembic.ini`'s `script_location` (Section 3) points at whichever name you chose. This course uses `alembic`, matching the overwhelming majority of real-world projects and Alembic's own documentation.

### 2.1 What gets generated

```
expenseflow/
├── alembic/
│   ├── versions/                  # empty for now — every future migration file lives here
│   ├── env.py                     # the migration environment script (Section 4)
│   ├── script.py.mako             # template used to generate new revision files (Section 6)
│   └── README
├── alembic.ini                    # Alembic's configuration file (Section 3)
├── app/
│   └── ...                        # unchanged, from Section 1
└── requirements.txt
```

Two files do essentially all of the work: `alembic.ini` (static configuration Alembic reads before doing anything else) and `env.py` (a real Python script Alembic executes on every single invocation — `upgrade`, `downgrade`, `revision`, `history`, all of it). The `versions/` directory is where every migration script will land, one file per revision, starting in Chapter 5. `script.py.mako` is the Mako template used to generate each new file's boilerplate. `README` is a one-line placeholder with no runtime effect — safe to ignore or delete.

---

## 3. `alembic.ini`: The Fields That Matter

`alembic.ini` is an [INI-format](https://docs.python.org/3/library/configparser.html) file, read once at startup by `alembic.config.Config`. Most of what `alembic init` generates is boilerplate logging configuration you'll never touch. A handful of settings under the `[alembic]` section, however, directly affect how migrations are located, named, and connected.

| Setting | Default (generated) | What it controls |
|---|---|---|
| `script_location` | `alembic` | The directory containing `env.py`, `script.py.mako`, and `versions/`. Must match whatever name you passed to `alembic init`. |
| `sqlalchemy.url` | `driver://user:pass@localhost/dbname` (placeholder) | The database connection string Alembic uses by default. **Section 5 shows why you should not put a real URL here** for anything beyond local, throwaway experimentation. |
| `file_template` | `%%(rev)s_%%(slug)s` (or similar, version-dependent) | Controls migration filenames. ExpenseFlow uses `%%(year)d%%(month).2d%%(day).2d_%%(rev)s_%%(slug)s` so files sort chronologically on disk — see Section 3.1. |
| `prepend_sys_path` | `.` | Directories prepended to `sys.path` before `env.py` runs. This is what lets `env.py` `import app.models` even though it lives in a different directory than your application code. |
| `timezone` | unset | If set (e.g. `UTC`), timestamps embedded in migration filenames/docstrings use this timezone instead of local system time — worth setting explicitly so a distributed team doesn't get filenames stamped in whoever's laptop timezone happened to run `alembic revision`. |
| `truncate_slug_length` | `40` | Maximum length of the human-readable "slug" derived from your `-m "message"` text, used in filenames. |
| `version_path_separator` | `os` | How Alembic joins path segments when `version_locations` (multiple migration directories — a later-chapter topic) is used. |
| `[loggers]` / `[handlers]` / `[formatters]` sections | boilerplate | Standard Python `logging` configuration for Alembic's own log output. Rarely needs editing beyond adjusting a log level. |

### 3.1 A concrete `alembic.ini` for ExpenseFlow

```ini
[alembic]
script_location = alembic
prepend_sys_path = .
file_template = %%(year)d%%(month).2d%%(day).2d_%%(rev)s_%%(slug)s
timezone = UTC
truncate_slug_length = 40

; sqlalchemy.url is intentionally left as a placeholder — env.py overrides
; it at runtime from application settings (Section 5). Never commit a real
; production connection string here.
sqlalchemy.url = driver://user:pass@localhost/dbname

[post_write_hooks]
; hooks = black
; black.type = console_scripts
; black.entrypoint = black
; black.options = -l 79 REVISION_SCRIPT_FILENAME

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARNING
handlers = console
qualname =

[logger_sqlalchemy]
level = WARNING
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %%(levelname)-5.5s [%%(name)s] %%(message)s
datefmt = %%H:%%M:%%S
```

The commented-out `[post_write_hooks]` block is worth knowing about even unused: it lets Alembic run a formatter (commonly `black`) against every newly generated migration file automatically, which keeps autogenerated scripts from looking visually inconsistent with the rest of the hand-formatted codebase. ExpenseFlow's team enables this in Chapter 16 once their formatting conventions are locked down; there's no need to turn it on yet.

Note the double percent signs (`%%`) in `file_template` and the logging `format` string — Python's `configparser` treats a single `%` as the start of an interpolation token, so literal percent signs in an INI file must be escaped as `%%`.

---

## 4. `env.py`, Line by Line

`env.py` is not a passive config file — it is a plain Python script that Alembic imports and executes every time you run any `alembic` subcommand. That's the single most important thing to internalize about it: **anything you can do in Python, you can do in `env.py`**, including reading environment variables, importing your application's settings module, and constructing a database URL dynamically.

Here is what `alembic init` generates, unmodified:

```python
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = None

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Walking through each piece:

| Line(s) | What it does |
|---|---|
| `config = context.config` | Grabs the `Config` object Alembic built from `alembic.ini`, plus any `-x` options passed on the command line. |
| `fileConfig(config.config_file_name)` | Applies the `[loggers]`/`[handlers]`/`[formatters]` sections of `alembic.ini` to Python's `logging` module — this is why `alembic upgrade head` prints `INFO [alembic.runtime.migration] Running upgrade ...` lines. |
| `target_metadata = None` | The variable Chapter 3 called out as the bridge between Alembic and your ORM models — currently disconnected. Autogenerate (Chapter 7) compares the live database against exactly this object; leaving it `None` means autogenerate always reports "nothing changed," even when your models say otherwise. Section 5 fixes this. |
| `run_migrations_offline()` | Configures the `MigrationContext` with a bare URL string (`url=url`) and `literal_binds=True`, which renders SQL parameter values directly into the emitted SQL text instead of leaving `%s` placeholders — because in offline mode there's no live connection to bind parameters against. This is the function that runs when you pass `--sql` (Chapter 6). |
| `run_migrations_online()` | Builds a real SQLAlchemy `Engine` from the `[alembic]` section's `sqlalchemy.*` keys via `engine_from_config`, opens a `Connection`, and configures the `MigrationContext` against that live connection. `poolclass=pool.NullPool` is deliberate: migrations are short-lived, one-shot processes, so there's no benefit to connection pooling and a real cost to leaving idle pooled connections around after the process exits. |
| `context.begin_transaction()` / `context.run_migrations()` | The two calls Chapter 3 introduced: open a transaction (PostgreSQL DDL is transactional, so a failed migration rolls back cleanly), then walk the migration graph from the current revision to the target, executing each script's `upgrade()`/`downgrade()`. |
| `if context.is_offline_mode(): ... else: ...` | The branch that decides which of the two functions above actually runs, based on whether `--sql` was passed on the command line. |

---

## 5. Wiring `env.py` to ExpenseFlow

Two things need to change before this generated file is useful: `target_metadata` needs to point at ExpenseFlow's actual `Base.metadata`, and the connection URL needs to come from `app.core.config.settings` rather than the placeholder in `alembic.ini`.

```mermaid
flowchart TD
    A["alembic upgrade head\n(or any alembic command)"] --> B["alembic.ini parsed\ninto Config object"]
    B --> C["env.py executed"]
    C --> D["sys.path.append(os.getcwd())\nvia prepend_sys_path"]
    D --> E["import app.core.config.settings\nimport app.db.base.Base\nimport app.models  (registers User, Expense)"]
    E --> F["config.set_main_option('sqlalchemy.url', ...)\noverrides alembic.ini's placeholder"]
    F --> G["target_metadata = Base.metadata"]
    G --> H{context.is_offline_mode?}
    H -- yes --> I["run_migrations_offline()\nemits SQL text, no DB connection"]
    H -- no --> J["run_migrations_online()\nopens a real Connection, executes DDL"]
```

Here is `alembic/env.py`, modified for ExpenseFlow:

```python
import os
import sys
from logging.config import fileConfig

from sqlalchemy import engine_from_config, pool

from alembic import context

# alembic.ini's prepend_sys_path=. already does this in most setups, but
# being explicit here means env.py works even if invoked from a different
# working directory (e.g. from inside a Docker container's entrypoint script).
sys.path.append(os.getcwd())

from app.core.config import settings  # noqa: E402
from app.db.base import Base  # noqa: E402
import app.models  # noqa: E402,F401 — registers User, Expense on Base.metadata

config = context.config

# Override alembic.ini's placeholder URL with the real, environment-driven
# one. This is the one line that keeps a real connection string out of
# version control entirely.
config.set_main_option(
    "sqlalchemy.url",
    settings.database_url_sync or settings.database_url.replace("+asyncpg", ""),
)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Now target_metadata reflects every model imported via app.models —
# autogenerate (Chapter 7) compares the live database against this object.
target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Two design decisions in this file are worth defending explicitly, because both are easy to get wrong:

1. **The URL is set via `config.set_main_option()`, not by editing `alembic.ini` per environment.** `alembic.ini` is committed to version control; a real database password should never live there. By overriding the URL in Python, at runtime, from `settings` (which itself reads `DATABASE_URL`/`DATABASE_URL_SYNC` environment variables — populated differently in local dev, CI, staging, and production), the same `alembic.ini` file works unmodified everywhere. This is exactly the same instinct as never hardcoding secrets in application code — it's just easy to forget that a migration tool's config file is also "config," not "code," and deserves the same discipline.
2. **`import app.models` runs purely for its side effect.** Nothing in `env.py` calls `User` or `Expense` directly — the import exists solely to make Python execute `app/models/__init__.py`, which registers both classes' tables onto `Base.metadata`. Forgetting this import (or forgetting to add a new model to `app/models/__init__.py` later) is the single most common reason autogenerate mysteriously "doesn't see" a table — a mistake worth remembering now, well before Chapter 7 covers autogenerate in depth.

### 5.1 `.env` and `database_url_sync`

```bash
# .env — never commit this file; ExpenseFlow's .gitignore excludes it
DATABASE_URL=postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow
DATABASE_URL_SYNC=postgresql+psycopg2://expenseflow:expenseflow@localhost:5432/expenseflow
```

Why two URLs, differing only in driver, pointing at the same database? Section 7 explains this in full — it's the async-vs-sync gotcha the chapter title promised.

---

## 6. `script.py.mako`: The Template Behind Every Revision

Every time you run `alembic revision`, Alembic renders this Mako template to produce the new file in `versions/`. The generated, unmodified template:

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

# revision identifiers, used by Alembic.
revision: str = ${repr(up_revision)}
down_revision: Union[str, None] = ${repr(down_revision)}
branch_labels: Union[str, Sequence[str], None] = ${repr(branch_labels)}
depends_on: Union[str, Sequence[str], None] = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

The `${...}` placeholders are Mako expressions, substituted with real values at generation time: `${message}` becomes your `-m "..."` text, `${up_revision}` becomes the new revision's ID, `${down_revision}` becomes the ID of whatever revision it follows (or `None` for the first migration), and `${upgrades}`/`${downgrades}` become either `pass` (manual migration, Chapter 8) or actual `op.*` calls (autogenerated, Chapter 7). You rarely need to edit this template — the default is exactly what you want most of the time — but teams sometimes customize it to add a standard docstring header, an author field, or a linked ticket-ID comment convention. Chapter 5 dissects a real, filled-in migration file produced from this template.

---

## 7. The Async Engine vs. Sync Migration Gotcha

Here is the point that catches nearly every team building a FastAPI + `asyncpg` application the first time they wire up Alembic: **ExpenseFlow's application code is fully async — `AsyncSession`, `create_async_engine`, `await session.execute(...)` everywhere — but the `env.py` generated by `alembic init` is entirely synchronous.** `engine_from_config()` builds a plain, blocking `Engine`; `connectable.connect()` returns a plain, blocking `Connection`; `context.run_migrations()` runs synchronously. There is no `await` anywhere in the default `env.py`.

This is not an oversight, and you should not try to "fix" it by force. Alembic's `MigrationContext` and `Operations` machinery is fundamentally synchronous internally — migrations execute one DDL statement after another, often with Python control flow (loops, conditionals, data backfills in Chapter 11) interleaved between them, and that execution model was never rewritten to be `async`. Running actual schema migrations is also, deliberately, not a place where you want the throughput benefits of async concurrency — you want exactly one connection, running exactly one migration, in a strict, serial, and easy-to-reason-about order.

You have two supported ways to reconcile this with an async application:

### 7.1 Approach A: a separate sync driver URL, used only by Alembic (recommended)

This is what Section 5's `env.py` already does. Keep `asyncpg` for the application (`DATABASE_URL`), and give Alembic a second connection string using a synchronous driver — `psycopg2` (or `psycopg`, the modern psycopg 3, in its sync mode) — pointing at the exact same database:

```python
config.set_main_option(
    "sqlalchemy.url",
    settings.database_url_sync or settings.database_url.replace("+asyncpg", ""),
)
```

`settings.database_url.replace("+asyncpg", "")` is a convenient fallback for local development — it turns `postgresql+asyncpg://...` into plain `postgresql://...`, which SQLAlchemy resolves to whatever default sync driver is installed (typically `psycopg2`). In staging and production, ExpenseFlow sets `DATABASE_URL_SYNC` explicitly rather than relying on the fallback, since being explicit about which driver runs migrations is safer than an implicit default. This approach adds one dependency (`psycopg2-binary` or `psycopg[binary]` in `requirements.txt`) and otherwise requires no async code in `env.py` at all — `run_migrations_online()` stays exactly as generated.

### 7.2 Approach B: reuse the async engine via `run_sync`

If, for some reason, you need migrations to run through the exact same async engine configuration as the app (a less common need — perhaps a custom connection pool class or a proxy that only exists in async form), SQLAlchemy's `AsyncConnection.run_sync()` bridges the gap: it takes a plain synchronous function and runs it against the async connection's underlying sync-facing DBAPI cursor, inside the async event loop.

```python
import asyncio

from sqlalchemy.ext.asyncio import async_engine_from_config


def do_run_migrations(connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())
```

This is precisely the pattern Alembic ships as an alternate scaffold — `alembic init -t async alembic` generates an `env.py` with this structure out of the box, for teams that know upfront they want it. Note that even here, `context.run_migrations()` itself is still called synchronously, from inside `do_run_migrations` — `run_sync` changes *what kind of connection/engine you started from*, not the fundamentally synchronous nature of Alembic's migration execution.

### 7.3 Which one should ExpenseFlow use?

| | Approach A: separate sync URL | Approach B: async engine + `run_sync` |
|---|---|---|
| Extra dependency | A sync driver (`psycopg2-binary`) alongside `asyncpg` | None beyond what the app already uses |
| `env.py` complexity | Minimal — stock generated code, one URL override | Requires `asyncio.run()`, a nested sync function, `async_engine_from_config` |
| Connection pooling / proxy consistency with the app | Not guaranteed — a genuinely different engine | Guaranteed — same engine construction path as the app |
| Common in practice | Yes — the default recommendation, including in this course | Less common; used when app infra genuinely requires it |

ExpenseFlow's team goes with Approach A. It is simpler, it is what most production FastAPI + Alembic projects actually do, and — critically — a migration's correctness has nothing to do with which driver executed it; `psycopg2` and `asyncpg` both ultimately send the same SQL text to the same PostgreSQL server. The only reason to reach for Approach B is a specific infrastructure requirement (a custom async connection wrapper, for instance) that doesn't apply here.

---

## Real-World Scenario

ExpenseFlow's backend engineer, mid-sprint, runs `alembic init alembic`, accepts every default, and — in a hurry to unblock a teammate waiting on a shared staging database — pastes the staging Postgres URL, password included, directly into `alembic.ini`'s `sqlalchemy.url` line. It works. She commits it, along with the rest of the scaffold, because the CI pipeline needs `alembic.ini` present to run migrations and she doesn't want to block the PR on "figuring out secrets management" right now.

Three days later, a security-conscious teammate reviewing the PR flags it: the staging database password is now sitting in plaintext, in git history, in a config file, readable by anyone with repo access — including the CI logs that `cat alembic.ini` during a debugging step earlier that week. Worse, it's *staging's* password specifically, which happens to be reused (a separate bad practice, but a common one) for the read-replica credentials a data analyst uses — so the blast radius of "someone finds this git history" is bigger than "just re-run migrations against staging."

The fix is exactly Section 5 of this chapter: `alembic.ini` keeps a harmless placeholder URL (`driver://user:pass@localhost/dbname` — the same one Alembic generates by default, which is deliberately fake), and `env.py` overrides it at runtime via `config.set_main_option()`, reading from `settings.database_url_sync`, which in turn reads from an environment variable that's injected per-environment — a `.env` file locally (excluded from git via `.gitignore`), a Kubernetes secret or equivalent in staging/production, and a CI-provided environment variable in the pipeline. The rotated staging password is set once, in the secrets manager, and every consumer — the app, Alembic, CI — reads it from there. Nobody has to remember "don't commit the real URL" as a matter of discipline, because there's structurally nowhere in version control for it to leak into. This is, not coincidentally, the same 12-factor-app principle ("store config in the environment") the FastAPI app itself already follows for `settings.database_url` — Alembic just needs to be brought into that same discipline explicitly, since its default scaffold doesn't nudge you there on its own.

---

## Best Practices

- **Never commit a real connection string in `alembic.ini`.** Leave the generated placeholder in place and override the URL from `env.py`, reading from the same settings/environment-variable mechanism your application already uses (Section 5).
- **Import every model module from one central place** (`app/models/__init__.py`) and import that module — not individual model classes — from `env.py`, purely for the side effect of populating `Base.metadata` fully. A model that never gets imported is invisible to Alembic.
- **Keep a distinct, explicit sync driver URL for migrations** (`DATABASE_URL_SYNC`) rather than relying on string-replacement fallbacks in production — be explicit about which driver runs your DDL.
- **Set `poolclass=pool.NullPool`** (already the generated default) for the migration engine — a one-shot CLI process has no business holding a connection pool open after it exits.
- **Set `timezone` in `alembic.ini`** so migration file timestamps are consistent across a distributed team, regardless of individual machines' local time zones.
- **Treat `env.py` as real application code, not disposable scaffold.** It gets code review, it gets linted, and changes to it (like the URL-override logic here) should be understood by the whole team, not just whoever ran `alembic init` first.

---

## Common Mistakes

- **Hardcoding a real database URL — with credentials — into `alembic.ini`** and committing it, exactly as in this chapter's Real-World Scenario.
- **Leaving `target_metadata = None`** after wiring up an app, then being confused later when `alembic revision --autogenerate` (Chapter 7) generates an empty migration with no detected changes.
- **Forgetting to add a new model to `app/models/__init__.py`** when adding a table later in the course — the model exists in the codebase, SQLAlchemy can use it fine at the ORM layer, but Alembic never sees it because nothing imports it before `target_metadata` is read.
- **Trying to run migrations directly through `create_async_engine` and `await`-ing `context.run_migrations()`** — this doesn't work; `run_migrations()` is not a coroutine, and Alembic's core execution model is synchronous by design (Section 7).
- **Mixing up which URL is which** — pointing the app's `asyncpg` URL at a variable meant for the sync migration driver (or vice versa) produces a driver-mismatch error (`asyncpg` cannot be used by a plain sync `Engine`, and `psycopg2` cannot be used by `create_async_engine`) that's confusing to debug if you don't already know two separate URLs are expected.
- **Editing `script.py.mako` casually** without understanding Mako's templating syntax, breaking every future `alembic revision` call with a template-rendering error that's hard to diagnose from the traceback alone.

---

## Summary

- `alembic init alembic` scaffolds `alembic.ini`, `env.py`, `script.py.mako`, and an empty `versions/` directory (Section 2).
- `alembic.ini`'s meaningful settings are `script_location`, `sqlalchemy.url` (kept as a placeholder), `file_template`, `prepend_sys_path`, and `timezone` — most of the rest is logging boilerplate (Section 3).
- `env.py` is a real Python script, executed on every Alembic command, that sets `target_metadata`, branches on `context.is_offline_mode()`, and calls either `run_migrations_offline()` or `run_migrations_online()` (Section 4).
- Wiring `env.py` to a real app means importing `Base.metadata` (via importing all models) and overriding the connection URL from application settings using `config.set_main_option()`, never editing `alembic.ini` per environment (Section 5).
- `script.py.mako` is the Mako template rendered into every new revision file — rarely edited, but worth understanding since Chapter 5 dissects its output (Section 6).
- Alembic's migration execution is fundamentally synchronous; an async FastAPI app needs either a separate sync driver URL for migrations (recommended) or `AsyncConnection.run_sync()` to bridge an async engine into Alembic's sync execution model (Section 7).

---

## Knowledge Check

1. Why does `alembic.ini` keep a placeholder `sqlalchemy.url` in this chapter's setup, and where does the real URL actually come from at runtime?
2. What does `target_metadata = None` cause to happen (or not happen) when you later run `alembic revision --autogenerate`?
3. Explain, in your own words, why `import app.models` appears in `env.py` even though nothing in `env.py` calls `User` or `Expense` directly.
4. Why is `poolclass=pool.NullPool` an appropriate choice for the engine built inside `run_migrations_online()`, when it would be a poor choice for the engine your FastAPI app uses to serve requests?
5. ExpenseFlow's app uses `asyncpg`. Explain why `env.py`'s default `run_migrations_online()` cannot simply reuse `settings.database_url` unmodified, and name the two supported ways to resolve this.
6. What is the difference in responsibility between `alembic.ini` and `env.py` — that is, why does Alembic need both a static config file and an executable Python script?
7. If you added a new `Category` model tomorrow but forgot to import it in `app/models/__init__.py`, what would you observe (or fail to observe) the next time you ran an Alembic command?

---

## Hands-On Exercise

**Goal:** Scaffold a working Alembic environment against the ExpenseFlow skeleton from Section 1, wired to read its database URL from environment configuration, ready for Chapter 5's first revision.

1. **Set up the project skeleton.** Create the `app/` directory structure from Section 1 (`core/config.py`, `db/base.py`, `db/session.py`, `models/user.py`, `models/expense.py`, `models/__init__.py`) exactly as shown, adjusting only the database name/credentials to match your local setup.

2. **Start a local PostgreSQL instance** via Docker: `docker run -d --name expenseflow-pg -e POSTGRES_USER=expenseflow -e POSTGRES_PASSWORD=expenseflow -e POSTGRES_DB=expenseflow -p 5432:5432 postgres:16`.

3. **Install dependencies**: `pip install alembic sqlalchemy asyncpg psycopg2-binary pydantic-settings`.

4. **Create a `.env` file** at the project root with `DATABASE_URL` and `DATABASE_URL_SYNC`, matching Section 5.1.

5. **Run `alembic init alembic`** from the project root, and inspect the generated `alembic.ini`, `alembic/env.py`, and `alembic/script.py.mako` before changing anything.

6. **Edit `alembic.ini`** to add `prepend_sys_path`, `file_template`, and `timezone` as shown in Section 3.1, leaving `sqlalchemy.url` as the generated placeholder.

7. **Replace `alembic/env.py`** with the version from Section 5, adjusting the import paths if your package name differs from `app`.

8. **Verify the wiring** by running `alembic current` — it should execute without error and print nothing (no revisions applied yet, correctly, since `versions/` is still empty). If it raises an import error, fix the `sys.path`/import lines before continuing; if it raises a connection error, double-check `DATABASE_URL_SYNC` against your running Postgres container.

9. **Confirm `target_metadata` is wired correctly** by starting a Python shell in the project root and running `from app.db.base import Base; import app.models; print(sorted(Base.metadata.tables.keys()))` — you should see `['expenses', 'users']`.

You now have a fully wired migration environment with zero migrations in it — exactly the starting point [Chapter 5](./05-revisions-and-version-history.md) builds on to create ExpenseFlow's first two revisions.

---

## Further Reading

- [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html) — the official walkthrough of `alembic init`, `alembic.ini`, and `env.py`, covering the same ground as this chapter from Alembic's own documentation.
- [Alembic Cookbook](https://alembic.sqlalchemy.org/en/latest/cookbook.html) — includes recipes for connecting via environment variables and other `env.py` customization patterns.
- [Alembic Official Documentation](https://alembic.sqlalchemy.org/en/latest/) — full reference, including the `Config` and `EnvironmentContext` APIs referenced throughout this chapter.
- [SQLAlchemy 2.0 Migration Notes (async)](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html) — background on `create_async_engine`, `AsyncConnection.run_sync()`, and the async ORM changes referenced in Section 7.
- [FastAPI SQL Databases guide](https://fastapi.tiangolo.com/tutorial/sql-databases/) — for the application-side session/engine patterns this chapter's `env.py` wiring assumes.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./03-architecture-and-internals.md">← Previous: Architecture & Internals</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./05-revisions-and-version-history.md">Next: Revisions & Version History →</a>
</div>
