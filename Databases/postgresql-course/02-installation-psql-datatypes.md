# Chapter 02 — Installation, `psql`, and Data Types

## Where you are
You understand that a table is a set of typed, rule-bound rows and that SQL is declarative. Now we make it practical: run Postgres, drive the `psql` client, and — most importantly — choose data types correctly. Type decisions made on day one quietly cause bugs and slowdowns for years, so this is the highest-leverage beginner skill.

---

## 1. Installing PostgreSQL

**Path A — Native on Ubuntu (learning on your own machine):**
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh   # adds the official PGDG repo
sudo apt install -y postgresql-18
sudo -u postgres psql      # the install creates a Linux user `postgres`
```

**Path B — Docker (fast, disposable, mirrors production) — recommended for this course:**
```bash
docker run --name pg \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:18

docker exec -it pg psql -U postgres
```
The `-v pgdata:...` volume is essential — without it your data vanishes when the container is removed. (The most common "where did my data go?" beginner mistake.)

**Path C — Managed (production reality):** RDS, Cloud SQL, Azure Database, Supabase, Neon. You skip install and get a connection string. We learn raw Postgres first so the managed versions hold no mystery.

Facts to internalize:
- Default port **5432**.
- Bootstrap superuser **`postgres`**.
- A cluster lives in one **data directory** (`PGDATA`, e.g. `/var/lib/postgresql/data`). That directory *is* your cluster.

### Authentication: `pg_hba.conf` (trips everyone up once)
Postgres decides *how* you may log in via **`pg_hba.conf`** (Host-Based Authentication). Common methods:
```
peer            trust the OS username (local only; why `sudo -u postgres psql` works)
scram-sha-256   password-based (modern secure default; replaced md5)
trust           no auth at all — NEVER outside a throwaway sandbox
```
Postgres uses the **first matching line**, top to bottom. When a connection is mysteriously rejected, look here first.

## 2. `psql` — your cockpit

`psql` is the official CLI. GUIs (pgAdmin, DBeaver, DataGrip) are fine, but professionals live in `psql` — it's fast, scriptable, and always on the server.

Connecting (host, port, user, database):
```bash
psql -h localhost -p 5432 -U postgres -d postgres
psql "postgresql://postgres:secret@localhost:5432/postgres"   # single URI
```

**Two kinds of input:**
```
SQL statements  → end with a semicolon ;   (sent to the server)
Meta-commands   → start with a backslash \  (handled by psql itself, no semicolon)
```
`SELECT now();` is SQL. `\dt` is a psql shortcut. The `\` commands never reach the server.

**Daily meta-commands:**
```
\l            list databases
\c dbname     connect to / switch database
\dt           list tables
\d name       describe a table (columns, types, indexes, constraints) ← used constantly
\du           list roles/users
\dn           list schemas
\x            toggle expanded (vertical) display — lifesaver for wide rows
\timing       show how long each query took
\i file.sql   run SQL from a file
\e            open $EDITOR to compose a query
\h CREATE TABLE   help on a SQL command's syntax
\?            help on meta-commands
\q            quit
```
The prompt shows context: `postgres=#` → superuser in db `postgres`; `shop=>` → non-superuser in db `shop`.

## 3. Data types — the heart of this chapter

> **A type is a promise and a constraint.** The right type makes data compact, auto-validated, correctly sortable/comparable, and index-efficient. The lazy choice (everything as `TEXT`) throws all of that away.

### Numbers
```
smallint     2 bytes   ±32K               rarely needed
integer/int  4 bytes   ±2.1 billion       default whole-number choice
bigint       8 bytes   ±9.2 quintillion   ids/counts that may grow huge
numeric(p,s) exact, arbitrary precision    MONEY, anything requiring exactness
real/double  4/8 bytes floating point      scientific, approximate only
```
**Never use floating point for money** — floats can't represent `0.1` exactly:
```sql
SELECT 0.1::double precision + 0.2;  -- 0.30000000000000004  (wrong)
SELECT 0.1::numeric + 0.2;           -- 0.3                   (exact)
```
Use `numeric(12,2)` for currency (or store integer *cents* — a valid pro pattern).

### Auto-incrementing IDs — the modern way
```sql
id SERIAL PRIMARY KEY                                -- older style; still everywhere
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY   -- modern, SQL-standard, preferred
```
Prefer `bigint` identity. Exhausting a 4-byte `integer` id at 2.1 billion rows has caused real outages. `bigint` from the start is cheap insurance. Alternatively **UUID** keys (`gen_random_uuid()`) are globally unique and don't leak row counts — great for distributed systems, at the cost of size and index ordering.

### Text
```
text         variable, unlimited      USE THIS by default
varchar(n)   variable, max n chars     only when n is a real domain rule (e.g. 2-char code)
char(n)      fixed, space-padded       almost never; avoid
```
**Postgres-specific truth:** `varchar(n)` has **no performance advantage** over `text` — they're stored identically. Default to `text`.

### Booleans
```sql
is_active BOOLEAN   -- true/false (accepts 't'/'f', 'yes'/'no', 1/0 on input)
```
Prefer a real `boolean` over a `text` `'Y'`/`'N'`.

### Dates and times — your most important type decision
```
date            calendar date, no time
time            time of day, no date
timestamp       date+time, NO timezone   ← dangerous default
timestamptz     date+time WITH timezone  ← almost always what you want
interval        a duration ('3 days 4 hours')
```
**Cardinal rule: use `timestamptz`, not `timestamp`.** Despite the name, `timestamptz` does *not* store a timezone — it stores an absolute instant (internally UTC) and converts to/from the client's timezone at the edges. Plain `timestamp` stores an unanchored wall-clock reading, so the same value means different real moments depending on where it's read — a notorious corruption source.
```sql
SET timezone = 'Asia/Kolkata';
SELECT now();                     -- local time…
SELECT now() AT TIME ZONE 'UTC';  -- …but anchored to a real UTC instant
SELECT now() + interval '7 days'; -- interval is lovely for date math
```
Store moments as `timestamptz`; convert for display at the edges.

### Semi-structured & modern
```
jsonb        binary JSON — indexable, queryable      use this (not `json`)
json         raw text, preserves formatting          rarely what you want
uuid         128-bit identifier
arrays       integer[], text[]  (any type)
enum         a fixed set of allowed string values
inet/cidr    IP addresses/networks (validated!)
```
Default to **`jsonb`**. Postgres lets you mix rigid relational columns with flexible JSONB in one table — often removing any need for a separate document database (Chapter 15).

`ENUM` example:
```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');
-- a column of type order_status can ONLY hold those four values
```

## 4. NULL and three-valued logic (a guaranteed gotcha)

`NULL` ≠ zero ≠ empty string. It means **"unknown / no value,"** and that breaks ordinary logic:
```sql
SELECT NULL = NULL;  -- NULL (not true!) "is unknown equal to unknown? unknown."
SELECT 5 = NULL;     -- NULL
SELECT 5 <> NULL;    -- NULL
```
Any comparison *with* `NULL` yields `NULL` (treated as not-true). So:
```sql
WHERE email IS NULL        -- correct
WHERE email IS NOT NULL    -- correct
WHERE email = NULL         -- WRONG — silently matches nothing
```
```
Three-valued logic:  TRUE | FALSE | NULL(unknown)
A WHERE clause keeps a row ONLY when its condition is TRUE.
NULL is not TRUE → the row is dropped → `= NULL` finds nothing.
```
This is *why* `NOT NULL` matters: it forbids "unknown" where your model requires a real value.

## 5. Bringing it together: your paper schema becomes real
```sql
CREATE TABLE tasks (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title       TEXT NOT NULL,                       -- required, unbounded
    details     TEXT,                                -- optional → NULL allowed
    priority    SMALLINT NOT NULL DEFAULT 3,         -- small range
    is_done     BOOLEAN NOT NULL DEFAULT false,      -- never "unknown"
    due_at      TIMESTAMPTZ,                         -- absolute moment, optional
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),  -- DB stamps it
    tags        TEXT[],                              -- array of labels
    metadata    JSONB                                -- flexible extras
);
```
Every line is a deliberate promise about what the data can be.

---

## Summary
- Install native, via Docker, or managed; a running instance is a **cluster** in one **data directory**, port **5432**, superuser `postgres`. **`pg_hba.conf`** governs *how* you authenticate (first matching line wins).
- **`psql`**: SQL ends in `;`; meta-commands start with `\`. Learn `\l \c \dt \d \x \timing \q`.
- **Type choice is foundational:** `integer`/`bigint` for whole numbers, `numeric` for money (never float), `text` for strings (no penalty vs `varchar`), `boolean` for true/false, **`timestamptz` for moments**, `jsonb` for JSON, `bigint GENERATED AS IDENTITY` for keys.
- **`NULL` means "unknown"** → three-valued logic → use `IS NULL` / `IS NOT NULL`, never `= NULL`.

## Test your understanding
1. Why is `varchar(50)` no faster than `text` in Postgres? When *would* you use `varchar(n)`?
2. Which type for prices, and why is `double precision` a bug waiting to happen?
3. `timestamp` vs `timestamptz` — practical difference, and which is the default and why?
4. `... WHERE deleted_at = NULL` returns zero rows though many lack a `deleted_at`. What's wrong and what's the fix?
5. SQL statement vs `psql` meta-command — how do you tell them apart?

## Hands-on exercise
With a live Postgres open, in `psql`:
1. `CREATE DATABASE` for your domain and `\c` into it.
2. Translate your Chapter 01 paper schema into one real `CREATE TABLE` — include at least a `bigint identity` key, a `NOT NULL text`, a `timestamptz DEFAULT now()`, and a `boolean`.
3. `\d yourtable` and verify types/defaults.
4. Deliberately break it: insert a row missing a `NOT NULL` column (watch it reject); run `... WHERE col = NULL` (watch it return nothing).
5. `\timing on`, run `SELECT now();`.
