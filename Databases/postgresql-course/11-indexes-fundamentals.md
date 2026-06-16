# Chapter 11 — Indexes: Fundamentals

## Where you are
You can write any query. But a correct query can still be unbearably slow on real data. **Indexes** are the single biggest lever for query speed — and one of the easiest things to get wrong. This chapter builds the intuition; Chapter 13 goes deep on index types and strategy, and Chapter 12 teaches you to *measure* their effect.

> **The "why":** Without an index, finding matching rows means reading the *entire table* row by row (a "sequential scan"). An index is a pre-sorted side structure that lets Postgres jump straight to the rows it needs — turning a scan of millions of rows into a handful of lookups.

---

## 1. The analogy
A book without an index: to find every mention of "Postgres," you read all 400 pages. With an index at the back: you look up "Postgres," see "pages 12, 88, 240," and jump there. A database index is exactly this — a sorted lookup structure mapping values to row locations.

```
Sequential scan (no index)        Index scan (with index on email)
read EVERY row, check each        look up value in sorted tree → jump to matching rows
O(n) — grows with table size      O(log n) — barely grows
  1M rows → 1M reads                1M rows → ~20 reads
```

## 2. Creating an index
```sql
CREATE INDEX idx_employees_email ON employees (email);
```
That's it. From now on, `WHERE email = '...'` can use the index instead of scanning. (A `PRIMARY KEY` and `UNIQUE` constraint create their indexes automatically — which is why lookups *by primary key* are already fast.)

The default and dominant type is the **B-tree** (balanced tree), which keeps values sorted and supports:
```
=   equality
<, <=, >, >=, BETWEEN   range queries
ORDER BY on the indexed column(s)   (the index is already sorted!)
LIKE 'prefix%'          anchored prefix matches
IS NULL / IS NOT NULL
```

## 3. What indexes can and can't accelerate
An index on `(email)` helps `WHERE email = ...` but **not** `WHERE lower(email) = ...` (the function changes the value). For that, index the *expression*:
```sql
CREATE INDEX idx_emp_lower_email ON employees (lower(email));   -- now WHERE lower(email)=... uses it
```
Likewise `LIKE '%term'` (leading wildcard) **cannot** use a normal B-tree — the index is sorted by prefix, and you've thrown the prefix away. That's a job for trigram or full-text indexes (Chapters 13, 16).

## 4. Composite (multi-column) indexes and the leftmost-prefix rule
```sql
CREATE INDEX idx_orders_cust_date ON orders (customer_id, created_at);
```
This one index serves queries filtering on:
```
✔ customer_id alone
✔ customer_id AND created_at
✘ created_at alone        ← cannot use this index efficiently
```
> **The leftmost-prefix rule:** a composite index can be used for a query only if the query uses a *leading prefix* of its columns. Think of a phone book sorted by (last name, first name): great for "find Smith" or "find Smith, John," useless for "find everyone named John." **Column order matters enormously** — put the most selective / most-frequently-filtered column first, and match the order to your common query patterns.

## 5. The cost of indexes (why not index everything?)
Indexes are not free:
- **They slow down writes.** Every `INSERT`/`UPDATE`/`DELETE` must also update every affected index. A table with 8 indexes pays 8× the index-maintenance cost on writes.
- **They consume disk and memory.** Indexes can collectively exceed the size of the table.
- **Unused indexes are pure cost** — maintenance and space with zero benefit.

So indexing is a **trade-off**: faster reads for slower writes and more storage. Index the columns your real queries filter, join, and sort on — and no more.

## 6. Specialized B-tree variants you'll reach for early
**Partial index** — index only the rows you actually query, saving space and write cost:
```sql
CREATE INDEX idx_orders_unshipped ON orders (created_at) WHERE status = 'pending';
-- tiny index covering only pending orders; perfect if you constantly query the unshipped backlog
```
**Unique index** — enforce uniqueness *and* speed lookups (this is what a `UNIQUE` constraint builds):
```sql
CREATE UNIQUE INDEX idx_users_email ON users (email);
```
**Covering index (INCLUDE)** — bundle extra columns so a query can be answered from the index alone (an "index-only scan"), skipping the table read entirely:
```sql
CREATE INDEX idx_orders_cust ON orders (customer_id) INCLUDE (total);
-- SELECT total FROM orders WHERE customer_id = ? can be served without touching the table
```

## 7. Indexing foreign keys (the commonly-forgotten one)
Recall from Chapter 03: **FK columns are not auto-indexed.** Unindexed FK columns make joins slow and make `ON DELETE CASCADE`/parent deletes acquire broad locks while they scan children. As a near-default habit, index your foreign key columns:
```sql
CREATE INDEX idx_order_items_order ON order_items (order_id);
```

## 8. Maintenance basics
- `CREATE INDEX CONCURRENTLY` builds an index **without locking writes** on the table — essential in production (a plain `CREATE INDEX` blocks writes for the duration). It's slower and can't run inside a transaction block, but it's safe on live systems.
- Inspect indexes with `\d tablename`. Find unused ones via the `pg_stat_user_indexes` view (look for `idx_scan = 0`).
- Indexes can bloat over time; `REINDEX` (ideally `REINDEX ... CONCURRENTLY`) rebuilds them.

> Whether an index actually *gets used* is decided by the planner based on statistics and cost — which you'll learn to read in Chapter 12. Creating an index is necessary but not sufficient; you must verify it's being used.

---

## Summary
- An index is a **sorted side structure** that converts full-table scans (O(n)) into fast lookups (O(log n)). PKs/UNIQUE build them automatically.
- The default **B-tree** serves equality, ranges, sorting, prefix `LIKE`, and NULL checks; functions on the column need an **expression index**, and leading-wildcard `LIKE` needs a different index type.
- **Composite indexes** obey the **leftmost-prefix rule** — column order is critical; lead with the most selective/most-filtered column.
- Indexes **trade read speed for slower writes and more storage** — index what your queries use, not everything.
- Reach for **partial**, **unique**, and **covering (INCLUDE)** indexes; **index your foreign keys**; build with **`CONCURRENTLY`** in production; and verify usage with the planner (Chapter 12).

## Test your understanding
1. In plain terms, why does an index turn a million-row scan into ~20 reads?
2. You have `INDEX (customer_id, created_at)`. Which of these can use it well: filter on `customer_id`? on `created_at` alone? on both? Why?
3. Give two concrete costs of adding an index. Why not index every column?
4. `WHERE lower(email) = 'a@x.com'` ignores your index on `email`. Why, and what's the fix?
5. What does a covering/`INCLUDE` index enable, and why is that faster?

## Hands-on exercise
Use enough data to see real effects (generate rows with `generate_series`):
```sql
INSERT INTO big (val, created_at)
SELECT (random()*100000)::int, now() - (random()*365)::int * interval '1 day'
FROM generate_series(1, 500000);
```
1. Query `WHERE val = 42` and note the time with `\timing` (no index yet).
2. `CREATE INDEX` on `val`, run the same query, and compare the time.
3. Create a **composite** index and demonstrate the leftmost-prefix rule: a query on the first column uses it; one on only the second column does not.
4. Create a **partial** index for a common filtered subset and confirm it's smaller (`\di+`).
5. Create an index `CONCURRENTLY` and confirm via `pg_stat_user_indexes` which indexes are actually being scanned.
