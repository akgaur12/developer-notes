# Chapter 25 — Capstone Projects, Resources & What Next

## Where you are
You've gone from "what is a relation?" to operating Postgres in production. This final chapter is about **consolidation**: three graded projects that force you to combine everything, a curated list of where to keep learning, and an honest map of what lies beyond this course. Reading builds recognition; *building* builds skill — so the real finish line is the projects below.

> **The "why":** Knowledge you haven't applied is fragile. Each project deliberately spans many chapters so the pieces connect into a working whole — which is exactly how the knowledge becomes durable and usable under pressure.

---

## Project 1 (Beginner→Intermediate): A Bookstore / Inventory System
**Goal:** model a real domain correctly and answer analytical questions.

Requirements:
1. **Schema (Chapters 02–05):** `authors`, `books`, `customers`, `orders`, `order_items` with proper types, a `bigint identity` PK on each, `timestamptz` timestamps, `numeric` for money, and a many-to-many (`books` ↔ `authors`) via a junction table. Normalize to 3NF; store `unit_price` on `order_items` (price-at-sale).
2. **Integrity (Chapter 03):** `NOT NULL`, `CHECK` (e.g. `quantity > 0`, `price >= 0`), `UNIQUE` (e.g. ISBN), foreign keys with justified `ON DELETE` actions. Index the FK columns.
3. **CRUD + queries (Chapters 04, 06–08):** seed data; write joins across the whole chain (customer → order → items → book → author); use aggregation for "revenue per author" and "top 5 customers by spend"; use a CTE and a correlated/`EXISTS` query (e.g. "customers who never ordered a specific author").
4. **A view (Chapter 10):** an `order_summary` view joining the chain with computed line totals.

**Done when:** you can answer "monthly revenue," "best-selling book per author," and "customers with zero orders" with correct, readable SQL.

## Project 2 (Intermediate→Advanced): A Multi-Tenant Task/Project Tracker
**Goal:** combine modeling with performance and security.

Requirements:
1. **Schema:** `tenants`, `users`, `projects`, `tasks`, `comments`, with `tasks` carrying a `status` enum, `priority`, `due_at timestamptz`, a `tags TEXT[]`, and a `metadata JSONB` column (Chapter 15).
2. **Window functions (Chapter 17):** "rank tasks by due date within each project," a "running count of completed tasks per week," and "each task's age vs the project average."
3. **Indexing & plans (Chapters 11–13):** generate enough rows to matter (`generate_series`), then make three representative queries fast — include a composite index (equality-then-range), a partial index (e.g. open tasks only), and a GIN index on `tags` or `metadata`. Prove each with `EXPLAIN (ANALYZE, BUFFERS)`.
4. **Full-text search (Chapter 16):** a `tsvector` generated column over task titles/descriptions with a GIN index and a `websearch_to_tsquery` search.
5. **Row-Level Security (Chapter 20):** enable RLS so each tenant sees only its own rows via a `current_setting`-based policy; index the `tenant_id`; prove isolation with two tenant contexts.
6. **Transactions (Chapter 09):** wrap a "move task + log activity" operation in a transaction; demonstrate rollback on a forced error.

**Done when:** queries are index-backed (verified in plans), search works, and tenant isolation is enforced by the database, not the app.

## Project 3 (Advanced/Professional): An Events / Time-Series Analytics Pipeline
**Goal:** operate Postgres at scale.

Requirements:
1. **Partitioning (Chapter 18):** an `events` table `RANGE`-partitioned by month; automate (or script) partition creation and a retention policy that `DETACH`/`DROP`s old partitions.
2. **Bulk load & MVCC awareness (Chapter 14):** load millions of rows; observe dead tuples under updates; tune `autovacuum_vacuum_scale_factor` on the hot table and justify it.
3. **Materialized views (Chapter 10) + scheduling (Chapter 23):** a `daily_metrics` materialized view refreshed by `pg_cron`; use `REFRESH ... CONCURRENTLY`.
4. **Performance tuning (Chapters 12, 19, 23):** enable `pg_stat_statements`, find the top offenders, `EXPLAIN` and fix them; tune `work_mem`/`random_page_cost` and measure the difference; put PgBouncer in front.
5. **HA & backups (Chapters 21, 22):** stand up a streaming replica; configure WAL archiving + a base backup; perform a **PITR** to a chosen timestamp; document RPO/RTO.
6. **Operations (Chapter 24):** complete the production-readiness checklist and a monitoring plan (what you'd alert on).

**Done when:** the system ingests at volume, prunes partitions, serves fast pre-aggregated metrics, survives a simulated primary failure, and can be restored to a point in time — and you can explain every operational choice.

---

## Where to keep learning

**Primary reference**
- **Official documentation** — `https://www.postgresql.org/docs/` — genuinely the best database documentation in existence. Read the "Server Administration" and "Performance Tips" sections in full at least once. The release notes for each major version are a great way to track what's new.

**Indexing & performance**
- **`use-the-index-luke.com`** — the clearest explanation of how indexes really work; free.
- **`explain.dalibo.com`** / **`explain.depesz.com`** — paste and visualize query plans.
- **`pgmustard.com`** blog and **`pganalyze.com`** blog — practical, deep performance writing.

**Books**
- *The Art of PostgreSQL* — Dimitri Fontaine — writing genuinely good SQL (thinking in sets).
- *PostgreSQL: Up and Running* — Obe & Hsu — practical administration and features.
- *PostgreSQL 14 Internals* — Egor Rogov — free PDF; the deepest accessible look at MVCC, vacuum, the planner, and indexes. Highly recommended once this course feels comfortable.
- *Designing Data-Intensive Applications* — Martin Kleppmann — not Postgres-specific, but the best book on the *concepts* (replication, consistency, storage engines) underpinning Chapters 09, 14, 21.

**Practice**
- **`pgexercises.com`** — interactive query exercises with solutions.
- Rebuild a real app's schema from scratch and load real-ish data with `generate_series` / `mockaroo`.

**Staying current**
- **Planet PostgreSQL** (`planet.postgresql.org`) — aggregated community blogs.
- **`postgresweekly.com`** — a weekly newsletter.
- Major releases land roughly annually (~September/October); skim each release's notes.

---

## What's beyond this course
You now have professional working command of Postgres. Directions to go deeper, each a field of its own:
```
• Server-side programming: PL/pgSQL functions, triggers, and stored procedures in depth;
  other languages (PL/Python, PL/v8) for in-database logic.
• Logical decoding & Change-Data-Capture: streaming changes to Kafka/data pipelines (Debezium).
• Advanced extensions: TimescaleDB (time-series), Citus (distributed/sharded Postgres for
  horizontal scale), PostGIS mastery, pgvector at scale for AI retrieval.
• Internals: the planner's join-order search, custom operator classes, writing an extension in C.
• Fleet operations: infrastructure-as-code for Postgres, chaos/failover testing, capacity planning,
  zero-downtime schema migrations at scale (expand/contract pattern, NOT VALID → VALIDATE).
```

## A final word on mindset
The throughline of this whole course: **understand the "why."** Postgres rewards people who know what the engine is doing underneath — why a plan was chosen, why a row version lingers, why a connection is expensive, why a constraint exists. When something behaves unexpectedly, you now have the mental models to reason from first principles instead of guessing: declarative queries and the planner (Ch 1, 12), MVCC and vacuum (Ch 9, 14), the WAL behind durability/replication/backup (Ch 9, 21, 22), and least privilege (Ch 20). Keep asking "why is it doing that?", keep reaching for `EXPLAIN` and the `pg_stat_*` views, and keep building. That habit — not any single fact — is what makes a professional.

Congratulations on finishing. Now go build Project 1.

---

## Final self-assessment
You're ready to call yourself professionally competent with Postgres when you can, without looking things up:
1. Design a normalized schema with correct types, constraints, and indexed foreign keys — and say *why* each choice.
2. Read an `EXPLAIN ANALYZE` plan, spot the bottleneck, and fix it with the right index or rewrite.
3. Explain MVCC, why VACUUM exists, and how a long-open transaction or wraparound can cause an outage.
4. Choose and justify an index *type* (B-tree/GIN/GiST/BRIN) for a given query and data shape.
5. Stand up replication + a tested backup/PITR strategy, and explain why you need both.
6. Lock down a database with least-privilege roles and (where needed) row-level security.
7. Diagnose a production slowdown using `pg_stat_statements`, `pg_stat_activity`, and the other stat views.

If any of those feel shaky, revisit that chapter and do its exercise for real on live data. That's the whole game.
