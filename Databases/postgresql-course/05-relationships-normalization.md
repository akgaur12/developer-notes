# Chapter 05 — Relationships & Normalization

## Where you are
You can manipulate single tables. Real systems have *many* tables that reference each other. This chapter teaches how to model relationships correctly and how **normalization** gives you a principled way to decide what goes in which table — and, just as importantly, when to break the rules on purpose.

> **The "why":** Good schema design prevents whole categories of bugs before they exist. Redundant data drifts out of sync (the same customer name spelled two ways in two places); a normalized schema stores each fact exactly once, so it can only be wrong in one place.

---

## 1. The three relationship shapes

```
ONE-TO-MANY  (the most common)
   one department  ──< many employees
   Implementation: the "many" side carries a foreign key to the "one" side.
   employees.dept_id → departments.id

ONE-TO-ONE
   one user  ──  one user_profile
   Implementation: a FK that is ALSO unique (or a shared primary key).

MANY-TO-MANY
   many students  >──< many courses
   Implementation: a JUNCTION (join/bridge) table holding two FKs.
```

One-to-many in practice:
```sql
CREATE TABLE departments (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);
CREATE TABLE employees (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    TEXT NOT NULL,
    dept_id BIGINT NOT NULL REFERENCES departments(id)   -- the FK lives on the "many" side
);
```

Many-to-many needs a junction table — there is no other correct way:
```sql
CREATE TABLE students ( id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, name TEXT NOT NULL );
CREATE TABLE courses  ( id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, title TEXT NOT NULL );

CREATE TABLE enrollment (                       -- the junction table
    student_id BIGINT NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id  BIGINT NOT NULL REFERENCES courses(id)  ON DELETE CASCADE,
    enrolled_on DATE NOT NULL DEFAULT CURRENT_DATE,     -- junction tables can carry their own data!
    PRIMARY KEY (student_id, course_id)                 -- prevents duplicate enrollments
);
```
A junction table is a first-class table — it can hold attributes of the *relationship itself* (here, when the enrollment happened).

## 2. Normalization — storing each fact once

Normalization is a sequence of "normal forms," each removing a kind of redundancy. You'll use the first three; the rest are rare in practice.

**The unnormalized starting point (a classic mess):**
```
orders
┌────┬───────────┬──────────────────────┬───────────────┬──────────┐
│ id │ customer  │ items                │ customer_city │ total    │
├────┼───────────┼──────────────────────┼───────────────┼──────────┤
│ 1  │ Asha      │ "Pen, Notebook, Pen" │ Bangalore     │ 230.00   │
└────┴───────────┴──────────────────────┴───────────────┴──────────┘
```
Problems: items crammed into one cell; customer city repeated on every order (drift risk); no clean way to query "all orders containing a Pen."

**First Normal Form (1NF): atomic values, no repeating groups.**
Each cell holds a single value; multi-valued data moves to its own rows.

**Second Normal Form (2NF): no partial dependency on part of a composite key.**
Every non-key column must depend on the *whole* primary key, not just part of it. (Only relevant when you have composite keys.)

**Third Normal Form (3NF): no transitive dependencies.**
Non-key columns must depend on the key *only* — not on another non-key column. `customer_city` depends on the *customer*, not the *order*, so it belongs in a `customers` table.

**The normalized result:**
```sql
CREATE TABLE customers (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    city TEXT
);
CREATE TABLE products (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0)
);
CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE order_items (                 -- junction between orders and products, with quantity
    order_id   BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)
);
```
Now a customer's city is stored **once**. A product's price is stored **once**. The order total is *derived* by summing items — never stored redundantly (until you choose to, deliberately — see below).

> **A subtle but important point about historical data:** if `order_items` should remember the price *at the time of sale*, store a `unit_price` column on `order_items` too — because the product's current price may change later. This isn't a normalization violation; the price-at-sale is genuinely a fact about the *order item*, distinct from the product's current price. Recognizing which facts belong where is the real art.

## 3. The mental rule of thumb
Ask of every column: **"What does this fact actually describe?"** Put it in the table for that thing.
- A city describes a *customer* → `customers.city`.
- A quantity describes an *order line* → `order_items.quantity`.
- A current price describes a *product* → `products.price`.
- A price *paid* describes an *order line* → `order_items.unit_price`.

## 4. When to denormalize (on purpose)
Normalization optimizes for **correctness and write integrity**. Sometimes you trade a little of that for **read speed**:
- **Caching a computed value** (e.g. storing `orders.total` so you don't re-sum items on every read of a high-traffic page).
- **Reporting/analytics tables** that pre-join and flatten data.
- **Materialized views** (Chapter 10) — a managed form of denormalization.

The rule: **normalize first; denormalize later, only with a measured reason, and only when you have a plan to keep the redundant copy in sync** (a trigger, a job, or a materialized view refresh). Premature denormalization is how data drifts apart. Note that JSONB columns (Chapter 15) are a controlled, modern way to keep some flexible/denormalized data inside an otherwise relational design.

## 5. Reading a schema as a diagram (ERD)
An Entity-Relationship Diagram makes structure visible:
```
   customers ──< orders ──< order_items >── products
       1        many  1      many    many   1

   ──<  means "one to many" (crow's foot on the many side)
   >──< means "many to many" (resolved by the junction table in the middle)
```
`psql` helpers: `\d tablename` shows a table's foreign keys and what references it; `\dt` lists tables.

---

## Summary
- Three shapes: **one-to-many** (FK on the many side), **one-to-one** (unique FK), **many-to-many** (junction table, which can carry its own attributes).
- **Normalization** (1NF atomic, 2NF no partial-key dependency, 3NF no transitive dependency) ensures each fact is stored **once**, preventing drift.
- Decide where a column lives by asking **what the fact describes**.
- Historical values (price-at-sale) legitimately live on the line item, not derived from the current product.
- **Denormalize only deliberately**, for measured read performance, with a sync plan.

## Test your understanding
1. You're modeling blog posts and tags (a post has many tags, a tag applies to many posts). What tables do you need and where do the foreign keys go?
2. A `bookings` table has columns `room_number, room_floor, guest_name`. Which column violates 3NF and where should it go?
3. Why should `order_items` often store a `unit_price` even though `products` already has `price`? Is that a normalization violation?
4. Give one legitimate reason to store a redundant `orders.total`, and the obligation that comes with doing so.
5. In a junction table `enrollment(student_id, course_id)`, why is the composite primary key valuable?

## Hands-on exercise
Take a domain with a genuine many-to-many relationship (students/courses, posts/tags, actors/movies):
1. Model it with at least three tables including a junction table that carries **at least one attribute of the relationship** (e.g. `enrolled_on`, `role`).
2. Add appropriate foreign keys with justified `ON DELETE` actions and a composite primary key on the junction.
3. Insert data: two parents on each side and several relationship rows.
4. Identify one column in your design that could be denormalized for read speed, and write (in a comment) how you'd keep it in sync.
5. Run `\d` on each table and trace the relationships in the output.
