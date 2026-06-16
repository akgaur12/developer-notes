# Chapter 08 — Subqueries & CTEs

## Where you are
You can join and aggregate. Now you learn to **compose** queries — use the result of one query inside another. This lets you express logic that a single flat query can't, and (with CTEs) write complex queries that are actually *readable*. This is the leap from "I can query" to "I can express anything."

---

## 1. Subqueries — a query inside a query
A subquery is a `SELECT` wrapped in parentheses, used in one of several positions.

**Scalar subquery (returns one value):**
```sql
SELECT name, salary,
       (SELECT AVG(salary) FROM employees) AS company_avg   -- one value, reused per row
FROM employees;
```

**Subquery in `WHERE` with IN / NOT IN:**
```sql
SELECT name FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE active = true);
```
> **Pitfall — `NOT IN` with NULLs:** if the subquery returns *any* NULL, `NOT IN` returns no rows (three-valued logic strikes again — "is x not in {1, NULL}?" evaluates to unknown). Prefer **`NOT EXISTS`** (below) for "not in another set"; it's NULL-safe and usually faster.

**Subquery in `FROM` (a derived table — must be aliased):**
```sql
SELECT dept_id, avg_pay
FROM (SELECT dept_id, AVG(salary) AS avg_pay FROM employees GROUP BY dept_id) AS dept_avgs
WHERE avg_pay > 80000;
```

## 2. Correlated subqueries — re-evaluated per outer row
A *correlated* subquery references a column from the outer query, so it runs once per outer row (conceptually):
```sql
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);
-- "employees earning more than their own department's average"
```
The inner query depends on `e.dept_id` from the outer row — that's the correlation. Powerful, but be aware it can be slow on large data (the planner often rewrites it, but not always); window functions (Chapter 17) frequently express the same idea faster.

## 3. EXISTS / NOT EXISTS — "does a related row exist?"
Often the clearest and most efficient way to test membership:
```sql
-- customers who have placed at least one order:
SELECT c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- customers who have NEVER ordered (NULL-safe anti-membership):
SELECT c.name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```
`EXISTS` stops at the first match (it only cares *whether* a row exists, not how many), so `SELECT 1` is idiomatic — the selected value is irrelevant.

## 4. CTEs — Common Table Expressions (the `WITH` clause)
A CTE names a subquery up front, so the main query reads top-to-bottom like steps in a recipe instead of nesting inside-out:
```sql
WITH dept_avgs AS (
    SELECT dept_id, AVG(salary) AS avg_pay
    FROM employees
    GROUP BY dept_id
),
high_paying AS (
    SELECT dept_id FROM dept_avgs WHERE avg_pay > 80000
)
SELECT e.name, e.salary
FROM employees e
JOIN high_paying h ON h.dept_id = e.dept_id;
```
This is the same logic as a nested subquery but vastly more readable, and CTEs can reference earlier CTEs — you build complex pipelines in legible stages.

> **Performance note (version-dependent — know this):** historically Postgres always *materialized* CTEs (computed them fully, like a temp table), which could be slower than an equivalent subquery. **Since PostgreSQL 12**, simple CTEs are *inlined* by default (optimized like subqueries) unless referenced multiple times or marked `MATERIALIZED`. You can force either behavior:
> ```sql
> WITH x AS MATERIALIZED (...)      -- force a temp result (an "optimization fence")
> WITH x AS NOT MATERIALIZED (...)  -- force inlining
> ```
> Materializing on purpose is useful to compute an expensive subresult once and reuse it.

## 5. Recursive CTEs — query hierarchies and graphs
The killer feature for trees (org charts, category trees, threaded comments) and graph traversal. A recursive CTE has a **base case** unioned with a **recursive step** that refers back to the CTE:
```sql
WITH RECURSIVE subordinates AS (
    -- base case: start at the CEO
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- recursive step: everyone who reports to someone already found
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT repeat('  ', level - 1) || name AS org_chart, level
FROM subordinates
ORDER BY level;
```
```
How it runs:
  level 1: CEO                        (base case)
  level 2: CEO's direct reports       (step 1)
  level 3: their reports              (step 2)
  ... until the recursive step returns no new rows, then it stops.
```
> **Pitfall:** if your data has a cycle (A reports to B reports to A), a recursive CTE loops forever. Guard with a depth limit (`WHERE level < 100`) or, in modern Postgres, the `CYCLE` clause that detects and stops on repeats.

## 6. Choosing between them
```
Need a single value to compare against        → scalar subquery
"Is there a related row?" / "no related row"   → EXISTS / NOT EXISTS  (NULL-safe)
A computed set to join against / filter by      → subquery in FROM, or a CTE
A multi-step query you (or a teammate) must read → CTE
A hierarchy / tree / graph                       → RECURSIVE CTE
A running total / rank / per-group calc          → window function (Chapter 17), often better
```

---

## Summary
- **Subqueries** nest a `SELECT` as a scalar value, an `IN`/`EXISTS` test, or a derived table (which must be aliased).
- **Correlated subqueries** reference the outer row and run per-row — expressive but watch performance.
- Prefer **`NOT EXISTS`** over `NOT IN` for anti-membership: it's NULL-safe and usually faster.
- **CTEs (`WITH`)** name subqueries for readability and staged pipelines; since PG12 they inline by default — use `MATERIALIZED`/`NOT MATERIALIZED` to control it.
- **Recursive CTEs** traverse hierarchies and graphs with a base case `UNION ALL` a recursive step; guard against cycles.

## Test your understanding
1. Why can `WHERE x NOT IN (SELECT y FROM t)` return zero rows unexpectedly, and what should you use instead?
2. What makes a subquery *correlated*, and what's the performance implication?
3. When does a CTE get materialized vs inlined in modern Postgres, and why might you force `MATERIALIZED`?
4. Write an `EXISTS` query for "products that appear in at least one order."
5. In a recursive CTE, what are the two parts and what causes it to stop (or fail to stop)?

## Hands-on exercise
On your schema:
1. Write a scalar subquery comparing each row to a table-wide aggregate (e.g. rows above the overall average).
2. Solve the same "rows above their *group's* average" problem with a correlated subquery.
3. Convert a two-level nested subquery into a readable CTE pipeline.
4. Write `NOT EXISTS` to find parents with no children, and verify it matches your Chapter 06 anti-join result.
5. Add a self-referencing `parent_id` column and write a **recursive CTE** that prints the hierarchy with indentation and a depth/level column.
