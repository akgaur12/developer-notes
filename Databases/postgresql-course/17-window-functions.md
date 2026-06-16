# Chapter 17 — Window Functions

## Where you are
`GROUP BY` (Chapter 07) collapses rows into summaries — you lose the individual rows. But often you want a per-row calculation that *also* references other rows: a running total, a rank, "each employee's salary vs their department average" while still showing every employee. **Window functions** do exactly that, and they replace a lot of slow correlated subqueries (Chapter 08) with fast, readable SQL. This is the single most powerful analytical feature in SQL.

> **The "why":** Aggregation answers "what's the total per group?" Window functions answer "for *this* row, what's its rank / running total / share / comparison *within* a group?" — without throwing the rows away. That "keep the rows AND compute across them" is the whole idea.

---

## 1. The core syntax: `OVER (...)`
A window function is any aggregate (or special function) followed by `OVER (...)`, which defines the **window** of rows it sees:
```sql
SELECT
  name, dept_id, salary,
  AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg   -- avg within each dept, shown on every row
FROM employees;
```
```
GROUP BY:                          WINDOW (OVER):
collapses to one row per dept      keeps every row, attaches the dept avg to each
  dept 3 | avg 82500                 Asha  | dept 3 | 90000 | dept_avg 82500
                                     Ravi  | dept 3 | 75000 | dept_avg 82500
```
The `OVER ()` clause has three parts, all optional:
```
OVER (
  PARTITION BY ...   split rows into groups (like GROUP BY, but rows are KEPT)
  ORDER BY ...       order within each partition (needed for ranking & running calcs)
  <frame>            which rows around the current one are included (ROWS/RANGE …)
)
```

## 2. Ranking functions
```sql
SELECT name, dept_id, salary,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
  RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense
FROM employees;
```
```
The difference shows up on ties (two equal salaries):
  salary  ROW_NUMBER  RANK  DENSE_RANK
  100     1           1     1
  100     2           1     1     ← RANK & DENSE_RANK tie at 1; ROW_NUMBER still increments
   90     3           3     2     ← RANK skips to 3; DENSE_RANK continues at 2
```
- `ROW_NUMBER()` — unique sequential number, arbitrary among ties.
- `RANK()` — ties share a rank, then it **skips** (1,1,3).
- `DENSE_RANK()` — ties share a rank, **no gap** (1,1,2).
- `NTILE(n)` — splits rows into `n` buckets (quartiles, percentiles).

**"Top-N per group" — the killer pattern** (e.g. top 3 earners per department):
```sql
SELECT * FROM (
  SELECT name, dept_id, salary,
         ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM employees
) ranked
WHERE rn <= 3;   -- filter AFTER the window (window funcs can't go in WHERE directly)
```

## 3. Offset functions: LAG and LEAD
Look at a previous or next row — perfect for period-over-period change:
```sql
SELECT month, revenue,
  LAG(revenue) OVER (ORDER BY month)  AS prev_month,
  revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change,
  LEAD(revenue) OVER (ORDER BY month) AS next_month
FROM monthly_revenue;
```
`LAG(col, n, default)` reaches `n` rows back. `FIRST_VALUE`, `LAST_VALUE`, and `NTH_VALUE` grab specific rows in the window.

## 4. Running totals and moving averages — the frame clause
`ORDER BY` inside `OVER` enables cumulative calculations; the **frame** controls exactly which rows are summed:
```sql
SELECT day, amount,
  SUM(amount) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
  AVG(amount) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)        AS moving_avg_7d
FROM daily_sales;
```
```
frame anatomy:
  UNBOUNDED PRECEDING  → start of partition
  N PRECEDING          → N rows before current
  CURRENT ROW
  N FOLLOWING
  UNBOUNDED FOLLOWING  → end of partition
```
> **ROWS vs RANGE (a real gotcha):** `ROWS` counts physical rows; `RANGE` groups by *value* (peers with the same `ORDER BY` value are treated as one). The **default frame** when you specify `ORDER BY` but no explicit frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — which can include *more rows than you expect* when there are ties. For precise running totals, specify `ROWS` explicitly.

## 5. WINDOW clause — name and reuse a window
When several functions share the same window, define it once:
```sql
SELECT name, salary,
  RANK()       OVER w AS rnk,
  DENSE_RANK() OVER w AS dense,
  AVG(salary)  OVER w AS avg_in_group
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

## 6. Window functions vs the alternatives
```
running total / rank / per-row share / period comparison   → window function (fast, clear)
"each row vs its group's aggregate"                          → window (replaces correlated subquery)
collapse to one summary row per group                        → GROUP BY
top-N per group                                              → ROW_NUMBER in a subquery, filter rn<=N
```
Window functions frequently turn an O(n²)-feeling correlated subquery (Chapter 08) into a single efficient pass — and they read far better.

> **Where window functions run (evaluation order):** they execute *after* `WHERE`, `GROUP BY`, and `HAVING`, but *before* the final `ORDER BY`/`LIMIT`. That's why you **can't** put a window function in `WHERE` or `HAVING` — wrap it in a subquery/CTE and filter the result (as in the top-N pattern).

---

## Summary
- A **window function** computes across a set of rows (`OVER (...)`) while **keeping every row** — unlike `GROUP BY`.
- `OVER` has **`PARTITION BY`** (groups, rows kept), **`ORDER BY`** (order within group, enables ranking/running calcs), and a **frame** (which surrounding rows count).
- **Ranking:** `ROW_NUMBER` (unique), `RANK` (ties + gaps), `DENSE_RANK` (ties, no gaps), `NTILE`. The **top-N-per-group** pattern uses `ROW_NUMBER` in a subquery filtered by `rn <= N`.
- **`LAG`/`LEAD`** compare to other rows (period-over-period); **frames** (`ROWS BETWEEN …`) build running totals and moving averages — specify **`ROWS`** to avoid `RANGE` tie surprises.
- Window functions run **after WHERE/GROUP BY/HAVING**, so filter their results in an outer query; they often replace slow correlated subqueries.

## Test your understanding
1. What's the fundamental difference between `GROUP BY AVG(salary)` and `AVG(salary) OVER (PARTITION BY dept_id)`?
2. Three rows have salaries 100, 100, 90. Give the `ROW_NUMBER`, `RANK`, and `DENSE_RANK` for each.
3. Write the "top 3 per group" pattern and explain why you can't just add `WHERE row_number <= 3`.
4. Why might a running total using the default frame (no explicit `ROWS`/`RANGE`) include unexpected rows?
5. How does `LAG(revenue) OVER (ORDER BY month)` help compute month-over-month change?

## Hands-on exercise
On a table with a category column and a numeric + date column:
1. Add a column showing each row's value alongside its category average using `AVG(...) OVER (PARTITION BY category)`.
2. Rank rows within each category by the numeric column using all three of `ROW_NUMBER`, `RANK`, `DENSE_RANK`; create a tie and observe the differences.
3. Implement top-2-per-category by wrapping `ROW_NUMBER` in a subquery.
4. Build a running total and a 7-row moving average with explicit `ROWS BETWEEN` frames.
5. Use `LAG` to compute the change from the previous period, and a `WINDOW` clause to share one window across several functions.
