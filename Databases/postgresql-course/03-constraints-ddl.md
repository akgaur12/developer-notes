# Chapter 03 — Constraints & DDL: Schemas That Guarantee Integrity

## Where you are
You can pick types. Now you'll attach **rules** to them so the database itself guarantees your data is valid. This is Postgres's superpower: integrity enforced at the lowest level, where no buggy application code can bypass it. DDL = Data Definition Language (the `CREATE`/`ALTER`/`DROP` statements that define structure).

> **The "why":** Application code has bugs and there's usually more than one app touching a database over its life. A rule enforced *in the database* can never be skipped. Constraints are not bureaucracy — they are the cheapest bug prevention you will ever buy.

---

## 1. The constraint family

```
NOT NULL      column must have a value (forbids "unknown")
DEFAULT       value used when none is supplied (not strictly a constraint, but related)
UNIQUE        no two rows may share this value (NULLs are exempt — see below)
PRIMARY KEY   = UNIQUE + NOT NULL; the row's canonical identity (one per table)
CHECK         an arbitrary boolean rule the row must satisfy
FOREIGN KEY   this value must exist in another table's key (referential integrity)
EXCLUSION     advanced: no two rows may "conflict" by a custom operator (e.g. overlapping ranges)
```

A worked schema showing each in place:
```sql
CREATE TABLE departments (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL UNIQUE
);

CREATE TABLE employees (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email      TEXT NOT NULL UNIQUE,
    name       TEXT NOT NULL,
    salary     NUMERIC(12,2) NOT NULL CHECK (salary > 0),
    dept_id    BIGINT REFERENCES departments(id),
    hired_on   DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 2. PRIMARY KEY — the row's identity

Every table should have one. It uniquely identifies a row and is what other tables point at. Properties:
- Automatically `UNIQUE` and `NOT NULL`.
- Automatically backed by a **unique index** (so lookups by primary key are fast — a fact that matters from Chapter 11 onward).
- **Surrogate vs natural keys:** a *surrogate* key is a meaningless generated id (`bigint identity`, `uuid`); a *natural* key is real-world data that's unique (e.g. an ISBN). Prefer surrogate keys for stability — natural keys have an annoying habit of changing or turning out not to be unique after all.

A **composite primary key** spans multiple columns (common in join tables):
```sql
CREATE TABLE enrollment (
    student_id BIGINT REFERENCES students(id),
    course_id  BIGINT REFERENCES courses(id),
    PRIMARY KEY (student_id, course_id)   -- a student can't enroll in the same course twice
);
```

## 3. UNIQUE — and the NULL subtlety
`UNIQUE` forbids duplicate values. **But NULLs are treated as distinct from each other**, so a `UNIQUE` column allows many NULL rows by default:
```sql
-- two rows with email = NULL are both allowed under plain UNIQUE
```
Since PostgreSQL 15 you can change this with `UNIQUE NULLS NOT DISTINCT` if you want at most one NULL. Usually you combine `UNIQUE` with `NOT NULL` to sidestep the question entirely.

## 4. CHECK — arbitrary business rules
A `CHECK` is any boolean expression over the row's columns:
```sql
CREATE TABLE products (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    price  NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    sale   NUMERIC(10,2),
    CHECK (sale IS NULL OR sale < price)   -- a table-level check spanning two columns
);
```
Column-level checks reference one column; table-level checks (written separately) can reference several. Use them for invariants that must *always* hold — they're validated on every insert and update.

## 5. FOREIGN KEY — referential integrity
A foreign key says "this column's value must exist as a key in another table." It prevents orphans (an `employee.dept_id` pointing at a department that doesn't exist).
```sql
dept_id BIGINT REFERENCES departments(id)
```
The crucial design decision is **what happens when the referenced row is deleted or updated** — the *referential action*:
```
ON DELETE RESTRICT  (default: NO ACTION)  block the delete if children exist
ON DELETE CASCADE                          delete the children too
ON DELETE SET NULL                         set the child's FK to NULL
ON DELETE SET DEFAULT                      set the child's FK to its default
```
```sql
CREATE TABLE order_items (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE
    -- deleting an order removes its items automatically
);
```
Choose deliberately: `CASCADE` is convenient but dangerous (one delete can wipe out thousands of rows); `RESTRICT` is safe but forces you to clean up children first. There is no universally right answer — it depends on whether the child is *part of* the parent (cascade) or *merely references* it (restrict/set null).

> **Pitfall:** Foreign key columns are **not automatically indexed** (only the *referenced* side is, via its primary key). Unindexed FK columns make joins and `ON DELETE CASCADE` slow, and can cause lock contention. Index your FK columns yourself (Chapter 11).

## 6. DEFAULT and generated columns
```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
status     TEXT NOT NULL DEFAULT 'pending'
```
A **generated column** is computed from others and stored automatically:
```sql
CREATE TABLE rectangles (
    width   NUMERIC NOT NULL,
    height  NUMERIC NOT NULL,
    area    NUMERIC GENERATED ALWAYS AS (width * height) STORED
);
```
You never write to `area`; Postgres keeps it correct. Great for derived values you query often.

## 7. Naming constraints (do this in production)
Postgres auto-names constraints (e.g. `employees_salary_check`), but explicit names make errors readable and migrations manageable:
```sql
CONSTRAINT salary_positive CHECK (salary > 0)
CONSTRAINT fk_dept FOREIGN KEY (dept_id) REFERENCES departments(id)
```

## 8. Evolving a schema with ALTER TABLE
Schemas change. `ALTER TABLE` modifies them in place:
```sql
ALTER TABLE employees ADD COLUMN phone TEXT;
ALTER TABLE employees ALTER COLUMN phone SET NOT NULL;
ALTER TABLE employees ADD CONSTRAINT uq_phone UNIQUE (phone);
ALTER TABLE employees DROP COLUMN phone;
ALTER TABLE employees RENAME COLUMN name TO full_name;
```
**Production caution:** some ALTERs rewrite the whole table or take strong locks that block reads/writes. Adding a column with a non-volatile `DEFAULT` is fast in modern Postgres; adding a `NOT NULL` to a huge existing table, or some type changes, can lock the table for a long time. Always test migrations on production-sized data, and learn the safe patterns (e.g. add nullable column → backfill in batches → add constraint `NOT VALID` then `VALIDATE`).

## 9. DROP and TRUNCATE
```sql
DROP TABLE employees;            -- removes the table entirely
DROP TABLE IF EXISTS employees;  -- no error if absent
TRUNCATE employees;              -- deletes ALL rows fast (no per-row WAL like DELETE), resets nothing else
TRUNCATE employees RESTART IDENTITY CASCADE;  -- also reset identity counter & cascade to FK children
```

---

## Summary
- Constraints enforce correctness **in the database**, where no app bug can bypass them — the cheapest bug prevention available.
- **PRIMARY KEY** (= UNIQUE + NOT NULL, auto-indexed) gives each row identity; prefer surrogate keys. Composite keys span columns.
- **UNIQUE** treats NULLs as distinct (allows many NULLs) unless `NULLS NOT DISTINCT`.
- **CHECK** enforces arbitrary invariants; **FOREIGN KEY** enforces referential integrity — and you must choose its `ON DELETE` action deliberately.
- FK columns are **not auto-indexed** — index them yourself.
- **DEFAULT** and **generated columns** let the database fill values for you.
- `ALTER TABLE` evolves schemas, but some operations lock the table — respect that in production.

## Test your understanding
1. Why enforce a rule with a `CHECK` constraint instead of in application code?
2. What two constraints does `PRIMARY KEY` combine, and what index does it create automatically?
3. A `UNIQUE` email column has three rows with `NULL` email and no error. Why? How would you forbid that?
4. You add `ON DELETE CASCADE` to `order_items.order_id`. What happens when an order is deleted, and what's the risk?
5. Why might adding a foreign key without an index on the FK column hurt performance later?

## Hands-on exercise
Extend your Chapter 02 schema into **two related tables** (e.g. `lists` and `tasks`, or `authors` and `books`):
1. Give each a `bigint identity` primary key.
2. Add a foreign key from the child to the parent with an explicit `ON DELETE` action you can justify.
3. Add at least one `CHECK` constraint and one `UNIQUE` constraint, both explicitly named.
4. Insert valid rows, then attempt: (a) a child row referencing a non-existent parent, (b) a duplicate of the unique value, (c) a row violating the check. Confirm each is rejected and read the error messages.
5. `ALTER TABLE` to add a new nullable column, then a `DEFAULT`.
