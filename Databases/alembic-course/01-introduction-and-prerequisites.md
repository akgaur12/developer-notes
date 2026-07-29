# Introduction & Prerequisites

This is the opening chapter of the Alembic & Database Schema Migrations course. The [course index](./00-index.md) laid out the roadmap: twenty chapters carrying you from "what is a schema migration?" to designing a zero-downtime production deployment for a real FastAPI + SQLAlchemy 2.0 + PostgreSQL application. Before any of that — before `alembic init`, before revision IDs, before `upgrade()`/`downgrade()` — you need a settled, honest picture of the problem Alembic exists to solve, and you need your local environment ready to follow along. This chapter builds that picture using **ExpenseFlow**, the expense-tracking API that will grow, branch, and get refactored across every remaining chapter of this course, and closes with Alembic installed and verified against ExpenseFlow's starting schema.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Define what a database schema migration is, and distinguish it clearly from a one-off manual `ALTER TABLE` statement.
- Explain, using a concrete failure scenario, why "drop the database and recreate it from the models" stops working the moment a database holds real data.
- Preview the distinction between a *schema* migration and a *data* migration (full depth arrives in [Chapter 2](./02-core-concepts.md)).
- Honestly self-assess whether your SQL, SQLAlchemy, and command-line/Docker background is sufficient to start this course productively.
- Describe ExpenseFlow's starting shape: a FastAPI + SQLAlchemy 2.0 (async) + PostgreSQL project with `users` and `expenses` tables.
- Install Alembic into a Python project and verify the installation against a real project skeleton.

---

## Prerequisites for This Chapter

This is the first chapter of the course, so there is no prior chapter to depend on — but there is real background this course assumes rather than teaches. Specifically, you should already be comfortable with:

- **SQL fundamentals.** You should be able to read and reason about `CREATE TABLE`, `ALTER TABLE`, primary keys, foreign keys, and indexes without translation. This course does not re-teach SQL; it teaches how to *version* the SQL your schema is built from.
- **SQLAlchemy's ORM, SQLAlchemy 2.0 style specifically.** You should recognize declarative models built with `DeclarativeBase`, `Mapped[...]` type annotations, and `mapped_column(...)` — the modern SQLAlchemy 2.0 idiom, as opposed to the older `Column(...)`-only style. If your SQLAlchemy experience predates 2.0, skim the [SQLAlchemy 2.0 ORM documentation](https://docs.sqlalchemy.org/en/20/orm/) before continuing; the syntax difference is small, but this course's code samples use it throughout.
- **Command-line and Docker comfort.** Running a shell command, installing a Python package, and starting a PostgreSQL container (`docker run` or `docker compose up`) should already be routine. You do not need Docker expertise beyond that.
- **Basic familiarity with FastAPI, helpful but not mandatory.** ExpenseFlow is a FastAPI application, and some code samples show FastAPI dependency wiring, but this course teaches Alembic, not FastAPI — you can follow along even if FastAPI itself is new, as long as the SQLAlchemy parts make sense.

**Self-assessment checklist** — confirm you can honestly check off each item before moving to Section 1:

- [ ] I can write a `CREATE TABLE` statement with a primary key and at least one foreign key from memory, or close to it.
- [ ] I recognize this style of SQLAlchemy model and know roughly what each line does:
  ```python
  class User(Base):
      __tablename__ = "users"
      id: Mapped[int] = mapped_column(primary_key=True)
      email: Mapped[str] = mapped_column(String(255), unique=True)
  ```
- [ ] I can run `pip install <package>` and `docker run <image>` without needing a refresher.
- [ ] I understand, at least loosely, that changing a live application's database schema is a different and riskier operation than changing a local SQLite file nobody depends on yet.

If any box is unchecked, spend thirty minutes closing that gap first — this course moves quickly once these basics are assumed settled, exactly as later chapters assume you remember this one.

---

## 1. What Is a Schema Migration?

A **schema migration** is a versioned, repeatable, forward-and-backward set of instructions that transforms a database's structure — its tables, columns, indexes, constraints, and types — from one known state to another known state. Read that definition again slowly, because every word in it is load-bearing:

- **Versioned**: each migration has an identity (a revision ID) and a defined position in a sequence, not just "some SQL someone ran once."
- **Repeatable**: the same migration, run against the same starting state, produces the same resulting state, every time, on every environment — a developer's laptop, a CI runner, staging, production.
- **Forward-and-backward**: a migration should (ideally) know how to apply its change (**upgrade**) and how to undo it (**downgrade**), not just how to move in one direction.
- **Known state to known state**: a migration always has a defined starting point and a defined ending point, which is precisely what lets you ask "which state is this database currently in?" and get a real answer.

Concretely, for ExpenseFlow's very first schema, a migration is the thing that takes a **completely empty PostgreSQL database** and turns it into a database containing a `users` table and an `expenses` table with a foreign key between them — and later, a different migration takes *that* state and adds a `categories` table on top of it. Each migration is a small, self-contained, ordered step. The full sequence of them, applied in order, *is* your schema's entire history.

Contrast this with what most people do the first time they touch a database from application code: hand-write a `CREATE TABLE` statement (or let the ORM's `Base.metadata.create_all()` do it), run it once, and move on. That works exactly once, on an empty database, with nobody else depending on the result. It has no version, no defined "undo," and no way to answer "has environment X already had this change applied?" other than manually inspecting the database. A migration tool exists to turn that one-off, unrepeatable act into a durable, trackable artifact — which is the entire subject of this course.

### 1.1 A migration is not the same thing as a raw `ALTER TABLE`

It's worth being precise here, because the two get conflated constantly. A migration typically *contains* one or more DDL statements (`ALTER TABLE`, `CREATE INDEX`, and so on), but the migration itself is a higher-level unit: it's the DDL *plus* its identity, its position in the sequence, its upgrade/downgrade pair, and (in Alembic specifically) the Python code that generates that DDL portably across database backends. Running a raw `ALTER TABLE ADD COLUMN currency VARCHAR(3)` directly against production, by hand, over an SSH session, is not a migration in this sense — even though it changes the schema — because nothing recorded that it happened, nothing else can reliably reproduce it, and nothing can undo it in a controlled way. Chapter 9's discussion of the `alembic_version` table returns to exactly this distinction from a different angle: Alembic's entire value proposition rests on the database itself recording which migrations have already been applied.

---

## 2. Life Without Migrations: The Drop-and-Recreate Pain

To feel why this matters — not just intellectually agree with it — walk through what happens to a small team building ExpenseFlow *without* a migration tool.

### 2.1 Day one: everything is fine

The ExpenseFlow team starts with two SQLAlchemy 2.0 models:

```python
# app/models.py
from datetime import datetime
from sqlalchemy import String, ForeignKey, Numeric
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)


class Expense(Base):
    __tablename__ = "expenses"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    amount_cents: Mapped[int]
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    description: Mapped[str] = mapped_column(String(500))
    expense_date: Mapped[datetime]
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

During early development, whenever a model changes, the whole team does the same thing: drop the local database, and let SQLAlchemy recreate every table from scratch via `Base.metadata.create_all(engine)`. This is fast, it's simple, and — critically — **it works fine here only because there is no data anyone cares about yet.** Every teammate's local database is disposable.

### 2.2 The moment it stops working

ExpenseFlow launches. Real users sign up, real expenses get entered. Now a developer needs to add a `category_id` foreign key to `expenses`. The old workflow — drop tables, recreate from models — would work *mechanically* (SQLAlchemy would happily generate the new schema), but it would also **destroy every user's account and every expense they've ever logged**, because `DROP TABLE` doesn't ask if you meant it. This is the wall every application hits, without exception, the first time its database stops being disposable:

```mermaid
flowchart TD
    A["Local dev, no real data\nDrop & recreate: fine"] --> B["App launches\nReal users, real expenses"]
    B --> C{"Need to change the schema"}
    C -->|"Drop & recreate"| D["Data loss.\nEvery user, every expense: gone."]
    C -->|"Hand-write ALTER TABLE\nand run it once, manually"| E["Works today.\nNo record of what ran, where,\nor how to undo it."]
    E --> F["Second developer makes\na different manual change\non a different environment"]
    F --> G["Environments drift silently.\nStaging schema != Production schema.\nNobody can say which is 'correct'."]
```

The team's next instinct is usually to hand-write the `ALTER TABLE` statement and run it manually against production once. That avoids data loss, but introduces a new, quieter failure mode: **nothing records that it happened.** Six weeks later, a second developer, working against a stale local database, writes a *different* manual fix for a *related* problem and runs it against staging. Now production, staging, and every developer's laptop have three subtly different schemas, and there is no artifact anywhere — no file, no table, no log — that authoritatively says what changed, in what order, or how to reproduce any of it on a fresh environment. This is **schema drift**, and it is the specific disease that a migration tool is the cure for.

### 2.3 What a migration tool actually buys you

Alembic's job, stated as plainly as possible, is to replace both bad options above with a third one: a small, ordered, checked-in Python file per schema change, applied through a single command (`alembic upgrade head`), with the database itself recording exactly which changes have already been applied. Every environment — a laptop, CI, staging, production — reaches the *same* schema by running the *same* sequence of migrations, and the question "is this environment up to date?" has an actual, mechanical answer instead of a guess. That is the entire pitch of this course, and everything from Chapter 2 onward is about doing this well rather than just doing it at all.

---

## 3. Schema Migration vs. Data Migration: A First Look

You will see this distinction sharpened considerably in [Chapter 2](./02-core-concepts.md) and given a full chapter of its own in [Chapter 11](./11-data-migrations.md), but it's worth previewing now so the vocabulary doesn't feel new later:

| | Schema migration | Data migration |
|---|---|---|
| **Changes** | The *structure* — tables, columns, types, constraints, indexes | The *contents* — the actual row values stored in existing tables |
| **ExpenseFlow example** | Adding a `category_id` column to `expenses` | Backfilling every existing expense's `currency` column to `"USD"` |
| **Typical operation** | `ALTER TABLE`, `CREATE INDEX`, `CREATE TABLE` | `UPDATE`, `INSERT` |
| **Risk profile** | Can lock tables; usually near-instant on empty/new tables | Can be slow and resource-intensive proportional to row count |
| **Alembic's role** | Its primary, most polished use case (`op.add_column`, etc.) | Fully supported, but requires more care (Chapter 11) |

Both kinds of change live inside Alembic's migration files and both get an upgrade/downgrade pair — Alembic does not force you to use a separate tool for data changes. But they behave very differently in production: a schema change to an empty new column is nearly instant, while backfilling millions of existing rows can hold locks or run for minutes. Keeping this distinction in your head from day one will make Chapter 11 and Chapter 14 (zero-downtime deployments) click much faster.

---

## 4. Meet ExpenseFlow

**ExpenseFlow** is the running example for this entire course: a small team's expense-tracking API, built with FastAPI, SQLAlchemy 2.0 (async), and PostgreSQL, deployed with Docker and a CI/CD pipeline. Its schema starts simple and grows chapter by chapter, exactly the way a real application's schema grows over the life of a real product — which is the whole point of using one continuous example instead of a different toy each chapter.

At the point this course picks it up, ExpenseFlow has exactly two tables:

```mermaid
erDiagram
    USERS ||--o{ EXPENSES : "logs"
    USERS {
        int id PK
        string email
        string hashed_password
        datetime created_at
    }
    EXPENSES {
        int id PK
        int user_id FK
        int amount_cents
        string currency
        string description
        datetime expense_date
        datetime created_at
    }
```

- **`users`** — `id`, `email` (unique), `hashed_password`, `created_at`.
- **`expenses`** — `id`, `user_id` (foreign key to `users.id`), `amount_cents` (money stored as integer cents, not float, to avoid rounding error — a deliberate modeling choice worth noting), `currency`, `description`, `expense_date`, `created_at`.

Over the course of this book, this schema will grow a `categories` table and a `category_id` foreign key (Chapter 7, via autogenerate), a many-to-many `tags`/`expense_tags` relationship and a `receipts` table (Chapter 8, hand-written), a `monthly_budgets` table (Chapter 10, as part of a branch/merge scenario), data backfills and seeded lookup data (Chapter 11), PostgreSQL-specific column types like `ENUM`, `JSONB`, and `UUID` (Chapter 12), and finally a production column rename executed with zero downtime (Chapter 14). Every one of those changes is a migration — this course is, in a real sense, the story of ExpenseFlow's `alembic/versions/` directory filling up one revision at a time.

ExpenseFlow's application code talks to PostgreSQL through an async SQLAlchemy engine, since the FastAPI app itself is asynchronous end to end:

```python
# app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "postgresql+asyncpg://expenseflow:expenseflow@localhost:5432/expenseflow"

engine = create_async_engine(DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

Hold onto that `asyncpg` detail — it matters more than it looks like it should. Alembic's own migration runner, as you'll see in [Chapter 4](./04-migration-environment-env-py.md), typically drives migrations through a **synchronous** database connection even inside a fully async application like this one, using SQLAlchemy's `run_sync` bridge or a sync driver URL. This is not a bug or an oversight in ExpenseFlow's setup — it's how Alembic's execution model works today, and this course explains it honestly rather than glossing over it, because it's one of the most common points of confusion for teams adopting Alembic in an async-first codebase.

---

## 5. Installing Alembic

With ExpenseFlow's shape in mind, get Alembic installed. Alembic is a regular PyPI package with SQLAlchemy as its main dependency (Alembic is written and maintained by the same author as SQLAlchemy, Mike Bayer, and the two projects version and release in close coordination).

### 5.1 Project skeleton

A minimal ExpenseFlow project skeleton, before Alembic touches anything, looks like this:

```
expenseflow/
├── app/
│   ├── __init__.py
│   ├── database.py      # engine, session factory (Section 4)
│   ├── models.py         # User, Expense models (Section 2.1)
│   └── main.py           # FastAPI app
├── requirements.txt
└── docker-compose.yml     # local PostgreSQL for development
```

A `docker-compose.yml` for local PostgreSQL, matching the `DATABASE_URL` from Section 4:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: expenseflow
      POSTGRES_PASSWORD: expenseflow
      POSTGRES_DB: expenseflow
    ports:
      - "5432:5432"
    volumes:
      - expenseflow_pgdata:/var/lib/postgresql/data

volumes:
  expenseflow_pgdata:
```

### 5.2 Installing the package

```bash
pip install alembic sqlalchemy asyncpg psycopg2-binary fastapi uvicorn
```

Notice two PostgreSQL drivers here, not one: `asyncpg` for the FastAPI application's async engine, and `psycopg2-binary` for Alembic's synchronous migration runner (per the note at the end of Section 4 — Chapter 4 shows exactly where each one gets used). This dual-driver setup is completely normal for an async SQLAlchemy application using Alembic and not something to try to "fix" away.

Pin versions in `requirements.txt` for anything beyond local experimentation:

```
alembic==1.13.2
sqlalchemy==2.0.31
asyncpg==0.29.0
psycopg2-binary==2.9.9
fastapi==0.111.0
uvicorn==0.30.1
```

Chapter 16 revisits *why* pinning Alembic and SQLAlchemy versions specifically matters (autogenerate behavior and migration file templates have changed across minor versions historically) — for now, just get comfortable with an explicit, checked-in version list rather than an unpinned `pip install alembic`.

### 5.3 Verifying the installation

Confirm the CLI is on your `PATH` and reports a sane version:

```bash
alembic --version
```

You should see output resembling `alembic 1.13.2`. If you get a `command not found`, the most common cause is installing into a different Python environment/virtualenv than the one your shell's `PATH` resolves `alembic` from — verify with `which alembic` (or `where alembic` on Windows) and `which python`, and make sure they point into the same virtual environment.

At this point you have *not* yet run `alembic init` — that command, and the full `env.py`/`alembic.ini` structure it creates, is the entire subject of [Chapter 4](./04-migration-environment-env-py.md). This chapter's goal is narrower and deliberately so: confirm Alembic is installed correctly, alongside a working FastAPI + SQLAlchemy 2.0 + PostgreSQL project skeleton, before you touch the migration environment itself.

---

## 6. What's Next: From Concept to Practice

You now have three things in place: a precise definition of what a schema migration is (Section 1), a visceral understanding of the pain it prevents (Section 2), and a working, verified Alembic installation sitting next to ExpenseFlow's `users`/`expenses` models (Section 5). It's tempting, at this point, to jump straight to `alembic init` and start generating revisions — and if this were a quick-start guide, that would be the very next step.

This course deliberately takes a slower path first. [Chapter 2](./02-core-concepts.md) builds the complete conceptual vocabulary — revision, `down_revision`, head, base, branch, merge point, and the migration graph they form together — entirely without touching the command line, because every one of those words will be used constantly, without re-explanation, from Chapter 4 onward. [Chapter 3](./03-architecture-and-internals.md) then goes one level deeper still, into the actual runtime components (`ScriptDirectory`, `MigrationContext`, the `op` module) that make those concepts real. Only after both of those chapters does [Chapter 4](./04-migration-environment-env-py.md) hand you `alembic init` and a line-by-line `env.py` walkthrough — by which point the file you're reading will already make structural sense, rather than looking like boilerplate to copy and hope works.

If your instinct right now is impatience — "just show me the commands" — that instinct is worth noticing and setting aside deliberately. The engineers who can only pattern-match Alembic commands are the ones who get stuck the first time a migration does something unexpected in production; the ones who understand the model underneath can reason their way through it. That is the entire premise this course is built on, stated plainly here at the start rather than left implicit.

---

## Real-World Scenario

ExpenseFlow's team is three developers: Priya (backend lead), Marcus, and a new hire, Dana, who joined in her second week of the project. In her first sprint, Dana is asked to add a `currency` column to `expenses` — a small, seemingly harmless change. Following the pattern she'd used at her previous job (a much earlier-stage prototype with no real users), Dana adds `currency: Mapped[str]` to the `Expense` model, then runs a local script that calls `Base.metadata.drop_all(engine)` followed by `Base.metadata.create_all(engine)` against her local database to "pick up the change."

It works instantly on her machine, because her local database only had test data she'd created that morning. She commits the model change and opens a pull request. Priya, reviewing it, asks a simple but pointed question: "How does this get applied to staging, where there's three weeks of real test data the QA team has been building up test scenarios against?" Dana doesn't have an answer — her workflow never needed one, because her local environment was always disposable.

This is the exact moment described abstractly in Section 2.2, now happening to a specific, sympathetic new teammate instead of a hypothetical "the team." Priya's response is not to blame Dana — the drop/recreate instinct is completely reasonable coming from disposable local environments — but to use this as the trigger to adopt Alembic *before* the `currency` column change ships anywhere near staging or production. The rest of this course is, in effect, Priya walking Dana (and you) through exactly how to do this properly: Chapter 4 sets up `env.py` against this same `DATABASE_URL`; Chapter 5 and 6 create and apply ExpenseFlow's first real revisions; and by Chapter 8, the `currency` column Dana wanted to add on day one gets added correctly — as a reviewable, reversible, environment-independent migration, not a destructive local script.

---

## Best Practices

- **Treat "drop and recreate" as a local-development-only technique, and retire it the moment any environment holds data you can't casually lose** — including a shared staging database used by QA, not just production.
- **Install Alembic and pin its version (and SQLAlchemy's) in your project's dependency file from day one**, even before you've written your first migration — Chapter 16 explains why version drift between teammates' Alembic installs causes subtle problems later.
- **Set up your local PostgreSQL via Docker Compose from the start**, matching your team's production database engine — testing schema changes against a different database engine than production (e.g., SQLite for "speed") introduces its own class of surprises, covered fully in Chapter 13.
- **Keep two database drivers in mind for an async SQLAlchemy project**: an async driver (`asyncpg`) for the application, and a sync driver (`psycopg2` or similar) that Alembic's migration runner will use — this isn't a workaround, it's the normal shape of an async app using Alembic today.
- **Read the self-assessment checklist honestly before starting Chapter 2.** This course assumes SQL and SQLAlchemy 2.0 fluency; time spent shoring up either now pays for itself many times over in later chapters.

---

## Common Mistakes

- **Continuing to drop-and-recreate the schema after real data exists**, simply because it's the habit that worked during early prototyping — the single most common way teams "discover" they need a migration tool, usually right after losing something they wish they hadn't.
- **Running one-off manual `ALTER TABLE` statements against production "just this once"** with no record of what ran or in what order — this doesn't cause immediate pain the way data loss does, but it seeds exactly the schema drift described in Section 2.2, which surfaces confusingly much later.
- **Skipping the SQLAlchemy 2.0 style refresher** if your prior experience is with the older `Column(...)`-only API — the `Mapped[...]`/`mapped_column(...)` style used throughout this course reads similarly but has real behavioral differences (e.g., how nullability is inferred from the type annotation) worth understanding before Chapter 4's `env.py` walkthrough.
- **Installing Alembic without pinning a version**, then being confused months later when a teammate's autogenerate output looks subtly different — covered in depth in Chapter 16, but avoidable entirely by pinning from the start.
- **Assuming the async driver used by the FastAPI app (`asyncpg`) is also what Alembic uses under the hood.** Alembic's default migration runner is synchronous; not installing a sync driver (`psycopg2-binary` or similar) alongside `asyncpg` is a setup mistake you won't notice until Chapter 4.

---

## Summary

- A **schema migration** is a versioned, repeatable, forward-and-backward transformation of a database's structure — distinct from a one-off manual DDL statement (Section 1).
- "Drop and recreate the schema" is a fine workflow for disposable local databases, but it destroys real data the moment any environment holds data worth keeping — this is the exact pain Alembic exists to eliminate (Section 2).
- **Schema migrations** change structure; **data migrations** change row contents — both live inside Alembic, but they carry very different operational risk profiles (Section 3), a distinction deepened in Chapters 2 and 11.
- **ExpenseFlow** is this course's running example: a FastAPI + SQLAlchemy 2.0 (async) + PostgreSQL expense tracker starting with `users` and `expenses`, whose schema grows chapter by chapter (Section 4).
- Alembic migrations typically run through a **synchronous** database connection even in a fully async application — an intentional design detail, not a bug, that this course explains rather than hides (Section 4).
- Alembic installs via `pip install alembic`, alongside both an async driver for your app and a sync driver for Alembic's migration runner; pin versions and verify with `alembic --version` (Section 5).

---

## Knowledge Check

1. In your own words, what makes a schema migration different from a single manually-run `ALTER TABLE` statement, even if they produce the identical schema change?
2. Walk through the drop-and-recreate failure scenario from Section 2 and explain the specific moment it stops being a safe workflow.
3. What is schema drift, and why does "just run the fix manually on production" tend to produce it over time?
4. Give one example of a schema migration and one example of a data migration using ExpenseFlow's tables, and explain why their operational risk profiles differ.
5. Why does an async FastAPI + SQLAlchemy 2.0 application still typically need a synchronous database driver installed alongside `asyncpg`?
6. What two things does `alembic --version` verify for you, and what would a `command not found` error most likely indicate about your environment?

---

## Hands-On Exercise

Set up ExpenseFlow's starting project skeleton locally, and confirm Alembic is correctly installed against it, before Chapter 2 introduces the vocabulary you'll need to actually create a migration.

1. **Create the project directory structure**:
   ```bash
   mkdir -p expenseflow/app
   cd expenseflow
   ```

2. **Write `app/models.py`** with the `Base`, `User`, and `Expense` classes exactly as shown in Section 2.1.

3. **Write `app/database.py`** with the async engine and session factory exactly as shown in Section 4.

4. **Write `docker-compose.yml`** exactly as shown in Section 5.1, and start PostgreSQL:
   ```bash
   docker compose up -d
   docker compose ps
   ```
   Confirm the `db` service reports as healthy/running.

5. **Create a virtual environment and install dependencies**:
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   pip install alembic sqlalchemy asyncpg psycopg2-binary fastapi uvicorn
   ```

6. **Freeze and pin your dependencies**:
   ```bash
   pip freeze > requirements.txt
   ```

7. **Verify Alembic is installed and resolvable**:
   ```bash
   alembic --version
   which alembic
   which python
   ```
   Confirm both paths resolve inside the same `.venv` directory.

8. **Sanity-check the models import cleanly** (a quick way to catch typos before Chapter 4 needs this to work):
   ```bash
   python -c "from app.models import Base, User, Expense; print(Base.metadata.tables.keys())"
   ```
   You should see `dict_keys(['users', 'expenses'])` printed.

If all eight steps succeed, you have a working ExpenseFlow skeleton with Alembic installed and verified — the exact starting point [Chapter 2](./02-core-concepts.md) and [Chapter 4](./04-migration-environment-env-py.md) build on next.

---

## Further Reading

- [Alembic Official Documentation](https://alembic.sqlalchemy.org/en/latest/) — the canonical reference you'll return to constantly throughout this course.
- [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html) — the official getting-started walkthrough, useful to skim alongside Chapter 4.
- [SQLAlchemy 2.0 ORM Documentation](https://docs.sqlalchemy.org/en/20/orm/) — the declarative modeling style (`Mapped[...]`, `mapped_column(...)`) used throughout this course.
- [SQLAlchemy 2.0 Migration Notes (async)](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html) — background on SQLAlchemy 2.0's async support, relevant to the `asyncpg`/sync-driver split discussed in Section 4.
- [FastAPI SQL Databases Guide](https://fastapi.tiangolo.com/tutorial/sql-databases/) — official FastAPI documentation on wiring SQLAlchemy into a FastAPI application, the same pattern ExpenseFlow uses.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./00-index.md">← Previous: Index</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./02-core-concepts.md">Next: Core Concepts →</a>
</div>
