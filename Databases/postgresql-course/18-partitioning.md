# Chapter 18 — Partitioning

## Where you are
Some tables grow enormous — billions of rows of events, logs, or time-series. Even with good indexes, a single giant table becomes unwieldy: queries slow, maintenance (`VACUUM`, index rebuilds) takes forever, and deleting old data is painful. **Partitioning** splits one logical table into many physical pieces, so the database only touches the relevant parts. This is a professional-scale tool — powerful, but easy to misapply.

> **The "why":** If you only ever query the last 7 days but the table holds 5 years, every query and every vacuum still has to reckon with all 5 years. Partitioning lets Postgres *skip* the irrelevant 99% entirely (partition pruning), and lets you drop old data by detaching a whole partition instantly instead of a slow mass `DELETE`.

---

## 1. The model: one parent, many child partitions
Since PostgreSQL 10, Postgres has **declarative partitioning**: you define a parent table partitioned by a key, then create child partitions. To queries, it still looks like one table.
```
                 orders  (parent, partitioned BY RANGE (created_at))
                /        |          \
   orders_2024   orders_2025   orders_2026     ← child partitions (real tables)
   (Jan–Dec 24)  (Jan–Dec 25)  (Jan–Dec 26)
```

## 2. Three partitioning strategies
```
RANGE   rows fall into ranges of the key. Classic for time-series (by month/year) or numeric ranges.
LIST    rows match a discrete list of values. e.g. partition by region: 'US', 'EU', 'APAC'.
HASH    rows distributed by a hash of the key into N partitions. For even spread when there's no
        natural range/list (e.g. partition by hash of customer_id for balanced load).
```

**RANGE example (the most common):**
```sql
CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id BIGINT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    total       NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (id, created_at)        -- ⚠ the partition key MUST be part of any PK/UNIQUE
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- a default partition catches anything that fits no range (optional, use with care):
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```
> **The PK constraint that surprises everyone:** any `PRIMARY KEY` or `UNIQUE` constraint on a partitioned table **must include the partition key**. You cannot have a globally-unique `id` alone if you partition by `created_at` — uniqueness is enforced per-partition. This is a fundamental trade-off of declarative partitioning; plan your keys around it.

## 3. Partition pruning — the payoff
When a query filters on the partition key, the planner **prunes** irrelevant partitions and never touches them:
```sql
EXPLAIN SELECT * FROM orders WHERE created_at >= '2026-06-01';
-- plan touches only orders_2026 — orders_2025 and earlier are skipped entirely
```
This is the whole point: a query against a 5-year, billion-row logical table reads only the one partition it needs. **Pruning only works if the query filters on the partition key** — a query without that filter scans every partition (and may be *slower* than an unpartitioned table). Design partitions around your dominant query's filter.

## 4. The operational superpowers
**Instant data retention.** Dropping old data is a metadata operation, not a billion-row `DELETE` (which would bloat the table and hammer vacuum):
```sql
ALTER TABLE orders DETACH PARTITION orders_2024;   -- instantly remove it from the set
DROP TABLE orders_2024;                            -- or archive it elsewhere first
```
**Bulk load then attach.** Build and index a partition offline, then attach it.
**Per-partition maintenance.** `VACUUM`/`REINDEX` run on one manageable partition at a time instead of one monstrous table.

## 5. Sub-partitioning and automation
Partitions can themselves be partitioned (e.g. by month, then by region). Creating partitions ahead of time is essential — an insert with no matching partition fails (unless a `DEFAULT` exists). In practice you **automate** partition creation with a scheduled job or the **`pg_partman`** extension, which manages rolling time-based partitions and retention for you. Don't hand-create monthly partitions forever.

## 6. When to partition — and when NOT to
```
PARTITION when:
  • the table is very large (rule of thumb: hundreds of GB / hundreds of millions+ rows)
  • there's a natural partition key your queries filter on (usually time)
  • you need cheap time-based data retention (drop old partitions)
  • maintenance on one huge table has become impractical

DON'T partition when:
  • the table is small/medium — indexes alone are simpler and faster; partitioning adds overhead
  • your queries DON'T filter on the partition key (no pruning → you've made things worse)
  • you'd create thousands of tiny partitions (planning overhead, too many relations)
```
> **The honest caution:** partitioning is frequently applied prematurely. It adds real complexity (key constraints, partition management, planning overhead) and only pays off at genuine scale with a query pattern that prunes. For most tables, **proper indexing (Chapters 11–13) is the right answer, not partitioning.** Reach for it when you've actually hit the wall, not preemptively.

---

## Summary
- **Declarative partitioning** (PG10+) splits one logical table into physical child partitions by **RANGE** (time/numeric), **LIST** (discrete values), or **HASH** (even spread).
- Any **PK/UNIQUE must include the partition key** — uniqueness is per-partition.
- **Partition pruning** skips irrelevant partitions *only when the query filters on the partition key* — this is the core benefit and the core requirement.
- Operational wins: **instant retention** via `DETACH`/`DROP`, bulk-load-then-`ATTACH`, and **per-partition maintenance**.
- Automate partition creation/retention (e.g. **`pg_partman`**).
- **Don't partition prematurely** — for most tables, indexing is the right tool; partition only at true scale with a pruning-friendly query pattern.

## Test your understanding
1. What are the three partitioning strategies and a good use case for each?
2. Why must a partitioned table's primary key include the partition key, and what limitation does that impose?
3. What is partition pruning, and what query characteristic is required for it to happen?
4. Why is dropping a month of old data via `DETACH`/`DROP PARTITION` vastly better than `DELETE FROM ... WHERE created_at < ...`?
5. Give two situations where partitioning would *hurt* rather than help.

## Hands-on exercise
1. Create a `RANGE`-partitioned table on a timestamp, with at least three monthly/yearly partitions and a `DEFAULT` partition.
2. Insert rows spanning multiple partitions; confirm with `\d+ parent` that they landed in the right children.
3. Run `EXPLAIN` on a query filtered by the partition key and confirm pruning (only one partition scanned); then run one *without* the key filter and observe all partitions scanned.
4. `DETACH` and `DROP` the oldest partition; confirm the data is gone instantly and the rest is intact.
5. Write (in comments) how you'd automate monthly partition creation and a 12-month retention policy.
