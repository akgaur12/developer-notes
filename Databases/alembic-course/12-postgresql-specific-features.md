# PostgreSQL-Specific Features

[Chapter 11](./11-data-migrations.md) treated the database underneath ExpenseFlow as a fairly generic SQL engine — `UPDATE`, `INSERT`, a `NOT NULL` constraint, all things that read almost identically whether you're running Postgres, MySQL, or SQLite. That genericity is one of Alembic's real strengths: most `op.*` directives compile down to whichever dialect `env.py` is connected to, and most migrations never need to know or care which database they're ultimately running against. This chapter is about the other side of that coin. ExpenseFlow runs on PostgreSQL in production, and Postgres has a set of genuinely excellent features — native enums, `JSONB`, `UUID`, generated columns, partial indexes, trigram fuzzy search — that have no equivalent in the generic SQL Alembic abstracts over. Using them well means knowing exactly where Alembic's `op.*` catalog stops covering you, and reaching for `op.execute()` deliberately rather than by accident.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Create and safely evolve a native PostgreSQL `ENUM` type from a migration, including the transaction-block restriction on `ALTER TYPE ... ADD VALUE`.
- Add a `JSONB` column and build a `GIN` index over it for efficient containment queries.
- Add a `UUID` primary key to a new table, and explain what makes converting an *existing* table's primary key to `UUID` a substantially harder problem.
- Add a PostgreSQL generated (computed) column using raw DDL via `op.execute()`.
- Enable a PostgreSQL extension (`pg_trgm`) from a migration and build a trigram `GIN` index for fuzzy text search.
- Create a partial index and explain when a `WHERE` clause on an index earns its keep.
- State the general rule for when a PostgreSQL feature requires `op.execute()` instead of a first-class `op.*` directive.

---

## Prerequisites for This Chapter

This chapter builds on [Chapter 8: Writing Manual Migrations](./08-writing-manual-migrations.md) (the `op.*` directive catalog) and [Chapter 11: Data Migrations](./11-data-migrations.md) (`op.get_bind()`, batching, and the discipline of not depending on live ORM models inside a migration). Familiarity with PostgreSQL's own data types and DDL — this repo's [PostgreSQL course](../postgresql-course/00-index.md) covers that ground in depth — will make the "why" behind several sections land faster, though this chapter explains each feature's PostgreSQL-side behavior as it goes.

---

## 1. Why PostgreSQL-Specific DDL Needs Special Handling

Alembic's `op.*` operations are built on top of SQLAlchemy's dialect system, which is precisely why `op.create_table()`, `op.add_column()`, and `op.create_index()` work unchanged whether `env.py` is pointed at Postgres, MySQL, or SQLite (modulo Chapter 13's batch-mode caveats for SQLite specifically). That portability comes from every one of those operations compiling down to whatever the connected dialect understands as its equivalent DDL.

The catch: some PostgreSQL features simply have no equivalent in other databases, and no amount of dialect abstraction can paper over a feature MySQL or SQLite doesn't have at all. A native `ENUM` type as its own named database object, `JSONB`'s specific binary storage and indexing model, `CREATE EXTENSION`, generated columns, and partial indexes are all real PostgreSQL capabilities with real Postgres-specific SQL syntax — and for exactly that reason, Alembic's cross-dialect `op.*` catalog either represents them through PostgreSQL-dialect-specific constructs (`sqlalchemy.dialects.postgresql`) or doesn't represent them as a first-class operation at all, leaving `op.execute()` — literally, "run this SQL string" — as the honest, correct tool.

This isn't a gap in Alembic's design so much as an accurate reflection of reality: **a migration that uses `CREATE EXTENSION IF NOT EXISTS pg_trgm` is a PostgreSQL migration**, full stop, and pretending otherwise by hunting for a nonexistent portable `op.enable_extension()` call would only hide that fact, not remove it. The right mental model, going into this chapter: reach for `sqlalchemy.dialects.postgresql` types and `postgresql_*` keyword arguments where Alembic/SQLAlchemy expose them (Sections 2, 3, and 6 all do), and reach for plain `op.execute()` with raw SQL the moment you hit something they don't (Sections 2.3, 4, and 5 all do too).

| PostgreSQL feature | Alembic/SQLAlchemy first-class support? | What you actually write |
|---|---|---|
| Native `ENUM` type creation | Yes — `sqlalchemy.dialects.postgresql.ENUM` | `op.add_column()` with a `postgresql.ENUM(...)` type |
| `ALTER TYPE ... ADD VALUE` | No | `op.execute("ALTER TYPE ... ADD VALUE ...")`, with a transaction caveat (Section 2.3) |
| `JSONB` column | Yes — `sqlalchemy.dialects.postgresql.JSONB` | `op.add_column()` with a `postgresql.JSONB` type |
| `GIN` index | Partially — `op.create_index(..., postgresql_using="gin")` | `op.create_index()` with `postgresql_using`/`postgresql_ops` kwargs |
| `UUID` column / `gen_random_uuid()` default | Yes — `sqlalchemy.Uuid` / `postgresql.UUID` | `op.add_column()` with a server default via `sa.text(...)` |
| Generated (computed) columns | No | `op.execute("ALTER TABLE ... ADD COLUMN ... GENERATED ALWAYS AS (...) STORED")` |
| `CREATE EXTENSION` | No | `op.execute("CREATE EXTENSION IF NOT EXISTS ...")` |
| Partial index (`WHERE` clause) | Yes — `op.create_index(..., postgresql_where=...)` | `op.create_index()` with `postgresql_where` |

---

## 2. Native ENUM Types

### 2.1 The situation: a legacy free-text column alongside the categories table

ExpenseFlow has had a proper `categories` table with a `category_id` foreign key on `expenses` since Chapter 7 — that's the correct, fully normalized, user-extensible design, and it isn't going anywhere. Separately, though, an old `expenses.category` `VARCHAR` column has quietly survived since before Chapter 7: it was added in the very first version of the app as a quick way to tag an expense's rough classification for a dashboard chart, and even after `categories`/`category_id` arrived, nobody got around to removing it — a denormalized column, left over from an earlier iteration, that a couple of reporting queries still read directly for speed. Its values have drifted over time — `"Food"`, `"food"`, `"Foods"`, and even a stray `"FOOD "` with a trailing space all show up, because nothing ever constrained it.

The team decides to finally clean this up: since the *legacy* column's real value set is small and fixed (unlike the open-ended, user-managed `categories` table), it's a textbook case for a native PostgreSQL `ENUM` type rather than free text.

### 2.2 Creating the ENUM type and converting the column

```python
"""convert expenses.category to native enum

Revision ID: f4b9d2e6a1c7
Revises: e1a5f7c3b8d2
Create Date: 2026-03-18 09:12:03.441822
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = "f4b9d2e6a1c7"
down_revision = "e1a5f7c3b8d2"
branch_labels = None
depends_on = None

expense_category = postgresql.ENUM(
    "food", "transport", "utilities", "entertainment", "other",
    name="expense_category",
)

expenses = sa.table(
    "expenses",
    sa.column("id", sa.Integer),
    sa.column("category", sa.String),
)


def upgrade() -> None:
    bind = op.get_bind()

    # Normalize existing free-text values before the type even exists —
    # trimming whitespace and lowercasing so every row will actually fit
    # one of the enum's fixed labels.
    bind.execute(sa.text("UPDATE expenses SET category = lower(trim(category))"))
    bind.execute(
        sa.text(
            "UPDATE expenses SET category = 'other' "
            "WHERE category IS NULL OR category NOT IN "
            "('food', 'transport', 'utilities', 'entertainment', 'other')"
        )
    )

    # create_type=True (the default) issues CREATE TYPE expense_category AS ENUM (...)
    expense_category.create(bind, checkfirst=True)

    op.execute(
        "ALTER TABLE expenses "
        "ALTER COLUMN category TYPE expense_category "
        "USING category::expense_category"
    )


def downgrade() -> None:
    op.execute("ALTER TABLE expenses ALTER COLUMN category TYPE VARCHAR USING category::text")
    expense_category.drop(op.get_bind(), checkfirst=True)
```

Two details worth slowing down on. First, the data cleanup (`UPDATE ... lower(trim(...))`, then a catch-all `'other'` for anything that still doesn't match) runs *before* the type conversion — this is Chapter 11's backfill-before-constrain discipline again, applied to a type change instead of a `NOT NULL` constraint: the `USING category::expense_category` cast will fail outright on the very first row whose value isn't one of the five labels, so every row has to already be valid before that `ALTER COLUMN` runs. Second, `USING category::expense_category` is doing real work — a plain `ALTER COLUMN ... TYPE` without a `USING` clause only succeeds when PostgreSQL has an implicit cast between the old and new types, which it does not between arbitrary text and a freshly created enum; the explicit cast tells Postgres exactly how to reinterpret each existing value.

### 2.3 Adding a new value later — the transaction-block caveat

Six months after this migration, the team wants to add a sixth category, `"travel"`. The natural-looking migration:

```python
def upgrade() -> None:
    op.execute("ALTER TYPE expense_category ADD VALUE 'travel'")
```

This has a sharp edge that catches almost everyone the first time. On PostgreSQL versions before 12, `ALTER TYPE ... ADD VALUE` could not run inside a transaction block *at all* — and Alembic, by default, wraps every migration's `upgrade()` in exactly one transaction. On PostgreSQL 12 and later, it's now permitted inside a transaction, but with a caveat that still trips people up: **the newly added value cannot be used in the same transaction that added it.** If the very next statement in the same migration tries to `INSERT` or filter on `'travel'`, it will fail — Postgres requires the `ADD VALUE` to actually commit before that label is usable anywhere.

Two ways to handle it safely, in order of preference:

1. **Split it into two migrations.** Add the enum value in one migration, deployed and committed on its own. Any migration that actually *uses* `'travel'` (say, a data migration reclassifying some rows into it) goes into a later, separate migration. This is the simplest, most broadly compatible option and needs no special connection handling.
2. **Run the `ALTER TYPE` outside Alembic's wrapping transaction**, using an autocommit-style execution option on the connection, if your PostgreSQL version and driver support it and you specifically need it in the same deploy as code that will use the new value immediately:

```python
def upgrade() -> None:
    connection = op.get_bind()
    with connection.execution_options(isolation_level="AUTOCOMMIT"):
        connection.execute(sa.text("ALTER TYPE expense_category ADD VALUE 'travel'"))
```

Option 1 is what ExpenseFlow's team actually does — it's simpler to reason about, doesn't depend on driver-specific autocommit behavior, and costs nothing beyond one extra, entirely ordinary migration file.

---

## 3. JSONB Columns and GIN Indexing

### 3.1 Adding a flexible `metadata` column

Different expense sources want to attach different structured extras — a mileage-tracking integration wants `{"miles": 42, "rate_cents": 67}`, a corporate-card import wants `{"card_last4": "4242", "merchant_category_code": "5812"}` — and none of it is common enough across every expense to justify its own column. This is exactly what `JSONB` is for: schemaless, per-row structured data, stored in an efficient binary format (unlike plain `JSON`, which stores an exact text copy and re-parses it on every access) and directly indexable.

```python
def upgrade() -> None:
    op.add_column(
        "expenses",
        sa.Column("metadata", postgresql.JSONB(astext_type=sa.Text()), nullable=True),
    )


def downgrade() -> None:
    op.drop_column("expenses", "metadata")
```

### 3.2 Indexing it with GIN

A `JSONB` column with no index still supports containment queries (`metadata @> '{"card_last4": "4242"}'`), but every such query does a full table scan without one. PostgreSQL's `GIN` (Generalized Inverted Index) index type is built specifically for this — it indexes every key/value pair inside the JSON structure, making containment and key-existence lookups efficient at scale:

```python
def upgrade() -> None:
    op.create_index(
        "ix_expenses_metadata_gin",
        "expenses",
        ["metadata"],
        postgresql_using="gin",
    )
```

This is a case where Alembic's dialect-specific keyword arguments *do* cover you fully — `op.create_index()` accepts `postgresql_using` directly, so no raw `op.execute()` is needed here at all. With this index in place, a query like `SELECT * FROM expenses WHERE metadata @> '{"card_last4": "4242"}'` can use the index instead of scanning every row, which matters as `expenses` grows into the millions of rows Chapter 11's scenario discussed.

---

## 4. UUID Primary Keys

### 4.1 The easy case: a brand-new table

ExpenseFlow's `receipts` table (added in [Chapter 10](./10-branches-and-merge-migrations.md)'s branch/merge scenario) has been live for only a few weeks and holds a small number of rows. The team wants receipt IDs to be non-guessable — receipt URLs are shared with users directly (`/receipts/{id}/download`), and a sequential integer ID leaks how many receipts exist and lets anyone probe adjacent IDs. Because the table is new and low-volume, converting its primary key to `UUID` now, before it accumulates real production scale, is the easy version of this problem:

```python
"""convert receipts primary key to UUID

Revision ID: a3c7e9f1b5d4
Revises: f4b9d2e6a1c7
Create Date: 2026-03-19 14:02:55.108733
"""
from alembic import op
import sqlalchemy as sa

revision = "a3c7e9f1b5d4"
down_revision = "f4b9d2e6a1c7"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # gen_random_uuid() is built into PostgreSQL 13+ core — no extension
    # needed. On PostgreSQL < 13, enable pgcrypto first:
    #   op.execute("CREATE EXTENSION IF NOT EXISTS pgcrypto")
    op.add_column(
        "receipts",
        sa.Column(
            "uuid_id",
            sa.Uuid(as_uuid=True),
            server_default=sa.text("gen_random_uuid()"),
            nullable=False,
        ),
    )
    op.create_unique_constraint("uq_receipts_uuid_id", "receipts", ["uuid_id"])

    # Swap primary keys: retire the old integer PK, rename the old column
    # out of the way, and promote uuid_id to be the new `id`.
    op.drop_constraint("receipts_pkey", "receipts", type_="primary")
    op.alter_column("receipts", "id", new_column_name="legacy_id")
    op.drop_constraint("uq_receipts_uuid_id", "receipts", type_="unique")
    op.alter_column("receipts", "uuid_id", new_column_name="id")
    op.create_primary_key("receipts_pkey", "receipts", ["id"])
    # legacy_id is deliberately kept for now (nullable, unused) rather than
    # dropped in this same migration — see the note below on why.


def downgrade() -> None:
    raise NotImplementedError(
        "Downgrading a primary-key type swap is not supported — see Section 4.2. "
        "Restore from a pre-migration backup if you need to reverse this."
    )
```

The old integer primary key isn't dropped outright in this same migration — it's renamed to `legacy_id` and left in place, nullable and unused, for one deploy cycle, in case any in-flight code, cached link, or external system still references the old integer ID during the rollout window. A later, separate migration drops `legacy_id` once the team has confirmed nothing reads it anymore. The important teaching point here isn't the exact DDL choreography — it's that **even the "easy" case involves several ordered steps** (add the new column, default it, add a temporary unique constraint, swap the primary key, rename the old column out of the way rather than dropping it immediately), because a primary key is never "just a column" — it's a column with a unique constraint every foreign key elsewhere in the schema may be depending on, and this same rename-before-drop caution is the same expand/contract instinct Chapter 14 turns into a full pattern.

### 4.2 The harder case: an existing, heavily-referenced primary key

Converting `receipts.id` was tractable specifically because `receipts` is small and — critically — **nothing else in the schema has a foreign key pointing at it**. Contrast that with a hypothetical, much harder version of this same problem: converting `expenses.id` itself to `UUID`. `expenses.id` is referenced by `receipts.expense_id` (a foreign key), was referenced by `expense_tags.expense_id` before that table's own design (Chapter 8), and would be referenced by any future table built the same way. Converting a primary key that other tables reference means:

- Every referencing foreign-key column must also change type, in careful lockstep — you cannot have `receipts.expense_id` still be an `Integer` pointing at an `expenses.id` that's now a `UUID`.
- On a large, heavily-written table, the type change itself (an `ALTER COLUMN ... TYPE`) is exactly the kind of operation Chapter 14 explains can hold a long lock or force a full table rewrite.
- Every application code path, every cached ID, every external system that stored an ExpenseFlow expense ID needs to handle the format change — a much bigger blast radius than a table nobody outside the app has ever referenced.

This is squarely a zero-downtime, multi-deploy problem — expand (add a new `UUID` column alongside the old integer one, backfill it, dual-write from the application during a transition period, migrate every referencing FK to point at the new column), not something to attempt in a single migration on a live table with real traffic. Chapter 14 covers the general expand/contract choreography this kind of change requires; the practical takeaway for this chapter is narrower: **converting a primary key's type is easy exactly when nothing references it yet, and gets progressively harder in direct proportion to how many foreign keys, how much data, and how much external state depend on the old value.**

---

## 5. Generated Columns

PostgreSQL 12+ supports **generated columns**: a column whose value is computed automatically from other columns in the same row, stored on disk (`GENERATED ALWAYS AS (...) STORED`), and recomputed automatically whenever the source columns change. ExpenseFlow's `expenses.amount_cents` is the canonical stored value, but several reporting queries would rather read a plain decimal dollar amount directly instead of dividing by 100 in every query:

```python
def upgrade() -> None:
    op.execute(
        "ALTER TABLE expenses "
        "ADD COLUMN amount_dollars NUMERIC(12, 2) "
        "GENERATED ALWAYS AS (amount_cents / 100.0) STORED"
    )


def downgrade() -> None:
    op.execute("ALTER TABLE expenses DROP COLUMN amount_dollars")
```

There is no `op.*` directive for generated columns at all — `GENERATED ALWAYS AS (...) STORED` is exactly the kind of PostgreSQL-only DDL Section 1's table flagged, so `op.execute()` with the raw statement is simply the correct, only tool. A few things worth knowing about generated columns before reaching for them: they cannot be written to directly (any `INSERT`/`UPDATE` that tries to set `amount_dollars` explicitly is rejected by Postgres), they add storage and a small write-time computation cost since the value is materialized (`STORED`, not computed on read), and — usefully — they can be indexed exactly like any other column, since the value genuinely lives on disk.

---

## 6. Extensions, Partial Indexes, and `pg_trgm` Fuzzy Search

### 6.1 Soft deletes and a partial index

ExpenseFlow is adding soft deletes — instead of `DELETE FROM expenses`, a row gets a `deleted_at` timestamp set, so it can be restored and so deletion is auditable. Almost every query in the application (`SELECT * FROM expenses WHERE user_id = ... AND deleted_at IS NULL`) will now carry that same `deleted_at IS NULL` predicate. A **partial index** — an index that only covers rows matching a `WHERE` condition — is a direct fit: it's smaller than a full-table index (deleted rows, which the application almost never queries, aren't indexed at all), and Postgres's query planner can use it precisely for the query shape the app actually runs.

```python
def upgrade() -> None:
    op.add_column("expenses", sa.Column("deleted_at", sa.DateTime(timezone=True), nullable=True))
    op.create_index(
        "ix_expenses_active",
        "expenses",
        ["user_id", "expense_date"],
        postgresql_where=sa.text("deleted_at IS NULL"),
    )


def downgrade() -> None:
    op.drop_index("ix_expenses_active", table_name="expenses")
    op.drop_column("expenses", "deleted_at")
```

`postgresql_where` is, like `postgresql_using` in Section 3.2, a first-class keyword argument `op.create_index()` understands directly — no raw SQL needed. The resulting index only contains entries for rows where `deleted_at IS NULL`, which on a table where soft-deleted rows accumulate over time (refunded expenses, corrected duplicate entries) can be meaningfully smaller — and therefore faster to scan and cheaper to maintain on every write — than an equivalent full-table index would be.

### 6.2 Enabling `pg_trgm`

ExpenseFlow's users want to search expenses by description with typo tolerance — searching `"restarant"` should still surface `"Restaurant - team lunch"`. PostgreSQL's `pg_trgm` extension supports exactly this: it indexes overlapping three-character sequences ("trigrams") of text, letting similarity and `ILIKE '%...%'`-style queries run efficiently instead of falling back to a full sequential scan for every search.

`pg_trgm` is not part of PostgreSQL's default set of enabled features — it ships with the standard distribution but must be explicitly enabled per database with `CREATE EXTENSION`, which, like generated columns, has no portable `op.*` equivalent and is written directly with `op.execute()`:

```python
"""enable pg_trgm and index expenses.description for fuzzy search

Revision ID: b6d1f3a8e2c9
Revises: a3c7e9f1b5d4
Create Date: 2026-03-20 11:47:29.502016
"""
from alembic import op
import sqlalchemy as sa

revision = "b6d1f3a8e2c9"
down_revision = "a3c7e9f1b5d4"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    op.create_index(
        "ix_expenses_description_trgm",
        "expenses",
        ["description"],
        postgresql_using="gin",
        postgresql_ops={"description": "gin_trgm_ops"},
    )


def downgrade() -> None:
    op.drop_index("ix_expenses_description_trgm", table_name="expenses")
    # Deliberately not dropping the pg_trgm extension itself — another
    # object elsewhere in the database may depend on it, and dropping an
    # extension is a database-wide action, not something one migration's
    # downgrade should own. See Section 6.3.
```

Note the `postgresql_ops` argument on `op.create_index()`, paired with `postgresql_using="gin"` — this is the specific combination that tells Postgres to build the `GIN` index using `pg_trgm`'s own trigram operator class (`gin_trgm_ops`) rather than the default operator class for the column's type. With this index in place, a query like `SELECT * FROM expenses WHERE description % 'restarant'` (the `%` operator being `pg_trgm`'s similarity match) or `SELECT * FROM expenses WHERE description ILIKE '%restaurant%'` can both use the trigram index instead of scanning every row.

### 6.3 Why extensions deserve a lighter touch on downgrade

Notice `downgrade()` above removes the index but not the extension. `CREATE EXTENSION`/`DROP EXTENSION` operate at the level of the whole database, not a single table or column — if some other migration, or some other part of the application, also relies on `pg_trgm` being enabled (another fuzzy-search index elsewhere, for instance), a downgrade that drops the extension out from under it breaks something this migration was never responsible for creating. The safe default is: enable extensions idempotently (`IF NOT EXISTS`) on the way up, and generally leave them enabled on the way down — treat extension enablement as closer to a one-way, database-level setting than a per-migration, per-table change.

---

## Real-World Scenario

ExpenseFlow's support team starts fielding a recurring complaint: users searching their expense history for a specific purchase — "that Uber ride last month," "the AWS bill" — get zero results whenever they mistype or use different capitalization/spacing than what was originally entered (`"aws"` vs. `"AWS"` vs. `"A.W.S."`). The existing search is a plain `description ILIKE '%query%'` with no index at all, which is both slow (a full sequential scan on every keystroke of a live-search UI) and exact-match-only in spirit — `ILIKE` handles case but nothing else.

The fix rolls out as two coordinated pieces, both covered in this chapter. First, the migration from Section 6.2: enable `pg_trgm`, build a `GIN` trigram index on `expenses.description`. Second, an application-layer change (outside Alembic entirely) to switch the search query from `ILIKE '%query%'` to `pg_trgm`'s similarity operator (`description % :query`, ordered by `similarity(description, :query) DESC`), which tolerates typos and word-order differences the old exact substring match never could.

The team tests this on a staging database seeded with a production-sized copy of `expenses` (a practice Chapter 15 formalizes as part of the CI/CD pipeline) before shipping to production, specifically to confirm two things: that `CREATE EXTENSION IF NOT EXISTS pg_trgm` succeeds cleanly (extensions occasionally require superuser or specific role grants, which is exactly the kind of thing you want to discover in staging, not production), and that the `GIN` index build itself — which on a multi-million-row table can take real time and hold a lock while building, unless built with the `CONCURRENTLY` option outside a transaction, a Chapter 14 topic — completes in an acceptable window. Both check out, the migration ships, and the very next week's support-ticket volume for "search isn't finding my expense" drops close to zero.

---

## Best Practices

- Reach for `sqlalchemy.dialects.postgresql` types (`ENUM`, `JSONB`, `UUID`) and Alembic's `postgresql_*` keyword arguments wherever they exist — they're the correct, well-supported path for Postgres-specific behavior Alembic *does* model.
- Reach for `op.execute()` with raw SQL deliberately, not reluctantly, for the genuine gaps (`CREATE EXTENSION`, generated columns, `ALTER TYPE ... ADD VALUE`) — this is the intended tool for those cases, not a workaround.
- Always normalize/validate data before casting a free-text column into a new `ENUM` type — the cast fails outright on the first non-matching value.
- Split "add a new enum value" and "use that new enum value" into separate migrations to sidestep the same-transaction restriction entirely, rather than reaching for autocommit connection tricks unless you specifically need both in one deploy.
- Treat converting an *existing*, referenced primary key's type as a Chapter 14-style, multi-deploy, expand/contract project — never as a single migration on a live table.
- Enable extensions idempotently (`IF NOT EXISTS`) and think twice before a downgrade disables one — other schema objects may depend on it.

---

## Common Mistakes

- **Casting a free-text column directly to a new `ENUM` type without first cleaning the data**, and having the migration fail mid-deploy on the first row whose value doesn't match one of the enum's labels.
- **Adding a new enum value and trying to use it in the same migration/transaction**, hitting Postgres's restriction that a value cannot be used until the `ALTER TYPE ... ADD VALUE` that created it has committed.
- **Reaching for `op.execute()` everywhere out of habit**, even for things Alembic already models cleanly (`JSONB`, `GIN` via `postgresql_using`, partial indexes via `postgresql_where`) — losing the readability and dialect-awareness those first-class options provide for no reason.
- **Underestimating a primary-key type conversion** by copying the "easy case" pattern from Section 4.1 onto a heavily-referenced table without accounting for every foreign key that points at it.
- **Forgetting `CREATE EXTENSION` can fail due to insufficient database privileges** in some managed-Postgres environments, and discovering that only when a production deploy fails, instead of catching it against a staging environment configured the same way.
- **Dropping a shared extension in a single migration's `downgrade()`** without checking whether anything else in the schema still depends on it.

---

## Summary

- Some PostgreSQL features are fully modeled by Alembic/SQLAlchemy's Postgres dialect (`ENUM`, `JSONB`, `UUID`, `GIN`/partial indexes via keyword arguments); others (`CREATE EXTENSION`, generated columns, `ALTER TYPE ... ADD VALUE`) have no portable `op.*` equivalent and correctly require raw `op.execute()` (Section 1).
- Converting a free-text column to a native `ENUM` requires cleaning existing data first, then an explicit `USING` cast; adding a new enum value later must respect the same-transaction usage restriction, usually by splitting into two migrations (Section 2).
- `JSONB` columns store flexible per-row data efficiently and can be indexed with `GIN` for fast containment queries, using `op.create_index(..., postgresql_using="gin")` (Section 3).
- Converting a new, unreferenced table's primary key to `UUID` is a tractable multi-step migration; converting an existing, heavily-referenced primary key is a much harder, Chapter 14-style zero-downtime project (Section 4).
- Generated columns compute and store a value from other columns automatically, and are created with raw `ALTER TABLE ... GENERATED ALWAYS AS (...) STORED` via `op.execute()` (Section 5).
- Partial indexes (`postgresql_where`) index only a subset of rows for a smaller, more targeted index; `pg_trgm`, enabled via `CREATE EXTENSION`, powers efficient fuzzy-text search through a trigram `GIN` index (Section 6).

---

## Knowledge Check

1. Why does Alembic not provide a portable `op.enable_extension()` operation, and what's the correct way to enable `pg_trgm` from a migration?
2. What happens if you try to cast a free-text column directly to a new native `ENUM` type when some existing rows contain values outside the enum's label set?
3. You add a new label to an existing enum type with `ALTER TYPE ... ADD VALUE` and, in the very next line of the same migration, try to insert a row using that new label. What happens, and why?
4. Why was converting `receipts.id` to `UUID` considered the "easy case" in Section 4, and what specifically would make the same conversion on `expenses.id` much harder?
5. What's the difference between a `JSONB` column with no index and one with a `GIN` index, in terms of what kind of query benefits?
6. Explain, in your own words, what a partial index is and why `ix_expenses_active`'s `WHERE deleted_at IS NULL` clause makes it smaller and more targeted than an equivalent full-table index.
7. Why does this chapter's `pg_trgm` migration avoid dropping the extension itself in `downgrade()`, even though it drops the index it created?

---

## Hands-On Exercise

**Goal:** Add and verify three PostgreSQL-specific features against a local ExpenseFlow database: a `JSONB` column with a `GIN` index, a partial index, and `pg_trgm` fuzzy search.

1. Write a migration adding `expenses.metadata` as `JSONB` (Section 3.1), apply it, then manually `UPDATE` a few rows with sample JSON (e.g., `{"source": "mileage_import", "miles": 42}`).
2. Add the `GIN` index from Section 3.2. Run `EXPLAIN SELECT * FROM expenses WHERE metadata @> '{"source": "mileage_import"}';` in `psql` before and after creating the index, and compare the query plan — confirm the plan changes from a sequential scan to an index-based plan once the index exists (on a small local table Postgres may still choose a sequential scan regardless — note this and explain why, referencing what you know about the query planner's cost estimates on small tables).
3. Add `expenses.deleted_at` and the partial index `ix_expenses_active` from Section 6.1. Soft-delete a few rows (`UPDATE expenses SET deleted_at = now() WHERE id IN (...)`), then run `\d expenses` in `psql` to confirm the partial index's `WHERE` clause is reflected in its definition.
4. Enable `pg_trgm` and build the trigram index from Section 6.2. Insert a row with `description = 'Team lunch at The Republic Restaurant'`, then run `SELECT description, similarity(description, 'republik restarant') FROM expenses ORDER BY similarity(description, 'republik restarant') DESC LIMIT 5;` and confirm the misspelled query still surfaces the row.
5. Write and apply the ENUM conversion migration from Section 2.2 against a small set of test rows with intentionally messy `category` values (mixed case, whitespace, one invalid value), and confirm every row ends up as a valid `expense_category` label after the migration runs.
6. Downgrade all of the above migrations in reverse order (`alembic downgrade`, one step at a time), confirming each step's `downgrade()` runs cleanly against the state left by the step after it.

---

## Further Reading

- [PostgreSQL Data Types documentation](https://www.postgresql.org/docs/current/datatype.html) — the authoritative reference for `ENUM`, `JSONB`, and `UUID` types used throughout this chapter.
- [PostgreSQL `ALTER TABLE` documentation](https://www.postgresql.org/docs/current/sql-altertable.html) — covers `ALTER TYPE` transaction semantics, generated columns, and the locking behavior of type changes referenced in Sections 2 and 5.
- [Alembic Operation Reference (`op.*`)](https://alembic.sqlalchemy.org/en/latest/ops.html) — the `postgresql_using`/`postgresql_where`/`postgresql_ops` keyword arguments used across this chapter.
- [Alembic Cookbook](https://alembic.sqlalchemy.org/en/latest/cookbook.html) — PostgreSQL-specific recipes, including enum handling patterns.
- [SQLAlchemy 2.0 ORM documentation](https://docs.sqlalchemy.org/en/20/orm/) — background on `sqlalchemy.dialects.postgresql` types (`ENUM`, `JSONB`, `UUID`) as used in application models versus migrations.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./11-data-migrations.md">← Previous: Data Migrations</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./13-sqlite-batch-migrations.md">Next: SQLite & Batch Migrations →</a>
</div>
