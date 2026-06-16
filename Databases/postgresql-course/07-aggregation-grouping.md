# Chapter 07 — Aggregation & Grouping

## Where you are
You can combine tables with joins. Now you **summarize** data — counts, sums, averages, per-group breakdowns. This is where SQL turns raw rows into answers like "revenue per month" or "average order value per customer." It's the backbone of all reporting and analytics.

> **The "why":** Aggregation *collapses* many rows into fewer summary rows. Understanding exactly *what collapses into what* is the whole skill — most aggregation bugs come from a fuzzy picture of the grouping.

---

## 1. Aggregate functions (collapse a column of values into one)
```
COUNT(*)        number of rows
COUNT(col)      number of rows where col IS NOT NULL  ← NULLs are skipped!
COUNT(DISTINCT col)  number of distinct non-NULL values
SUM(col)        total
AVG(col)        mean (also skips NULLs)
MIN(col), MAX(col)
ARRAY_AGG(col)  collect values into an array
STRING_AGG(col, ', ')  concatenate values into a delimited string
```
Without `GROUP BY`, an aggregate collapses the **entire result** into one row:
```sql
SELECT COUNT(*), AVG(salary), MAX(salary) FROM employees;   -- one summary row
```

> **The NULL subtlety that bites everyone:** `COUNT(*)` counts rows; `COUNT(column)` counts non-NULL values in that column; `AVG`/`SUM` ignore NULLs entirely. So `AVG(salary)` over 10 employees where 2 have NULL salary divides by **8**, not 10. Decide whether that's what you want.

## 2. GROUP BY — one summary row per group
```sql
SELECT dept_id, COUNT(*) AS headcount, AVG(salary) AS avg_pay
FROM employees
GROUP BY dept_id;
```
```
Conceptually GROUP BY sorts rows into buckets, then runs each aggregate per bucket:

  employees                       result (one row per dept_id)
  dept 3: Asha, Ravi      ─────▶  dept 3 | headcount 2 | avg_pay …
  dept 1: Meena           ─────▶  dept 1 | headcount 1 | avg_pay …
```

**The iron rule of GROUP BY:** every column in the `SELECT` must either be (a) inside an aggregate function, or (b) listed in `GROUP BY`. Otherwise Postgres can't know which value to show for the group, and it errors. (Some databases silently pick a random value — Postgres correctly refuses. This strictness is a feature.)
```sql
-- ERROR: column "name" must appear in GROUP BY or an aggregate
SELECT dept_id, name, COUNT(*) FROM employees GROUP BY dept_id;
```

You can group by multiple columns (one row per *combination*):
```sql
SELECT dept_id, is_manager, COUNT(*)
FROM employees
GROUP BY dept_id, is_manager;
```

## 3. HAVING — filter groups (not rows)
`WHERE` filters rows *before* grouping; `HAVING` filters groups *after* aggregation. You need `HAVING` because aggregates don't exist yet when `WHERE` runs (recall the evaluation order from Chapter 04).
```sql
SELECT dept_id, COUNT(*) AS headcount
FROM employees
WHERE salary > 0          -- row filter, BEFORE grouping
GROUP BY dept_id
HAVING COUNT(*) > 5;      -- group filter, AFTER aggregation
```
```
FROM → WHERE (rows) → GROUP BY → HAVING (groups) → SELECT → ORDER BY → LIMIT
```
> **Pitfall:** putting an aggregate condition in `WHERE` (`WHERE COUNT(*) > 5`) is an error. Aggregate conditions belong in `HAVING`. Conversely, filtering plain rows in `HAVING` works but is wasteful — filter early in `WHERE` so fewer rows enter the grouping.

## 4. Aggregating across joins (the common real query)
Join first, then group:
```sql
SELECT c.name, COUNT(o.id) AS order_count, COALESCE(SUM(o.total), 0) AS lifetime_value
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id   -- LEFT JOIN so zero-order customers still appear
GROUP BY c.id, c.name                         -- group by id (the key); include name for output
ORDER BY lifetime_value DESC;
```
Two craft points here: (1) `LEFT JOIN` keeps customers with no orders, and `COUNT(o.id)` correctly returns 0 for them (because `COUNT(column)` skips the NULLs produced by the unmatched join); (2) group by the primary key `c.id` (guaranteed unique) and include `c.name` for display — grouping by id avoids accidentally merging two customers who happen to share a name.

## 5. FILTER — conditional aggregates (cleaner than CASE)
Postgres lets you aggregate a *subset* per group without separate queries:
```sql
SELECT
  COUNT(*)                                   AS total_orders,
  COUNT(*) FILTER (WHERE status = 'paid')    AS paid_orders,
  SUM(total) FILTER (WHERE status = 'paid')  AS paid_revenue
FROM orders;
```
This replaces the older `SUM(CASE WHEN status='paid' THEN 1 ELSE 0 END)` idiom and reads far better. (Both still work.)

## 6. GROUPING SETS, ROLLUP, CUBE — subtotals in one pass
For reports needing subtotals and grand totals:
```sql
SELECT dept_id, is_manager, COUNT(*)
FROM employees
GROUP BY ROLLUP (dept_id, is_manager);
-- yields per (dept,manager), per dept subtotal, and a grand total — in one query
```
`GROUPING SETS` lets you specify exactly which groupings you want; `CUBE` gives all combinations. These save multiple round-trips for dashboards.

---

## Summary
- Aggregate functions **collapse** values: `COUNT`, `SUM`, `AVG`, `MIN`/`MAX`, `ARRAY_AGG`, `STRING_AGG`. `COUNT(*)` counts rows; `COUNT(col)`/`AVG`/`SUM` **skip NULLs**.
- **GROUP BY** makes one summary row per group; every non-aggregated `SELECT` column must be in `GROUP BY` (Postgres enforces this strictly — a good thing).
- **WHERE** filters rows before grouping; **HAVING** filters groups after — and aggregate conditions must go in `HAVING`.
- For per-parent summaries, **LEFT JOIN then GROUP BY the key**, using `COUNT(child.id)` to get true zeros.
- **`FILTER (WHERE …)`** gives clean conditional aggregates; **`ROLLUP`/`CUBE`/`GROUPING SETS`** produce subtotals in one query.

## Test your understanding
1. A table has 100 rows; 30 have `NULL` in `score`. What do `COUNT(*)`, `COUNT(score)`, and `AVG(score)` (divided by what?) return conceptually?
2. Why does `SELECT dept_id, name, COUNT(*) FROM employees GROUP BY dept_id;` error?
3. You want only departments with more than 10 people. Which clause, and why not `WHERE`?
4. Why is grouping by `customer_id` safer than grouping by `customer_name` for a per-customer total?
5. Rewrite `SUM(CASE WHEN status='shipped' THEN total ELSE 0 END)` using `FILTER`.

## Hands-on exercise
On your schema:
1. Write a single-row summary: total count, average of a numeric column, and the min/max.
2. Write a `GROUP BY` producing one summary row per category, then add a second grouping column.
3. Add a `HAVING` clause to keep only groups above some threshold; confirm a `WHERE` row filter and the `HAVING` group filter coexist correctly.
4. Write a per-parent report with `LEFT JOIN ... GROUP BY parent.id` that shows `0` for parents with no children.
5. Use `COUNT(*) FILTER (WHERE ...)` to break a total into two conditional sub-counts in one query.
