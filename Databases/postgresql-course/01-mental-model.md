# Chapter 01 — The Mental Model: What PostgreSQL Actually Is

## Prerequisites
- Comfort with a terminal (running commands, navigating directories).
- Helpful but not required: any prior exposure to thinking about data as tables; basic programming intuition.

You do **not** need another database, advanced math, or a specific programming language. SQL is its own thing, and we build it from zero.

---

## 1. The simplest possible definition

A **database** is an organized collection of data. A **DBMS** (Database Management System) is the software that stores, protects, and serves it. PostgreSQL is a specific DBMS — more precisely a **relational** DBMS (RDBMS).

The word "relational" is slightly misleading. People assume it means "tables related by foreign keys." That's a *consequence*, not the root meaning.

> **The "relation" in relational databases is the table itself.** A relation is a mathematical concept — a set of rows (tuples) sharing the same set of columns (attributes). The model comes from Edgar Codd's 1970 relational theory, which is *why* SQL behaves the way it does.

```
        TABLE (a "relation")  =  employees
   ┌─────────┬──────────────┬───────────┬────────────┐
   │  id     │  name        │  salary   │ dept_id    │   ← columns (attributes)
   ├─────────┼──────────────┼───────────┼────────────┤
   │  1      │  Asha        │  90000    │  3         │   ← a row (a tuple / record)
   │  2      │  Ravi        │  75000    │  3         │
   │  3      │  Meena       │  120000   │  1         │
   └─────────┴──────────────┴───────────┴────────────┘
        each column has a fixed TYPE and rules (constraints)
```

Two things make this powerful and different from a spreadsheet:

1. **Every value in a column must obey that column's type and rules.** `salary` can't accidentally hold `"forty thousand"`. The database *refuses* bad data. A spreadsheet lets you type garbage anywhere.
2. **You query by describing *what* you want, not *how* to get it.** This is *declarative* querying — the single most important idea in this chapter.

## 2. Declarative querying — the "why" behind SQL

Most programming is *imperative*: you write the steps (loop, check, collect). SQL is *declarative*:

```sql
SELECT name, salary
FROM employees
WHERE salary > 80000
ORDER BY salary DESC;
```

You never said *how* — whether to scan the whole table, use an index, what order to read disk pages. You described the **result**, and a component called the **query planner** chooses the fastest execution strategy.

**Analogy:** Imperative code is giving a taxi driver turn-by-turn directions. Declarative SQL is saying "take me to the airport, fastest route" and trusting the driver to know the roads and traffic. Liberating — but it's also *why* "why is my query slow?" becomes a real skill later: you must learn to read the driver's chosen route. That's `EXPLAIN` (Chapter 12).

## 3. Why PostgreSQL specifically?

- **ACID-compliant and correctness-obsessed.** It guards data integrity ferociously (ACID defined in Chapter 09). It won't quietly lose or corrupt data to take a shortcut.
- **Object-relational and extensible.** The crown jewel: you can define your own types, functions, operators, and even index methods. This is *why* PostGIS (geospatial) and pgvector (AI embeddings) exist as extensions — the engine was built to be extended.
- **Standards-compliant.** Follows the SQL standard faithfully, so your skills transfer.
- **Open source, no single corporate owner.** Developed by a global volunteer group under a permissive license.
- **Rich built-in types** — JSON/JSONB, arrays, ranges, full-text search, network addresses.

Orientation against neighbors:

```
SQLite     → tiny, file-based, no server. Great for apps/embedded; not multi-user at scale.
MySQL      → also a popular RDBMS; historically simpler. Postgres is generally favored for
             complex queries, correctness, and extensibility.
MongoDB    → document/NoSQL; schema-flexible but trades away relational integrity & joins.
             (Postgres does JSON documents too, via JSONB — often the best of both.)
Postgres   → the "serious general-purpose default." Hard to outgrow.
```

If you learn Postgres well, you rarely *need* to leave it, and your knowledge is highly portable.

## 4. How Postgres is structured (high-level architecture)

Postgres uses a **client/server** architecture with **multiple processes** (not threads):

```
   YOUR MACHINE / NETWORK                 THE POSTGRES SERVER
 ┌─────────────────────┐         ┌───────────────────────────────────────┐
 │  Client             │         │   postmaster (the supervisor process)  │
 │  • psql (terminal)  │ ──TCP──▶│        │                               │
 │  • a Python app     │  conn   │        ├─▶ backend process (for you)    │
 │  • a GUI tool       │         │        ├─▶ backend process (other user) │
 └─────────────────────┘         │        └─▶ background workers           │
                                 │              (autovacuum, WAL writer,   │
                                 │               checkpointer, etc.)       │
                                 │   ┌─────────────────────────────────┐  │
                                 │   │ Shared Memory (shared_buffers):  │  │
                                 │   │ cached data pages everyone uses  │  │
                                 │   └─────────────────────────────────┘  │
                                 │   Disk: data files + WAL (write-ahead  │
                                 │         log = durability)              │
                                 └───────────────────────────────────────┘
```

Key facts:

- A running Postgres instance is a **cluster** (nothing to do with Kubernetes clusters — it just means "one server managing a set of databases on one data directory").
- One cluster contains **multiple databases**. Each database contains **schemas** (namespaces), which contain **tables** and other objects. The default schema is `public`.
- On connect, the postmaster forks a dedicated **backend process** per connection. Each connection is relatively heavyweight — *why* connection pooling becomes essential at scale (Chapter 19).
- The **WAL (Write-Ahead Log)** records every change *before* it touches data files. It powers crash recovery, durability, and replication. Remember the name — it recurs in Chapters 09, 14, 21, 22.

## 5. Your first contact

After installing (Chapter 02 covers this properly), connect with `psql`:

```bash
psql -h localhost -U postgres
```

A taste of the whole loop — create a relation, put data in, ask a question:

```sql
CREATE TABLE employees (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    TEXT NOT NULL,
    salary  INTEGER CHECK (salary > 0)
);

INSERT INTO employees (name, salary) VALUES
    ('Asha', 90000),
    ('Ravi', 75000),
    ('Meena', 120000);

SELECT name, salary
FROM employees
WHERE salary > 80000
ORDER BY salary DESC;
```

Result:

```
 name  | salary
-------+--------
 Meena | 120000
 Asha  |  90000
```

Ravi is filtered out by `WHERE`; order is highest-first. And `INSERT ... VALUES ('Bad', -5);` would be **rejected** by the `CHECK` constraint. That rejection is the database doing its real job: being the last line of defense for correctness.

---

## Summary

- A relational database organizes data into **relations (tables)** — sets of typed rows sharing the same columns; "relational" refers to the tables themselves (Codd's model).
- The database **enforces correctness** via types and constraints, actively rejecting bad data.
- SQL is **declarative**: you describe the result; the **query planner** chooses how. This is the source of both Postgres's power and the future skill of tuning.
- **PostgreSQL** stands out for being correct (ACID), **extensible**, standards-compliant, and open source.
- Architecture: **client/server, process-per-connection**. One **cluster** holds many databases; the **WAL** provides durability and replication.

## Test your understanding
1. Why does "relational" not primarily mean "tables related by foreign keys"? What does it actually refer to?
2. Declarative vs imperative — which is SQL, and what's the practical consequence?
3. Name two things a database does that a spreadsheet does not regarding data correctness.
4. What does the postmaster do when a client connects, and why does that make pooling matter in production?
5. What is the WAL, and one problem it solves?

## Hands-on exercise
Don't write SQL yet — do a **modeling** exercise (where beginners most often go wrong). Pick a domain you understand (task tracker, movie collection, gym memberships). On paper:
1. Identify 2–3 tables.
2. For each, list columns and the **type** of each (text? whole number? date? true/false?).
3. Note any **rules** per column (unique? required? positive? must match another table?).
4. Write one plain-English **question** you'd ask the data ("which tasks are overdue?"). Don't write SQL yet.

Carry this design forward — Chapters 02–04 turn it into real tables.
