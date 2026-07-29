# Upgrade & Downgrade

[Chapter 5](./05-revisions-and-version-history.md) built ExpenseFlow's first real migration graph — two revisions, `create users table` followed by `create expenses table`, linked by `down_revision` into a two-node chain running from base to head — and taught you to inspect that graph with `history`, `current`, `heads`, and `show`. Everything so far has been read-only: no SQL has actually touched a database. This chapter changes that. We're going to run `alembic upgrade` and `alembic downgrade` for real, understand exactly what moves a database from one position in the graph to another, learn the difference between targeting a revision absolutely versus relatively, generate a reviewable SQL script with `--sql` instead of executing directly, and — critically — understand why a `downgrade()` function that actually works is not optional busywork, even though it's tempting to leave it as `pass` and move on.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Run `alembic upgrade head`, `alembic upgrade +1`, and `alembic upgrade <rev>` and explain exactly what each moves a database from and to.
- Run `alembic downgrade -1`, `alembic downgrade base`, and `alembic downgrade <rev>` and explain the same for reverse movement.
- Distinguish relative targeting (`+1`, `-1`) from absolute targeting (`head`, `base`, a specific revision ID), and choose correctly between them.
- Generate a plain SQL script with `--sql` instead of executing a migration directly, and explain why a DBA or change-review process would want this.
- Explain, with concrete examples, why neglected downgrade paths are a real production risk, not a theoretical one.
- Run a full upgrade-then-downgrade-then-upgrade cycle against ExpenseFlow's `users`/`expenses` schema and verify the database's state at each step with `alembic current`.

---

## Prerequisites for This Chapter

This chapter assumes:

- ExpenseFlow's two revisions from [Chapter 5](./05-revisions-and-version-history.md) — `create users table` and `create expenses table` — exist in your `alembic/versions/` directory, with working `upgrade()`/`downgrade()` bodies.
- A running local PostgreSQL instance, reachable via the connection string `env.py` resolves (Chapter 4).
- Comfort with the terms *revision*, *head*, *base*, and the migration graph as a linked list (Chapter 5, Section 4) — this chapter is entirely about moving along that linked list.

---

## 1. `alembic upgrade`: Moving Forward

`alembic upgrade` moves a database *forward* along the migration graph — from wherever it currently sits, toward some target further along the chain. The target can be specified three different ways.

### 1.1 `alembic upgrade head` — go to the end of the chain

```bash
alembic upgrade head
```

```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 1a2b3c4d5e6f, create users table
INFO  [alembic.runtime.migration] Running upgrade 1a2b3c4d5e6f -> 9f8e7d6c5b4a, create expenses table
```

Against a brand-new, empty ExpenseFlow database, this is the single command that takes you from nothing to the full current schema in one step. Alembic reads `alembic_version` (empty — no row yet), determines the database is at "base," and walks the entire chain forward, one revision's `upgrade()` at a time, until it reaches the current head. `head` is far and away the most common upgrade target you'll ever type — "bring this database fully up to date" is the default thing you want in local development, in CI, and (with real care, per Chapter 14 and Chapter 15) in production.

### 1.2 `alembic upgrade +1` — move forward one revision

```bash
alembic upgrade +1
```

This is a **relative** target: "from wherever the database currently is, apply exactly one more revision." If ExpenseFlow's database is currently stamped at `1a2b3c4d5e6f` (`users` only), `alembic upgrade +1` applies exactly `create expenses table` and stops — even though `head` is only one step away in this particular example, `+1` would stop one step short of `head` in a longer chain. This is useful for stepping through a long migration history one revision at a time — during debugging, or when manually verifying each step of a suspicious chain rather than trusting a single `upgrade head` to apply everything correctly in one shot.

### 1.3 `alembic upgrade <revision-id>` — go to a specific, absolute revision

```bash
alembic upgrade 1a2b3c4d5e6f
```

This is an **absolute** target: "move forward (or backward) until the database is stamped at exactly this revision, regardless of where it currently is." If the database is currently at base, this applies only `create users table` and stops — it does *not* also apply `create expenses table`, even though that revision exists further down the chain. You'll use this when you specifically need a database at some intermediate historical state — reproducing a bug that only existed before a later migration, for instance, or preparing a fixture that represents "the schema exactly as it looked before we added `categories`" (Chapter 7).

You can also abbreviate a revision ID to any prefix that's unambiguous — `alembic upgrade 1a2b` works exactly like `alembic upgrade 1a2b3c4d5e6f` as long as no other revision in the graph shares that same prefix.

---

## 2. `alembic downgrade`: Moving Backward

`alembic downgrade` is the mirror image, walking the graph *backward*, executing each revision's `downgrade()` function instead of `upgrade()`.

### 2.1 `alembic downgrade -1` — move back one revision

```bash
alembic downgrade -1
```

```
INFO  [alembic.runtime.migration] Running downgrade 9f8e7d6c5b4a -> 1a2b3c4d5e6f, create expenses table
```

From `head` (both revisions applied), this runs `create expenses table`'s `downgrade()` — dropping the `expenses` table's index and the table itself — and leaves the database stamped at `1a2b3c4d5e6f`, with only `users` remaining. Exactly like `+1`, `-1` is relative: "one step back from wherever we are right now."

### 2.2 `alembic downgrade base` — undo everything

```bash
alembic downgrade base
```

```
INFO  [alembic.runtime.migration] Running downgrade 9f8e7d6c5b4a -> 1a2b3c4d5e6f, create expenses table
INFO  [alembic.runtime.migration] Running downgrade 1a2b3c4d5e6f -> , create users table
```

This walks the entire chain backward, executing every `downgrade()` in reverse order, until the database is back at the empty base state — no tables, no `alembic_version` row at all. This is the command you run to prove your entire migration history is reversible, exactly as clean going down as it was going up — the exercise this chapter's Hands-On section has you do explicitly, and one CI pipelines can automate (Chapter 15) as a genuine correctness check, not just a formality.

### 2.3 `alembic downgrade <revision-id>` — go to a specific, absolute revision

```bash
alembic downgrade 1a2b3c4d5e6f
```

Absolute, exactly like its `upgrade` counterpart: move backward (or forward) until the database sits at precisely this revision. From `head`, this drops back to just the `users` table. From base, this would actually move *forward*, applying only `create users table` — `upgrade`/`downgrade` naming reflects intent, but Alembic will happily move in whichever direction actually gets the database to the specified target.

---

## 3. Relative vs. Absolute Targeting: A Reference Table

| Target syntax | Kind | Meaning | Depends on current position? |
|---|---|---|---|
| `head` | Absolute | The newest revision in the chain (or a specific head, e.g. `some_branch@head`, in multi-head graphs — Chapter 10) | No — always resolves to the same file(s) |
| `base` | Absolute | The empty state before any revision has run | No |
| `<revision-id>` (full or unambiguous prefix) | Absolute | That exact revision, whichever direction reaching it requires | No |
| `+N` | Relative | N revisions forward from wherever the database currently is | Yes — result depends entirely on current position |
| `-N` | Relative | N revisions backward from wherever the database currently is | Yes |
| `<rev>+N` / `<rev>-N` | Relative to a named point | N revisions forward/backward starting *from* a specific revision, not from current position | Partially — anchored to `<rev>`, not "current" |

The practical rule: **use absolute targets (`head`, `base`, a specific ID) for anything scripted, automated, or run against a database whose exact current state you haven't just checked** — CI pipelines, deployment jobs, onboarding scripts. `alembic upgrade head` is safe to run from *any* starting position, including one you're not entirely sure of, because "the end of the chain" means the same thing regardless of where you started. Reserve relative targets (`+1`, `-1`) for interactive, exploratory work where you've just run `alembic current` and know precisely where you're standing — typing `alembic downgrade -1` against a database whose current position you're not sure of is a good way to undo the wrong revision.

---

## 4. Watching the Full Cycle Against ExpenseFlow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant CLI as alembic CLI
    participant DB as PostgreSQL (ExpenseFlow)

    Dev->>CLI: alembic current
    CLI->>DB: SELECT version_num FROM alembic_version
    DB-->>CLI: (no rows — table doesn't exist yet)
    CLI-->>Dev: (nothing printed)

    Dev->>CLI: alembic upgrade head
    CLI->>DB: CREATE TABLE alembic_version (...)
    CLI->>DB: CREATE TABLE users (...); CREATE UNIQUE INDEX ix_users_email ...
    CLI->>DB: INSERT INTO alembic_version VALUES ('1a2b3c4d5e6f')
    CLI->>DB: CREATE TABLE expenses (...); CREATE INDEX ix_expenses_user_id ...
    CLI->>DB: UPDATE alembic_version SET version_num = '9f8e7d6c5b4a'
    CLI-->>Dev: done — DB now at head

    Dev->>CLI: alembic current
    CLI->>DB: SELECT version_num FROM alembic_version
    DB-->>CLI: 9f8e7d6c5b4a
    CLI-->>Dev: 9f8e7d6c5b4a (head)

    Dev->>CLI: alembic downgrade -1
    CLI->>DB: DROP INDEX ix_expenses_user_id; DROP TABLE expenses
    CLI->>DB: UPDATE alembic_version SET version_num = '1a2b3c4d5e6f'
    CLI-->>Dev: done — DB now at users-only

    Dev->>CLI: alembic upgrade head
    CLI->>DB: CREATE TABLE expenses (...); CREATE INDEX ix_expenses_user_id ...
    CLI->>DB: UPDATE alembic_version SET version_num = '9f8e7d6c5b4a'
    CLI-->>Dev: done — DB back at head
```

Two details in this trace matter beyond the obvious command sequence:

1. **The very first `alembic upgrade head` also creates `alembic_version` itself.** That table doesn't exist until Alembic's first `upgrade` run creates it — Chapter 9 covers this table in full, but it's worth noticing here that "applying the first migration" and "Alembic starting to track state at all" happen in the same step.
2. **Every single `upgrade()`/`downgrade()` call is immediately followed by an update to `alembic_version`**, and — because PostgreSQL DDL is transactional — both the schema change and the version-table update happen inside the same transaction (Chapter 3's `context.begin_transaction()`). If the `CREATE TABLE expenses` statement had failed partway through, the `alembic_version` update would never have committed either, so the database would still correctly report itself at `1a2b3c4d5e6f` — not silently drifted into an inconsistent "table exists but version says it doesn't" state.

---

## 5. `--sql`: Offline Mode for DBA/Change Review

Everything so far has executed directly against a live connection. Pass `--sql` instead, and Alembic switches into the offline mode Chapter 3 introduced conceptually: instead of connecting to a database and running each statement, it renders the SQL that *would* run, as plain text, to stdout — without ever opening a connection.

```bash
alembic upgrade head --sql
```

```sql
BEGIN;

CREATE TABLE alembic_version (
    version_num VARCHAR(32) NOT NULL,
    CONSTRAINT alembic_version_pkc PRIMARY KEY (version_num)
);

CREATE TABLE users (
    id SERIAL NOT NULL,
    email VARCHAR(255) NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    PRIMARY KEY (id)
);

CREATE UNIQUE INDEX ix_users_email ON users (email);

INSERT INTO alembic_version (version_num) VALUES ('1a2b3c4d5e6f');

CREATE TABLE expenses (
    id SERIAL NOT NULL,
    user_id INTEGER NOT NULL,
    amount_cents INTEGER NOT NULL,
    currency VARCHAR(3) NOT NULL,
    description VARCHAR(500),
    expense_date DATE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY(user_id) REFERENCES users (id) ON DELETE CASCADE
);

CREATE INDEX ix_expenses_user_id ON expenses (user_id);

UPDATE alembic_version SET version_num='9f8e7d6c5b4a' WHERE alembic_version.version_num = '1a2b3c4d5e6f';

COMMIT;
```

This is exactly the artifact a regulated environment's change-management process wants: a plain, human-readable SQL script that a DBA (or a peer engineer, or a change-advisory board) can read, understand, and formally approve *before* anyone runs it against a production database — without needing to trust that reading Python `op.*` calls correctly predicts what SQL will actually execute. Because offline mode never opens a real connection (`run_migrations_offline()`, Chapter 4, Section 4), you can generate this script from a laptop with no network access to production at all, save it to a file (`alembic upgrade head --sql > release_2026_07_01.sql`), send that file through whatever review/approval process your organization requires, and only then have someone with production access actually execute it.

`--sql` works with `downgrade` too, and with any target syntax from Section 3 — `alembic downgrade -1 --sql` produces the reviewable `DROP`/`ALTER` statements for rolling back exactly one revision, which is precisely the artifact you want ready *before* a risky production deploy, not improvised during an incident.

One caveat worth stating plainly: `--sql` mode does not evaluate any Python logic that depends on querying the live database — a migration that reads existing row values to decide what to backfill (Chapter 11's data migrations) cannot meaningfully render as static SQL text ahead of time, because the values it would operate on aren't known until it actually runs. `--sql` is a precise, reliable preview for schema-only DDL; it is not a full dry-run simulator for every migration, a nuance Chapter 11 returns to.

---

## 6. Why Downgrade Paths Matter

It's tempting, especially under deadline pressure, to write a real `upgrade()` and leave `downgrade()` as `pass` — "we're never going backward anyway." This chapter pushes back on that instinct directly, for three concrete reasons.

### 6.1 Reason one: rollback is your actual incident-response plan

When a deployed migration turns out to be wrong — a constraint that breaks a code path nobody tested, an index that wasn't actually needed and is now slowing down writes — the fastest safe response is often "revert the schema change and revert the app deploy together." If `downgrade()` is `pass`, that option doesn't exist: reverting the *application* code back to a version that expects the old schema, while the *database* is stuck on the new schema (because there's no working downgrade to run), can leave the reverted app broken in a new way. A working `downgrade()` keeps rollback available as a real incident-response tool, not just an aspiration.

### 6.2 Reason two: untested downgrades often don't actually work

Writing `downgrade()` and *never running it* is barely better than leaving it as `pass` — Chapter 17 covers, as a named common mistake, downgrade paths that were written once, never executed, and quietly broken (a renamed column in a later revision, a constraint added elsewhere that the downgrade doesn't know to drop first) by the time anyone actually needs them. This chapter's Hands-On Exercise has you run a full `upgrade head` → `downgrade base` → `upgrade head` cycle specifically because *running* the downgrade, not just writing it, is what proves it works. Chapter 15 turns this into an automated CI check for exactly this reason.

### 6.3 Reason three: downgrade is how you build confidence in a migration before it ships

A revision whose `upgrade()` and `downgrade()` cleanly cancel each other out — running one then the other returns the schema to a byte-for-byte identical state — is strong evidence the migration was written with real care about what it's actually doing to the schema, not just "make the error go away." This is a code-quality signal for migration review, not merely an operational safety net.

None of this means every downgrade needs to be data-preserving — dropping a table's `downgrade()` reasonably drops the table (Section 2.1's example does exactly that), and data written after the `upgrade()` ran is legitimately gone once you downgrade. The point is narrower and non-negotiable: **`downgrade()` should correctly reverse the schema change**, even when it can't magically un-lose data that only existed because of the schema it's removing.

---

## 7. Command Reference: Every Command From Chapters 5–6, Side by Side

With both chapters' commands now in hand, this is the reference table worth bookmarking — every read-only inspection command from Chapter 5 alongside every state-changing command from this chapter, in one place.

| Command | Changes the database? | Target syntax accepted | Typical use |
|---|---|---|---|
| `alembic current` | No | — | "Where is this specific database right now?" |
| `alembic history` | No | — | "What revisions exist, and in what order?" |
| `alembic heads` | No | — | "Is this graph a clean single chain, or are there multiple heads?" |
| `alembic show <rev>` | No | full/prefix revision ID, `head`, `current` | "What exactly does this one revision do?" |
| `alembic upgrade <target>` | Yes | `head`, `+N`, `<rev>`, `<rev>+N` | Move forward |
| `alembic downgrade <target>` | Yes | `base`, `-N`, `<rev>`, `<rev>-N` | Move backward |
| `alembic upgrade <target> --sql` | No (offline mode) | same as `upgrade` | Generate a reviewable forward SQL script |
| `alembic downgrade <target> --sql` | No (offline mode) | same as `downgrade` | Generate a reviewable rollback SQL script |

The pattern worth internalizing: everything from Chapter 5 is safe to run against any database, at any time, with zero risk, because none of it changes anything — which is exactly why "when in doubt, run `alembic current` first" (Section 3) is such cheap, reliable advice. Everything from this chapter's Sections 1–2 changes the database for real, unless you add `--sql`, which converts even those into inspection-only commands that happen to describe a state change rather than perform one.

### 7.1 A worked example: targeting a specific revision from a specific starting point

The `<rev>+N` / `<rev>-N` forms from Section 3's table deserve one concrete example, since they're the least intuitive entry in it. Suppose a future ExpenseFlow migration graph has five revisions in sequence, and you want "two revisions after the one that added the `categories` table" — regardless of where the database currently sits:

```bash
alembic upgrade categories_table_rev+2
```

This anchors to the named revision first, then moves two steps forward *from that anchor* — not from wherever the database happens to be right now. It's a small feature, reached for rarely, but it's the right tool exactly once: when you need to describe a target relative to a specific point in history rather than relative to "now" or relative to either end of the chain.

---

## Real-World Scenario

ExpenseFlow ships a new revision that adds a `NOT NULL` constraint to `expenses.currency` (foreshadowing Chapter 11's data-migration chapter, which covers the full backfill-then-constrain pattern properly). During deployment to staging, the migration fails partway — a handful of pre-existing rows in the staging database have `NULL` currency values that nobody backfilled first, and the `ALTER COLUMN ... SET NOT NULL` statement rejects them.

Because PostgreSQL DDL runs inside a transaction (Section 4), the failed migration doesn't leave staging in a half-migrated state — the whole revision's `upgrade()` rolled back automatically, and `alembic current` still correctly reports the previous revision. But the on-call engineer, following the deploy runbook, still needs to confirm the *previous* revision's schema is fully intact and the app can run against it safely, before deciding whether to fix the data and retry or roll the whole release back.

They run:

```bash
alembic current
# 9f8e7d6c5b4a (head)   -- confirms the failed revision never actually got applied

alembic upgrade head --sql > /tmp/pending_migration.sql
# review the exact SQL that would run, confirms the failing statement, no surprises
```

Because `alembic current` and `--sql` are both purely inspective — neither one touches the schema — the engineer gets a precise, risk-free read on exactly where staging stands and exactly what the next attempt would do, before making any further changes. They decide against a `downgrade`, since nothing was actually applied to reverse; instead, they add a data-backfill step ahead of the constraint (the safe pattern Chapter 11 covers properly), and the retried migration succeeds cleanly. The incident report notes, approvingly, that the transactional nature of the failed `upgrade()` combined with `alembic current`'s precise state reporting meant nobody had to guess whether staging was in some ambiguous in-between state — it demonstrably wasn't.

---

## Best Practices

- **Default to absolute targets (`head`, `base`, a specific revision ID) in any scripted or automated context** — CI, deploy jobs, onboarding scripts — since they don't depend on knowing the database's current position ahead of time.
- **Run `alembic current` before any interactive `+1`/`-1` command** so relative targeting moves the database exactly where you intend.
- **Generate and review `--sql` output for any migration touching a production-scale table**, before running it live — this is cheap insurance against a DDL statement doing something other than what you assumed from reading the Python.
- **Write and actually run `downgrade()` for every revision**, not just `upgrade()` — Section 6 covers why an unexecuted downgrade is not meaningfully safer than a missing one.
- **Practice full `upgrade head` → `downgrade base` → `upgrade head` cycles locally**, especially for any migration you're not fully confident in, before it ever reaches a shared environment.
- **Treat `--sql` output as an artifact, not just terminal noise** — save it (`> filename.sql`) when it's meant to support a review or approval process, rather than reading it once and discarding it.

---

## Common Mistakes

- **Running `alembic upgrade +1` or `alembic downgrade -1` without first checking `alembic current`**, moving the database to an unintended revision because "current position" wasn't what was assumed.
- **Leaving `downgrade()` as `pass`** and discovering, during an actual incident, that rollback isn't actually possible — see Section 6 and the Real-World Scenario.
- **Writing a `downgrade()` once and never running it**, only to find it's silently broken (references a column a later revision renamed, drops a constraint in the wrong order) the one time it's actually needed.
- **Assuming `--sql` output is a complete dry-run for every migration**, including ones with data-dependent Python logic (Chapter 11) that can't be meaningfully rendered as static SQL ahead of a real run.
- **Confusing relative and absolute semantics** — expecting `alembic upgrade +1` to always land on a specific, memorized revision, rather than understanding it depends entirely on wherever the database currently sits.
- **Running `alembic downgrade base` against a database with real user data**, without recognizing that every `downgrade()` along the way that drops a table or column genuinely discards whatever data was in it — appropriate in development, dangerous without a very deliberate reason anywhere near production data.

---

## Summary

- `alembic upgrade` moves a database forward through the migration graph, targeted via `head` (absolute, end of chain), `+N` (relative, N steps forward), or a specific revision ID (absolute) (Section 1).
- `alembic downgrade` is the mirror operation, targeted via `base` (absolute, empty state), `-N` (relative), or a specific revision ID (Section 2).
- Absolute targets don't depend on the database's current position; relative targets do — prefer absolute targets for anything scripted or automated (Section 3).
- Every `upgrade()`/`downgrade()` runs inside a transaction alongside its corresponding `alembic_version` update, so a failed migration never leaves the database in an inconsistent, half-applied state (Section 4).
- `--sql` renders the exact SQL a migration would run, as plain text, without ever opening a live connection — the artifact a DBA or change-review process needs to approve a production change ahead of time (Section 5).
- A working, actually-tested `downgrade()` is real incident-response infrastructure, not optional boilerplate — an untested or missing downgrade quietly removes rollback as an option exactly when you'd need it most (Section 6).

---

## Knowledge Check

1. What's the difference in behavior between `alembic upgrade head` and `alembic upgrade +1`, run against a database that's currently three revisions behind head?
2. Why is `alembic upgrade head` generally safer to run in an automated deploy script than `alembic upgrade +1`?
3. What does `--sql` actually do differently from a normal `alembic upgrade head`, and what does it deliberately not do?
4. Why does a failed `upgrade()` (say, a constraint violation partway through) not leave ExpenseFlow's database in a half-migrated state?
5. Give a concrete reason, beyond "just in case," why a team should actually run `alembic downgrade base` against a migration before considering it fully tested.
6. If `alembic downgrade <rev>` is called with a revision ID that is actually *ahead* of the database's current position, what happens?
7. Why can't `--sql` mode meaningfully preview a migration whose logic depends on reading existing row values from the live database?

---

## Hands-On Exercise

**Goal:** Run ExpenseFlow's two Chapter 5 revisions through a full upgrade/downgrade cycle, and generate a reviewable SQL script.

1. **Confirm your starting state** with `alembic current` against your local database — it should print nothing (empty database, no migrations applied).

2. **Run `alembic upgrade head`** and watch the log output. Confirm both `create users table` and `create expenses table` run, in that order.

3. **Run `alembic current`** and confirm it now reports the `expenses` revision's ID with `(head)`.

4. **Inspect the actual tables** with `psql` (`\dt`, `\d users`, `\d expenses`) and confirm the schema matches what the migration files declared, including the foreign key and both indexes.

5. **Run `alembic downgrade -1`** and confirm the `expenses` table is dropped, and `alembic current` now reports only the `users` revision.

6. **Run `alembic downgrade base`** and confirm `users` is dropped too, and `alembic current` prints nothing again — back to the true starting state.

7. **Run `alembic upgrade head`** one more time to confirm the full chain re-applies cleanly from empty, completing a full round trip.

8. **Generate a reviewable SQL script** with `alembic upgrade base:head --sql > /tmp/expenseflow_full_schema.sql` (or, from an already-upgraded database, `alembic downgrade base --sql > /tmp/expenseflow_full_rollback.sql`), and read the generated file. Confirm it matches Section 5's expected shape — `BEGIN`, both `CREATE TABLE` statements, both `alembic_version` writes, `COMMIT`.

9. **Time-box a deliberate failure test**: temporarily break `create expenses table`'s `upgrade()` (e.g., reference a nonexistent column in a constraint), run `alembic upgrade head` from a users-only state, confirm it fails and rolls back cleanly, then run `alembic current` to confirm the database is still correctly reported at the `users` revision — not stuck in an ambiguous state. Revert your deliberate breakage afterward.

You've now exercised every command this chapter covers, against a real database, in both directions. [Chapter 7](./07-autogenerate-migrations.md) introduces `--autogenerate`, letting Alembic generate the `upgrade()`/`downgrade()` bodies you've been writing by hand, starting with ExpenseFlow's next schema change: a `categories` table.

---

## Further Reading

- [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html) — the official section on running upgrades and downgrades, including relative/absolute targeting syntax.
- [Alembic Official Documentation](https://alembic.sqlalchemy.org/en/latest/) — the `alembic.command.upgrade`/`downgrade` API reference, and the `--sql` offline-mode flag.
- [Alembic Cookbook](https://alembic.sqlalchemy.org/en/latest/cookbook.html) — recipes for generating and working with offline SQL scripts.
- [PostgreSQL ALTER TABLE docs](https://www.postgresql.org/docs/current/sql-altertable.html) — background on the transactional DDL behavior underpinning Section 4's failed-migration example.
- [SQLAlchemy 2.0 ORM documentation](https://docs.sqlalchemy.org/en/20/orm/) — for the `Mapped[...]`/`mapped_column(...)` model definitions ExpenseFlow's migrations continue to track.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./05-revisions-and-version-history.md">← Previous: Revisions & Version History</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./07-autogenerate-migrations.md">Next: Autogenerate Migrations →</a>
</div>
