# Chapter 06 — JOINs

## Where you are
Your data is spread across related tables (Chapter 05). **JOINs** recombine it in queries — they are the single most important querying skill in relational databases, and the place beginners most often go wrong. Master this chapter and most of SQL opens up.

> **The "why":** Normalization deliberately *splits* data so each fact lives once. Joins are how you *reassemble* it on demand. The split (storage) and the join (query) are two halves of the same design philosophy.

---

## 1. The mental model: a join matches rows from two tables

A join pairs each row of table A with rows of table B that satisfy a **join condition** (usually `A.fk = B.pk`). The *type* of join decides what happens to rows that have **no match**.

```
employees                 departments
┌────┬───────┬─────────┐  ┌────┬─────────────┐
│ id │ name  │ dept_id │  │ id │ name        │
├────┼───────┼─────────┤  ├────┼─────────────┤
│ 1  │ Asha  │ 3       │  │ 1  │ Finance     │
│ 2  │ Ravi  │ 3       │  │ 3  │ Engineering │
│ 3  │ Meena │ NULL    │  │ 5  │ Marketing   │  ← no employees
└────┴───────┴─────────┘  └────┴─────────────┘
```

## 2. INNER JOIN — only matching rows
Returns rows that have a match on **both** sides.
```sql
SELECT e.name, d.name AS dept
FROM employees e
JOIN departments d ON e.dept_id = d.id;
```
Result: Asha/Engineering, Ravi/Engineering. **Meena drops** (her `dept_id` is NULL → no match) and **Marketing drops** (no employees). `JOIN` alone means `INNER JOIN`.

## 3. LEFT JOIN — keep all left rows
Returns every row from the **left** table; where there's no right match, right columns are `NULL`.
```sql
SELECT e.name, d.name AS dept
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;
```
Result includes **Meena / NULL** — she's kept even with no department. This is the workhorse for "list all X and their Y *if any*."

```
INNER JOIN: ●∩●   only the overlap
LEFT JOIN:  ◖●    all of left + matching right (unmatched right cols = NULL)
RIGHT JOIN: ●◗    all of right + matching left (just a flipped LEFT JOIN)
FULL JOIN:  ●∪●   everything from both; unmatched cells NULL on either side
```

## 4. RIGHT and FULL OUTER JOIN
`RIGHT JOIN` keeps all right-table rows (it's a `LEFT JOIN` written backwards — most people just use `LEFT`). `FULL OUTER JOIN` keeps unmatched rows from **both** sides:
```sql
SELECT e.name, d.name AS dept
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.id;
-- includes Meena (no dept) AND Marketing (no employees)
```

## 5. Finding rows that DON'T match (the anti-join)
A LEFT JOIN plus a NULL check finds "rows with no partner" — extremely useful:
```sql
-- departments with no employees:
SELECT d.name
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.id
WHERE e.id IS NULL;          -- the match failed → e.id is NULL
```

## 6. Joining more than two tables
Chain joins left to right; each `ON` connects to something already in the query:
```sql
SELECT c.name AS customer, p.name AS product, oi.quantity
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id   = o.id
JOIN products p     ON p.id          = oi.product_id
WHERE o.created_at >= now() - interval '30 days';
```
This is the everyday pattern for navigating a normalized schema (here: customer → orders → line items → product).

## 7. Self join — a table joined to itself
For hierarchies (employee → manager) where both are in one table:
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;   -- aliases distinguish the two roles
```
Aliases are mandatory here — you must tell the two copies apart.

## 8. CROSS JOIN — every combination (Cartesian product)
Pairs every left row with every right row (no condition). Rarely intentional — but deadly by accident:
```sql
SELECT * FROM sizes CROSS JOIN colors;   -- all size×color combinations
```
> **The #1 join pitfall: the accidental cross join.** If you forget the `ON` condition (or join on the wrong columns), you get an explosion — 10,000 × 10,000 = 100,000,000 rows. Symptoms: a query that "hangs" and row counts that are products, not sums, of table sizes. Always confirm every join has a correct `ON`.

## 9. ON vs WHERE on outer joins (a real gotcha)
On a `LEFT JOIN`, a condition in `ON` filters the *match*; the same condition in `WHERE` filters the *result* — and can silently turn your LEFT JOIN back into an INNER JOIN:
```sql
-- keeps all employees; only matches ACTIVE departments (Meena stays, dept may be NULL):
LEFT JOIN departments d ON e.dept_id = d.id AND d.active = true

-- looks similar but DROPS employees whose dept is NULL (because NULL = true is not TRUE):
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.active = true     -- ← turns the LEFT JOIN into an effective INNER JOIN
```
Rule of thumb: conditions about the **outer (kept) table** generally go in `WHERE`; conditions about the **optional table** that should *not* eliminate kept rows go in `ON`.

## 10. USING and NATURAL (minor conveniences)
```sql
JOIN departments USING (dept_id)   -- when both columns share the exact name; merges them
NATURAL JOIN departments           -- auto-joins on all same-named columns — AVOID; too implicit/fragile
```
Prefer explicit `ON`. `NATURAL JOIN` breaks subtly when someone adds a same-named column later.

---

## Summary
- A join pairs rows on a **condition**; the join *type* decides what happens to **unmatched** rows.
- **INNER** = only matches; **LEFT** = all left + matching right (NULLs otherwise); **RIGHT** = flipped LEFT; **FULL** = everything from both.
- **Anti-join** (LEFT JOIN + `WHERE right.id IS NULL`) finds rows with no partner.
- Chain joins to traverse a normalized schema; **self-join** with aliases handles hierarchies.
- **CROSS JOIN / a missing `ON`** causes a Cartesian explosion — the most common serious join bug.
- On outer joins, a filter in **`WHERE` can collapse a LEFT JOIN into an INNER JOIN**; put match-conditions in `ON`.

## Test your understanding
1. You want every customer listed, with their order count (0 for those who never ordered). Which join type, and why does `INNER JOIN` give the wrong answer here?
2. Write a query to find products that have never been ordered.
3. A LEFT JOIN report is mysteriously missing rows that have no match on the right. What single mistake most likely caused it?
4. Why are table aliases mandatory in a self-join?
5. You run a join and get far more rows than either table has. What probably happened?

## Hands-on exercise
Using your normalized multi-table schema:
1. Write an `INNER JOIN` across three tables (e.g. customer → orders → order_items).
2. Rewrite the top-level join as a `LEFT JOIN` so parents with no children still appear, and confirm the difference in row counts.
3. Write an **anti-join** to find parents with no children (e.g. customers with no orders).
4. Deliberately create an accidental cross join by removing an `ON`, observe the row count explosion, then fix it.
5. Add a self-referencing column (e.g. `manager_id` or `parent_id`) and write a self-join that pairs each row with its parent.
