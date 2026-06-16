# Chapter 12 — Query Planning: EXPLAIN and the Cost Model

## Where you are
You've created indexes — but how do you *know* they're being used, or *why* a query is slow? This chapter teaches you to read the planner's mind. Reading `EXPLAIN` output is the defining skill of a professional Postgres user: it's how you turn "the app feels slow" into "this query does a sequential scan; add this index." Everything in the advanced tier connects back here.

> **The "why" (recall Chapter 01):** SQL is declarative — you say *what*, the **planner** decides *how*. The planner estimates the cost of many possible execution strategies and picks the cheapest. `EXPLAIN` shows you the plan it chose; `EXPLAIN ANALYZE` shows what actually happened when it ran.

---

## 1. EXPLAIN vs EXPLAIN ANALYZE
```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- shows the PLAN and ESTIMATES, without running the query

EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- actually RUNS it and shows estimates vs ACTUAL times and row counts
```
> **Warning:** `EXPLAIN ANALYZE` executes the query — including `INSERT`/`UPDATE`/`DELETE`. To analyze a write safely, wrap it: `BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;`.

The most useful form adds buffer (I/O) information:
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;
```

## 2. Reading a plan
Plans are **trees**, read inside-out / bottom-up: the most indented nodes run first, feeding their parents.
```
Sort  (cost=812.34..823.10 rows=4305 width=58) (actual time=12.1..12.4 rows=4290 loops=1)
  Sort Key: created_at DESC
  ->  Index Scan using idx_orders_customer on orders
        (cost=0.42..501.20 rows=4305 width=58) (actual time=0.03..7.8 rows=4290 loops=1)
        Index Cond: (customer_id = 42)
```
Each node shows:
```
cost=startup..total   the planner's ESTIMATE in arbitrary cost units (not ms; lower = cheaper)
rows=...              ESTIMATED rows the node emits
width=...             estimated average row size in bytes
actual time=...       (ANALYZE only) real time in ms: first row .. last row
actual rows=...       (ANALYZE only) real rows produced
loops=...             how many times this node executed (multiply for nested loops!)
```

## 3. The scan and join node types you must recognize
```
SCANS
  Seq Scan        read the whole table. Fine for small tables; a red flag on large ones with a selective filter.
  Index Scan      use an index to find rows, then fetch them from the table.
  Index Only Scan  answer entirely from the index (covering index / all needed cols indexed) — fastest.
  Bitmap Index Scan + Bitmap Heap Scan  build a bitmap of matching rows, then fetch in physical order —
                  chosen when a query matches "many but not most" rows.

JOINS
  Nested Loop     for each outer row, look up matches (great when one side is tiny / well-indexed).
  Hash Join       build a hash table of one side, probe with the other (great for large unsorted joins).
  Merge Join      both inputs sorted, walk them together (great when inputs are already ordered).
```
The planner picks among these based on estimated row counts and available indexes. None is "best" universally — a Nested Loop is ideal for a few rows and catastrophic for millions.

## 4. The four diagnostic questions
When a query is slow, ask in order:

**1) Is there a Seq Scan where a selective filter should use an index?**
A `Seq Scan` on a big table with `WHERE selective_col = x` usually means a missing or unusable index. (But a Seq Scan is *correct* when the query returns most of the table — reading it all sequentially beats random index lookups.)

**2) Do estimated rows match actual rows?**
A big gap (`rows=10` estimated, `actual rows=100000`) means the planner has **stale or insufficient statistics** and is making bad choices. Fix:
```sql
ANALYZE orders;          -- refresh statistics (autovacuum does this, but not always promptly)
```
Bad estimates are the root cause of most "the planner chose a dumb plan" situations.

**3) Where is the time actually going?**
Find the node with the largest `actual time` (remembering to multiply by `loops`). That's your bottleneck — optimize *that* node, not your guess.

**4) Is a Nested Loop running a huge number of loops?**
`loops=500000` on an inner index scan means it ran half a million times. Often fixed by an index on the inner side, or by nudging the planner toward a Hash/Merge join via better statistics.

## 5. The cost model and key configuration knobs
The planner's cost is a weighted estimate of disk page reads and CPU. A few settings shape its choices (more in Chapter 19):
```
random_page_cost       cost of a random page read (default 4.0). On SSDs, lowering to ~1.1
                       makes the planner more willing to use indexes (random I/O is cheap on SSD).
effective_cache_size   the planner's ESTIMATE of OS+DB cache available. Set it generously
                       (~50-75% of RAM) so the planner knows index pages are likely cached.
```
These don't change correctness, only which plan looks cheapest. Misconfigured `random_page_cost` on SSD hardware is a classic reason Postgres "refuses to use my index."

## 6. Why a "correct" index still isn't used
The planner may rationally skip an index when:
- The query returns a **large fraction** of the table (Seq Scan is genuinely cheaper).
- **Statistics are stale** (run `ANALYZE`).
- The condition isn't index-compatible (`lower(col)` without an expression index; leading-wildcard `LIKE`).
- The table is **tiny** (scanning a few pages beats index overhead).
- A type mismatch forces a cast that defeats the index (e.g. comparing a `text` column to a number).

Don't fight the planner blindly — read the plan, fix the actual cause (usually statistics or a missing/expression index), and re-check.

## 7. Tooling
- `explain.dalibo.com` and `explain.depesz.com` paste a plan and visualize it — invaluable for big plans.
- `auto_explain` (a built-in module) logs plans of slow queries automatically in production.
- `pg_stat_statements` (Chapter 23) tells you *which* queries to run `EXPLAIN` on by ranking them by total time.

---

## Summary
- `EXPLAIN` shows the planner's chosen **plan + estimates**; `EXPLAIN (ANALYZE, BUFFERS)` runs it and shows **actuals + I/O**. Wrap analyzed writes in `BEGIN ... ROLLBACK`.
- Plans are **trees read bottom-up**; each node reports cost (arbitrary units), estimated/actual rows, and `loops` (multiply for nested loops).
- Know the **scan types** (Seq, Index, Index-Only, Bitmap) and **join types** (Nested Loop, Hash, Merge) and when each is appropriate.
- Diagnose by asking: unexpected **Seq Scan**? **estimate vs actual** row mismatch (→ `ANALYZE`)? where's the **real time**? a **huge-loop Nested Loop**?
- The planner is cost-based; **`random_page_cost`** and **`effective_cache_size`** strongly influence index usage — tune them for SSD/RAM reality.

## Test your understanding
1. What's the difference between `EXPLAIN` and `EXPLAIN ANALYZE`, and why is the latter dangerous on an `UPDATE`?
2. You see `rows=5` estimated but `actual rows=80000` on a node. What's the likely cause and the fix?
3. When is a `Seq Scan` actually the *right* choice, not a bug?
4. In a Nested Loop, why must you multiply a node's time by `loops` to find the real cost?
5. Your query has a perfect index but the plan ignores it on your SSD server. Name two plausible reasons.

## Hands-on exercise
Reuse the half-million-row table from Chapter 11:
1. Run `EXPLAIN ANALYZE` on a selective `WHERE` before and after adding an index; identify the change from `Seq Scan` to `Index Scan` and the time difference.
2. Find a query where the planner chooses a `Bitmap Heap Scan` (a filter matching many-but-not-most rows).
3. Deliberately make statistics stale (insert many rows without `ANALYZE`), observe a bad estimate, run `ANALYZE`, and watch the plan improve.
4. Write a two-table join and identify whether Postgres chose a Nested Loop, Hash, or Merge join — and reason about why.
5. Paste a plan into `explain.dalibo.com` and locate the most expensive node visually.
