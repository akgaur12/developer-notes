# Chapter 14 — MVCC, VACUUM, and Bloat

## Where you are
You can make queries fast. Now you learn how Postgres handles concurrency *internally* — because the same mechanism that makes it great at concurrency (MVCC) creates a maintenance obligation (VACUUM) that, if neglected, is the single most common cause of mysterious production degradation and even outages. This is core operational knowledge.

> **The "why" (from Chapter 09):** Postgres lets readers and writers not block each other by keeping **multiple versions** of each row. That's MVCC. The flip side: obsolete versions pile up and must be reclaimed. Understanding this loop is what separates someone who *runs* Postgres from someone surprised by it at 2 a.m.

---

## 1. How MVCC actually works (the row version model)
Every row (tuple) carries hidden system columns, chiefly:
```
xmin   the transaction id that CREATED this row version
xmax   the transaction id that DELETED/superseded it (0/null if still live)
```
- An **INSERT** writes a tuple with `xmin = my_txid`.
- A **DELETE** doesn't erase the tuple; it sets `xmax = my_txid` (marks it dead-as-of that transaction).
- An **UPDATE** is a DELETE + INSERT: it marks the old tuple dead (`xmax`) and writes a **new** tuple version.

Each transaction has a **snapshot** — a list of which transaction ids were committed when it started. A tuple is *visible* to a transaction if its `xmin` is committed-and-visible and its `xmax` is not. This is how two transactions see different, consistent versions of the same table at the same time, with no locking on reads.
```
   UPDATE accounts SET balance=900 WHERE id=1;

   before:  (id=1, balance=1000, xmin=10, xmax=0)   ← live
   after :  (id=1, balance=1000, xmin=10, xmax=55)  ← now DEAD (old version, still on disk!)
            (id=1, balance=900,  xmin=55, xmax=0)   ← live (new version)
```
The old version doesn't disappear — it becomes a **dead tuple**. Readers with older snapshots may still need it; once no snapshot needs it, it's garbage.

## 2. Dead tuples and bloat
Dead tuples accumulate with every UPDATE and DELETE. If not reclaimed:
- **Table bloat:** the table file grows with dead versions, so scans read more pages for the same live data — queries slow down even though row counts haven't changed.
- **Index bloat:** indexes also accumulate dead entries.
This is *why* a table can get slower over time despite no growth in real data, and why `VACUUM` exists.

## 3. VACUUM — reclaiming dead space
```sql
VACUUM;                  -- mark dead tuple space reusable (does NOT shrink the file)
VACUUM ANALYZE orders;   -- also refresh planner statistics (Chapter 12)
VACUUM FULL orders;      -- rewrites the table to physically shrink it — but takes an EXCLUSIVE lock!
```
- Plain `VACUUM` makes dead space **reusable** by future inserts/updates; it runs online without blocking reads/writes. It does **not** return disk to the OS.
- `VACUUM FULL` actually shrinks the file by rewriting the table, but it **locks the table completely** for the duration — never run it casually on a live busy table. For online shrinking, use the `pg_repack` extension instead.

## 4. Autovacuum — the background janitor
You rarely run `VACUUM` by hand because **autovacuum** does it automatically: background workers vacuum and analyze tables once dead tuples or changes cross a threshold.
```
autovacuum triggers roughly when:
  dead_tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * table_rows
  (defaults: threshold 50, scale_factor 0.2  → vacuum after ~20% of the table churns)
```
> **The 20% problem on big tables:** a 0.2 scale factor means a 100-million-row table waits for ~20M dead tuples before autovacuum kicks in — far too late, causing heavy bloat. **Best practice: lower the scale factor per-table for large/hot tables:**
> ```sql
> ALTER TABLE big_hot_table SET (autovacuum_vacuum_scale_factor = 0.02);
> ```
Monitor it via `pg_stat_user_tables` (`n_dead_tup`, `last_autovacuum`).

## 5. The long-transaction hazard (a top production footgun)
`VACUUM` can only remove a dead tuple once **no snapshot still needs it**. A single long-running or forgotten-open transaction holds an old snapshot, so vacuum **cannot clean any dead tuples newer than that snapshot anywhere in the cluster** — bloat accumulates everywhere until that transaction ends.
```
   App opens a transaction, then hangs on a slow API call for 40 minutes.
   → autovacuum runs but can't remove dead tuples → tables bloat → everything slows.
```
This is *why* Chapter 09 stressed: **keep transactions short, never leave one open across user/network waits.** Watch for offenders in `pg_stat_activity` (look for old `xact_start` and `state = 'idle in transaction'`).

## 6. Transaction ID wraparound (the one that causes outages)
Transaction ids are a finite 32-bit counter (~4 billion). To prevent old `xmin` values from appearing "in the future" as the counter wraps, Postgres must "freeze" old rows (mark them as permanently visible) via vacuum before the counter laps. If autovacuum falls so far behind that wraparound looms, Postgres will:
1. emit increasingly urgent warnings, then
2. force aggressive "anti-wraparound" autovacuums, and ultimately
3. **refuse new writes** (read-only "to prevent data loss") until a vacuum freezes the old rows.

This has caused real outages at major companies. Modern Postgres (esp. v14+) handles freezing far better, and 64-bit transaction ids are on the roadmap, but the lesson stands: **don't disable autovacuum, and monitor table age.**
```sql
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind='r' ORDER BY xid_age DESC LIMIT 10;
```

## 7. HOT updates and fillfactor (a nice optimization to know)
A **HOT (Heap-Only Tuple)** update is an optimization: if an `UPDATE` doesn't change any indexed column *and* there's free space on the same page, Postgres can store the new version on the same page without updating every index — much cheaper, and self-cleaning. Leaving some free space per page helps:
```sql
ALTER TABLE hot_table SET (fillfactor = 90);   -- reserve 10% per page for in-place updates
```
This is *why* avoiding indexes on frequently-updated columns (and not over-indexing) helps write performance — it preserves HOT updates.

---

## Summary
- **MVCC** keeps multiple row versions (via hidden `xmin`/`xmax`); transactions see a consistent **snapshot**, so readers and writers don't block — the core of Postgres concurrency.
- UPDATE/DELETE leave **dead tuples**, which cause **table/index bloat** and gradual slowdown if not reclaimed.
- **`VACUUM`** marks dead space reusable online (no shrink); **`VACUUM FULL`** shrinks but locks the table (prefer `pg_repack` online). **Autovacuum** does this automatically — tune `scale_factor` lower for big hot tables.
- **Long/idle-in-transaction sessions block vacuum cluster-wide** → keep transactions short; hunt offenders in `pg_stat_activity`.
- **Transaction ID wraparound** can force the DB read-only — never disable autovacuum; monitor `age(relfrozenxid)`.
- **HOT updates** + `fillfactor` reduce index churn on frequently-updated rows.

## Test your understanding
1. What exactly happens on disk during an `UPDATE` in MVCC, and what is a "dead tuple"?
2. Why can a table get *slower* over time even though its live row count is unchanged?
3. Difference between `VACUUM` and `VACUUM FULL`, and why is the latter dangerous on a live table?
4. Why can a single forgotten-open transaction cause bloat across many tables?
5. What is transaction ID wraparound and what's the worst outcome if autovacuum can't keep up?

## Hands-on exercise
1. Create a table, insert rows, then run many `UPDATE`s on them. Check `n_dead_tup` and `n_live_tup` in `pg_stat_user_tables`.
2. Observe table size (`SELECT pg_size_pretty(pg_total_relation_size('t'))`) growing despite a constant live row count; run `VACUUM` and `VACUUM FULL` and compare sizes.
3. In one session run `BEGIN; SELECT 1;` and leave it open; in another, churn a table and run `VACUUM` — observe dead tuples *not* getting cleaned. Commit the first session and re-vacuum.
4. Inspect `pg_stat_activity` for `idle in transaction` sessions and their `xact_start`.
5. Lower `autovacuum_vacuum_scale_factor` on a test table and reason about why that helps for large tables.
