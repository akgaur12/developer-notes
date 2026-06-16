# Chapter 04 — DML: INSERT, SELECT, UPDATE, DELETE & Shaping Results

## Where you are
You have well-typed, constrained tables. Now you manipulate the data inside them. DML = Data Manipulation Language. By the end you can do all of **CRUD** (Create, Read, Update, Delete) and shape query results precisely. The `SELECT` skills here are the foundation for *everything* in the intermediate tier.

---

## 1. INSERT — create rows
```sql
INSERT INTO tasks (title, priority) VALUES ('Write report', 1);

-- multiple rows in one statement (faster than many single inserts):
INSERT INTO tasks (title, priority) VALUES
    ('Email client', 2),
    ('Review PR', 3),
    ('Deploy', 1);

-- get back values the database generated (id, defaults):
INSERT INTO tasks (title) VALUES ('New task')
RETURNING id, created_at;
```
`RETURNING` is a Postgres gem — it hands back generated ids/defaults without a second query. It works on `UPDATE` and `DELETE` too.

**Upsert** — insert, or update if it already exists (`ON CONFLICT`):
```sql
INSERT INTO users (email, name) VALUES ('a@x.com', 'Asha')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;   -- EXCLUDED = the row you tried to insert
-- or ignore the conflict entirely:
INSERT INTO users (email) VALUES ('a@x.com') ON CONFLICT (email) DO NOTHING;
```

## 2. SELECT — read rows
The full logical shape (we'll meet the later clauses in Chapters 06–08):
```sql
SELECT   columns / expressions
FROM     table
WHERE    row filter
GROUP BY grouping
HAVING   group filter
ORDER BY sort
LIMIT    cap
OFFSET   skip;
```

> **Critical "why": SQL clauses do NOT execute in the order you write them.** The *logical* evaluation order is roughly:
> ```
> FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET
> ```
> This single fact explains many beginner confusions — e.g. why you can't use a `SELECT` column alias in `WHERE` (the alias doesn't exist yet when `WHERE` runs), but you *can* in `ORDER BY` (which runs after `SELECT`).

Basics:
```sql
SELECT * FROM tasks;                          -- all columns (avoid in production code)
SELECT title, priority FROM tasks;            -- specific columns
SELECT title AS task_name FROM tasks;         -- alias
SELECT title, priority * 10 AS weight FROM tasks;  -- computed expression
```

## 3. WHERE — filtering rows
```sql
WHERE priority = 1
WHERE priority IN (1, 2)
WHERE priority BETWEEN 1 AND 3          -- inclusive
WHERE title LIKE 'Deploy%'              -- pattern; % = any chars, _ = one char
WHERE title ILIKE 'deploy%'             -- case-insensitive (Postgres extension)
WHERE due_at IS NULL                    -- NULL handling (Chapter 02!)
WHERE priority = 1 AND is_done = false
WHERE priority = 1 OR priority = 3
WHERE NOT is_done
```
**Pattern-matching tiers:** `LIKE`/`ILIKE` for simple wildcards; `SIMILAR TO` for SQL-regex; `~` / `~*` for POSIX regular expressions. Leading-wildcard patterns (`'%term'`) can't use an ordinary index — relevant in Chapters 13 and 16.

## 4. ORDER BY, LIMIT, OFFSET — shaping output
```sql
SELECT * FROM tasks ORDER BY priority ASC, created_at DESC;
SELECT * FROM tasks ORDER BY priority NULLS LAST;   -- control where NULLs land
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 10;        -- newest 10
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 10 OFFSET 20;  -- "page 3"
```
> **Pitfall: `LIMIT` without `ORDER BY` returns rows in an *undefined* order.** It may look stable, then change after an update or version upgrade. Always pair `LIMIT` with a deterministic `ORDER BY`.
>
> **Pitfall: `OFFSET` pagination gets slow** on deep pages (the DB still scans and discards all skipped rows) and can skip/duplicate rows when data changes between page loads. For large datasets use **keyset (cursor) pagination** instead: `WHERE created_at < :last_seen ORDER BY created_at DESC LIMIT 10`.

## 5. DISTINCT and DISTINCT ON
```sql
SELECT DISTINCT priority FROM tasks;                 -- unique values
SELECT DISTINCT ON (user_id) user_id, created_at     -- Postgres extension:
FROM events ORDER BY user_id, created_at DESC;        -- one row per user (the newest)
```
`DISTINCT ON` is a uniquely Postgres way to grab "the first row per group" — pair it with a matching `ORDER BY`.

## 6. UPDATE — modify rows
```sql
UPDATE tasks SET is_done = true WHERE id = 5;
UPDATE tasks SET priority = priority + 1 WHERE due_at < now();   -- expression
UPDATE tasks SET is_done = true WHERE id = 5 RETURNING *;        -- see the result
```
> **The most dangerous mistake in SQL:** an `UPDATE` (or `DELETE`) with **no `WHERE`** changes *every row*. There is no undo outside a transaction. Habits that save you: wrap risky changes in `BEGIN; ... ROLLBACK/COMMIT;` (Chapter 09), and run a `SELECT` with the same `WHERE` *first* to see exactly which rows you're about to hit.

## 7. DELETE
```sql
DELETE FROM tasks WHERE is_done = true;
DELETE FROM tasks WHERE id = 5 RETURNING *;
```
`DELETE` is row-by-row and logged in detail (and respects foreign keys). To empty a whole table fast, prefer `TRUNCATE` (Chapter 03). **Soft delete** is a common alternative — instead of removing the row, set a `deleted_at TIMESTAMPTZ` and filter `WHERE deleted_at IS NULL` in queries; this preserves history and is reversible, at the cost of carrying dead rows.

## 8. NULL behavior in expressions (recap with teeth)
Arithmetic and concatenation with `NULL` yield `NULL`:
```sql
SELECT 5 + NULL;                  -- NULL
SELECT 'Hi ' || NULL;             -- NULL
SELECT COALESCE(nickname, name);  -- first non-NULL value → a safe default
SELECT NULLIF(a, b);              -- NULL if a = b, else a
```
`COALESCE` is your everyday tool for "use X, or fall back to Y if X is NULL."

---

## Summary
- **INSERT** creates rows; do multi-row inserts for speed, use `RETURNING` to get generated values, and `ON CONFLICT` for upserts.
- **SELECT** clauses are written in one order but **evaluated** in another (`FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`) — which explains alias-scoping rules.
- **WHERE** filters with comparisons, `IN`, `BETWEEN`, `LIKE`/`ILIKE`, regex, and `IS [NOT] NULL`.
- **ORDER BY** must accompany `LIMIT` for deterministic results; prefer **keyset pagination** over deep `OFFSET`.
- **UPDATE/DELETE without `WHERE` hit every row** — guard with transactions and a prior `SELECT`.
- `COALESCE`/`NULLIF` tame NULLs in expressions.

## Test your understanding
1. Why can you reference a column alias in `ORDER BY` but not in `WHERE`?
2. Write an upsert that inserts a user by email, updating the name if the email already exists.
3. Why is `LIMIT 10` without `ORDER BY` a latent bug? Why is deep `OFFSET` pagination problematic?
4. What's the difference between `DELETE FROM t;`, `TRUNCATE t;`, and a soft delete? When would you choose each?
5. `SELECT 'Total: ' || total_label FROM ...` returns NULL for some rows even though `total_label` looks set. What's likely happening and how do you fix it?

## Hands-on exercise
Using your two related tables:
1. Insert several parent and child rows; use `RETURNING` to capture generated ids.
2. Write an `ON CONFLICT` upsert against a unique column.
3. Write a `SELECT` that filters with `WHERE`, sorts with `ORDER BY ... NULLS LAST`, and pages with `LIMIT/OFFSET`.
4. Run a `SELECT` with a specific `WHERE`, then run the matching `UPDATE ... RETURNING *` and confirm only those rows changed.
5. Implement soft delete: add `deleted_at`, "delete" a row by setting it, and write the query that hides soft-deleted rows.
