# Chapter 15 — JSON & JSONB

## Where you are
You've mastered the relational core. Postgres also speaks **JSON** fluently — letting you store flexible, semi-structured data inside a relational database. Used well, this often removes any need for a separate document database (MongoDB et al.). Used badly, it throws away everything relational integrity gives you. This chapter teaches the discipline.

> **The "why":** Some data is genuinely schema-flexible — third-party API payloads, per-product custom attributes, event metadata. Forcing it into rigid columns is painful; scattering it into a separate NoSQL store loses joins and transactions. JSONB lets you keep flexible data *and* your relational guarantees, in one place, in one transaction.

---

## 1. `json` vs `jsonb` — always choose `jsonb`
```
json   stores the exact text you inserted (whitespace, key order, duplicate keys preserved).
       No indexing into the structure. Parsed on every access. Rarely what you want.
jsonb  parsed into a binary form on insert: no whitespace/dup keys, keys reordered,
       INDEXABLE, fast to query into. Slightly slower to write, much faster to read/query.
```
**Default to `jsonb`.** Use `json` only in the rare case you must preserve exact input formatting.

## 2. Creating and inserting
```sql
CREATE TABLE products (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL,
    attrs JSONB NOT NULL DEFAULT '{}'
);

INSERT INTO products (name, attrs) VALUES
  ('Laptop', '{"brand":"Acme","ram_gb":16,"ports":["usb-c","hdmi"],"specs":{"cpu":"i7"}}');
```

## 3. Accessing data — the operator cheat sheet
```
->    get field/element as JSONB        attrs -> 'brand'        → "Acme" (jsonb)
->>   get field/element as TEXT         attrs ->> 'brand'       → Acme   (text)
#>    get nested by path as JSONB       attrs #> '{specs,cpu}'  → "i7"   (jsonb)
#>>   get nested by path as TEXT        attrs #>> '{specs,cpu}' → i7     (text)
```
> **The classic gotcha:** `->` returns JSONB (a `"Acme"` *with quotes*); `->>` returns plain text (`Acme`). Use `->>` when comparing to a string or extracting a value for the app; use `->` when you need to keep digging into nested JSON. Mixing them up causes "why does my `WHERE` match nothing?" bugs.
```sql
SELECT name FROM products WHERE attrs ->> 'brand' = 'Acme';        -- correct (text vs text)
SELECT attrs #>> '{specs,cpu}' AS cpu FROM products;              -- nested extraction
SELECT name, (attrs ->> 'ram_gb')::int AS ram FROM products       -- cast for numeric comparison
WHERE (attrs ->> 'ram_gb')::int >= 16;
```

## 4. Containment and existence — the indexable operators
These are the operators a GIN index accelerates (Chapter 13):
```
@>   "contains"            attrs @> '{"brand":"Acme"}'         row's JSON contains this
<@   "contained by"
?    key exists            attrs ? 'brand'
?|   any of these keys     attrs ?| array['brand','model']
?&   all of these keys     attrs ?& array['brand','ram_gb']
```
```sql
CREATE INDEX idx_products_attrs ON products USING GIN (attrs);
SELECT name FROM products WHERE attrs @> '{"brand":"Acme"}';   -- uses the GIN index
```
`@>` containment is the bread-and-butter "find JSON matching this shape" query and is fully indexable — prefer it over `->>` equality when you can, because `->>` equality needs an *expression* index on that specific path while `@>` works against one general GIN index.

## 5. JSONPath (SQL/JSON) — querying like XPath
Modern Postgres supports the SQL standard JSONPath for richer queries:
```sql
SELECT name FROM products
WHERE jsonb_path_exists(attrs, '$.ports[*] ? (@ == "hdmi")');     -- has an "hdmi" port
SELECT jsonb_path_query(attrs, '$.specs.cpu') FROM products;       -- extract by path
```

## 6. Modifying JSONB
```sql
UPDATE products SET attrs = attrs || '{"warranty_months":24}'    -- merge/add a key (|| operator)
WHERE id = 1;

UPDATE products SET attrs = jsonb_set(attrs, '{specs,cpu}', '"i9"')  -- set a nested value
WHERE id = 1;

UPDATE products SET attrs = attrs - 'warranty_months'            -- remove a key (- operator)
WHERE id = 1;
```
Note each update rewrites the whole JSONB value (MVCC: a new row version) — fine for occasional changes, costly if you hammer one giant JSON blob constantly.

## 7. Bridging JSON and relational — set-returning functions
Turn JSON into rows (and back), which is how you join JSON against real tables:
```sql
-- expand a JSON array into rows:
SELECT id, port
FROM products, jsonb_array_elements_text(attrs -> 'ports') AS port;

-- expand an object into key/value rows:
SELECT key, value FROM jsonb_each_text('{"a":1,"b":2}');

-- build JSON from relational rows (great for API responses):
SELECT jsonb_build_object('id', id, 'name', name, 'attrs', attrs) FROM products;
SELECT jsonb_agg(jsonb_build_object('id', id, 'name', name)) FROM products;  -- array of objects
```

## 8. When to use JSONB — and when NOT to
```
GOOD uses of JSONB:
  • genuinely variable attributes (per-product specs that differ by category)
  • opaque third-party payloads you store and occasionally inspect
  • sparse optional fields that would be a sea of NULL columns
  • event/audit metadata

BAD uses (use real columns instead):
  • core, queried-on-every-request fields (put them in typed columns with constraints & indexes)
  • relationships (use foreign keys, not embedded ids)
  • anything you need to enforce types/uniqueness/foreign keys on
  • data you frequently update field-by-field at high volume
```
> **The discipline:** JSONB is a *complement* to columns, not a replacement. Promote any field you filter, join, sort, or constrain on into a real typed column (it can even be a `GENERATED` column extracted from the JSONB). Reserve JSONB for the truly variable parts. This gives you NoSQL flexibility *without* losing relational integrity — the best of both worlds, in one transactional store.

---

## Summary
- Use **`jsonb`** (binary, indexable, deduped) over `json` (raw text) almost always.
- Access with **`->`/`#>`** (returns JSONB) vs **`->>`/`#>>`** (returns text) — mixing them up is the #1 JSONB bug; cast text for numeric comparisons.
- **`@>` containment** and key-existence operators are **GIN-indexable** — prefer `@>` for shape-matching queries.
- **JSONPath** (`jsonb_path_query`/`_exists`) gives XPath-like power; **`||`, `jsonb_set`, `-`** modify JSONB; set-returning functions (`jsonb_array_elements`, `jsonb_each`, `jsonb_build_object`, `jsonb_agg`) bridge JSON and relational.
- **Discipline:** keep core/queried/related fields as typed columns; reserve JSONB for genuinely variable data. JSONB complements the relational model — it doesn't replace it.

## Test your understanding
1. Why prefer `jsonb` over `json`? Name one case where `json` is actually correct.
2. `WHERE attrs -> 'brand' = 'Acme'` returns nothing but you know the brand is Acme. What's wrong?
3. Which JSONB operator is GIN-indexable for "find rows whose JSON matches this shape," and why prefer it over `->>` equality?
4. You have a `status` field you filter on in every query but it currently lives inside a JSONB column. What should you do and why?
5. How would you turn a JSON array stored in a column into one row per element so you can join it to another table?

## Hands-on exercise
1. Create a table with a `jsonb` column and insert several rows with nested objects and arrays.
2. Write queries using `->`, `->>`, `#>>`, and a cast to compare a numeric JSON field.
3. Add a GIN index and prove with `EXPLAIN` that an `@>` containment query uses it.
4. Use `jsonb_set` to update a nested value and `||` to add a key; verify with a read.
5. Expand a JSON array with `jsonb_array_elements_text` and join the result to another table; then build an array-of-objects API response with `jsonb_agg(jsonb_build_object(...))`.
6. Identify one field in your JSON that *should* be a real column, and migrate it (optionally as a `GENERATED` column extracted from the JSONB).
