# Chapter 10 — Views & Materialized Views

## Where you are
You can write complex queries. **Views** let you save and name them so they behave like virtual tables — improving reuse, readability, and security. **Materialized views** cache the *results* for speed. This is your first taste of managing complexity and performance at the schema level.

---

## 1. A view is a saved query (a virtual table)
```sql
CREATE VIEW active_customers AS
SELECT id, name, email
FROM customers
WHERE deleted_at IS NULL;
```
Now query it like a table:
```sql
SELECT * FROM active_customers WHERE name LIKE 'A%';
```
A view stores **no data** — it's a named query that runs fresh each time. Querying `active_customers` is exactly like inlining its definition. So a view is *always current* but costs the same as running its query.

**Why views earn their keep:**
- **Abstraction / reuse:** wrap a gnarly 5-table join once; everyone queries the simple view. Change the join in one place and all callers benefit.
- **A stable interface:** the underlying tables can be refactored while the view's columns stay constant for applications.
- **Security:** grant access to a view exposing only safe columns/rows, without granting access to the base table. A view selecting non-sensitive columns lets you hide salaries or PII (pairs with Chapter 20).

## 2. Updatable views
A simple view (one table, no aggregation/DISTINCT/GROUP BY) is automatically **updatable** — you can `INSERT`/`UPDATE`/`DELETE` through it and it affects the base table:
```sql
UPDATE active_customers SET name = 'Asha R.' WHERE id = 5;   -- updates customers
```
Add a guard so writes can't "escape" the view's filter:
```sql
CREATE VIEW active_customers AS
SELECT id, name, email FROM customers WHERE deleted_at IS NULL
WITH CHECK OPTION;   -- rejects inserts/updates that wouldn't satisfy WHERE deleted_at IS NULL
```
Complex views (joins/aggregates) aren't auto-updatable, but you can make them writable with `INSTEAD OF` triggers — an advanced escape hatch you'll rarely need.

## 3. Materialized views — caching the result
A materialized view **stores** the query's output physically, like a snapshot. Reads are then as fast as reading a table — but the data is **stale** until you refresh it.
```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT date_trunc('month', created_at) AS month, SUM(total) AS revenue
FROM orders
GROUP BY 1;

SELECT * FROM monthly_revenue;            -- instant; reads the stored snapshot

REFRESH MATERIALIZED VIEW monthly_revenue;   -- recompute (locks the MV during refresh)
```
```
Regular VIEW         : query runs every time → always fresh, cost each read
Materialized VIEW    : result stored          → fast reads, but stale until REFRESH
```

**Concurrent refresh** avoids locking out readers during the rebuild (requires a unique index on the MV):
```sql
CREATE UNIQUE INDEX ON monthly_revenue (month);
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;   -- readers keep working during refresh
```

## 4. When to use which
```
Use a VIEW when:
  • you want reuse/abstraction/security and the underlying query is cheap enough to run live
  • data must always be current

Use a MATERIALIZED VIEW when:
  • the query is expensive (big aggregations, multi-table joins over lots of rows)
  • it's read far more often than the data changes
  • some staleness is acceptable (dashboards, daily reports, leaderboards)
  • you have a plan to refresh it (a cron job, a scheduled task, or pg_cron extension)
```
This is the controlled denormalization promised in Chapter 05: a materialized view is a managed, refreshable redundant copy — far safer than hand-maintained duplicate columns.

## 5. Practical considerations
- **Refresh strategy is the whole game.** Decide *when* and *how* to refresh: on a schedule (`pg_cron`/external cron), after relevant writes, or on demand. A materialized view nobody refreshes is just stale data with extra steps.
- **Refresh cost:** `REFRESH` recomputes the *entire* query — Postgres has no built-in incremental refresh. For huge MVs, consider rolling your own incremental update via triggers, or evaluate the `pg_ivm` extension.
- **Views don't speed up the underlying query.** A view over a slow query is still slow — to make it fast you need indexes (Chapter 11) or a materialized view.
- Inspect with `\dv` (views) and `\dm` (materialized views).

---

## Summary
- A **view** is a named, always-fresh saved query that stores no data — used for reuse, a stable interface, and security (exposing safe columns/rows).
- Simple views are **updatable**; add `WITH CHECK OPTION` to stop writes escaping the filter.
- A **materialized view** stores results for fast reads but goes **stale** until `REFRESH`; use `REFRESH ... CONCURRENTLY` (needs a unique index) to avoid blocking readers.
- Choose views for freshness and cheap queries; materialized views for expensive, read-heavy, staleness-tolerant workloads — and **always have a refresh plan**.

## Test your understanding
1. Does a regular view store data? What's the cost of querying it compared to running its definition directly?
2. Give two reasons to expose a view to an application instead of the base table.
3. What's the core trade-off a materialized view makes, and what does `REFRESH` do?
4. Why does `REFRESH MATERIALIZED VIEW CONCURRENTLY` require a unique index, and what problem does it solve?
5. You wrap a slow 4-table join in a view and it's still slow. Why, and what would actually help?

## Hands-on exercise
On your schema:
1. Create a view that joins two tables and exposes only a useful subset of columns; query it.
2. Add `WITH CHECK OPTION` to a simple single-table view and prove it rejects an out-of-filter write.
3. Create a materialized view of an expensive aggregation (e.g. totals per month/category). Time a read of it vs the live query with `\timing`.
4. Add a unique index and run `REFRESH MATERIALIZED VIEW CONCURRENTLY`.
5. Write a one-paragraph comment describing exactly how/when you'd refresh it in production.
