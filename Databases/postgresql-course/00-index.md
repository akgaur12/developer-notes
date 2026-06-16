# PostgreSQL: From Absolute Beginner to Professional

A structured, self-paced course. Each chapter builds on the previous one — read them in order. Every chapter follows the same shape: where you are, simple concepts first, then depth, the *why* behind everything, examples and ASCII diagrams, a summary, comprehension questions, and a hands-on exercise.

> **Version note:** Written against **PostgreSQL 18** (current stable as of 2026; PostgreSQL 19 in beta). Everything here applies to versions 14–19. Install the latest stable minor release.

---

## How to use this course

1. **Don't just read — type.** Keep a live Postgres open (Docker is easiest: see Chapter 02) and run every example yourself.
2. **Do the exercises.** Reading SQL builds recognition; writing it builds skill. They're different.
3. **Chase the "why."** Each chapter explains not only how something works but why it was designed that way. That understanding is what separates someone who *uses* Postgres from someone who can *operate* it.
4. **Revisit.** Indexing, MVCC, and query planning especially reward a second read once you've felt the problems they solve.

---

## The Roadmap

### Beginner — "I can store and retrieve data correctly"
- **01** — The Mental Model: what a relational database really is, and why Postgres
- **02** — Installation, `psql`, and Data Types
- **03** — Constraints & DDL: schemas that guarantee integrity
- **04** — DML: INSERT, SELECT, UPDATE, DELETE, and shaping results

### Intermediate — "I can model real domains and ask hard questions"
- **05** — Relationships & Normalization
- **06** — JOINs
- **07** — Aggregation & Grouping
- **08** — Subqueries & CTEs
- **09** — Transactions & ACID
- **10** — Views & Materialized Views
- **11** — Indexes: Fundamentals

### Advanced / Professional — "I can run this in production and make it fast"
- **12** — Query Planning: EXPLAIN and the cost model
- **13** — Index Deep-Dive: B-tree, GIN, GiST, BRIN, Hash & strategies
- **14** — MVCC, VACUUM, and Bloat
- **15** — JSON & JSONB
- **16** — Full-Text Search
- **17** — Window Functions
- **18** — Partitioning
- **19** — Performance Tuning & Connection Pooling
- **20** — Security: Roles, Privileges & Row-Level Security
- **21** — Replication & High Availability
- **22** — Backup & Recovery
- **23** — Extensions (pg_stat_statements, PostGIS, pgvector, …)
- **24** — Operating Postgres in Production (Docker/Kubernetes, monitoring, upgrades)
- **25** — Capstone Projects, Resources & What Next

---

## Milestones to aim for

| Tier | You can… |
|------|----------|
| Beginner | Design a small schema with proper types and constraints, and run all CRUD operations correctly. |
| Intermediate | Build a normalized multi-table schema with referential integrity and answer analytical questions with joins, aggregation, and CTEs. |
| Advanced | Read a query plan, add the right index to make a slow query dramatically faster, and stand up a primary/replica with a sane backup strategy. |

---

## Core reference resources (used throughout)

- **Official documentation** — `https://www.postgresql.org/docs/` (genuinely excellent; the reference you'll keep returning to).
- **`use-the-index-luke.com`** — the best free resource for indexing intuition.
- **Books** — *The Art of PostgreSQL* (Dimitri Fontaine); *PostgreSQL: Up and Running* (Obe & Hsu); *PostgreSQL 14 Internals* (Egor Rogov, free PDF — concepts apply broadly).
- **`pgexercises.com`** — interactive query practice.
- **`explain.dalibo.com`** — visualizes query plans (used in Chapter 12).

Good luck. Start with **01-mental-model.md**.
