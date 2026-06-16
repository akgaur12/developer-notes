# Chapter 13 — Index Deep-Dive: Types & Strategy

## Where you are
You understand B-tree indexes (Chapter 11) and can read plans to verify usage (Chapter 12). Now you learn the **other index types** Postgres offers — each built for a specific kind of data and query — and the strategic patterns that separate adequate indexing from expert indexing. Choosing the right index *type* is what makes JSONB, full-text, geospatial, and array queries fast.

> **The "why":** A B-tree is a sorted structure — perfect for scalar values you compare with `=`, `<`, `>`. But "does this JSON contain that key?", "is this point near that point?", "does this text match these words?" aren't order questions. Different data shapes need differently-structured indexes.

---

## 1. The index type menu
```
B-tree   default. Equality, ranges, sorting, prefix LIKE. Use for almost all scalar columns.
Hash     equality ONLY (=). Rarely worth it over B-tree; B-tree handles = fine and does more.
GIN      "Generalized Inverted Index." For columns containing MULTIPLE values per row:
         arrays, JSONB, full-text (tsvector), trigrams. The workhorse for "contains" queries.
GiST     "Generalized Search Tree." For geometric/spatial data, ranges, nearest-neighbor,
         and overlap. The basis of PostGIS and range/exclusion constraints.
SP-GiST  space-partitioned GiST: non-balanced structures (quadtrees, tries) — e.g. IP, phone prefixes.
BRIN     "Block Range Index." Tiny index for HUGE tables where rows are naturally ordered on disk
         (e.g. append-only time-series). Stores min/max per block range. Minimal space, modest speed.
```

## 2. GIN — the "contains" index (you'll use this a lot)
Use GIN when one row holds many searchable values and you ask "does it contain X?"
```sql
-- arrays: which rows have the 'urgent' tag?
CREATE INDEX idx_tasks_tags ON tasks USING GIN (tags);
SELECT * FROM tasks WHERE tags @> ARRAY['urgent'];     -- @> = "contains"

-- JSONB: which rows have metadata.country = 'IN'?
CREATE INDEX idx_docs_meta ON docs USING GIN (metadata);
SELECT * FROM docs WHERE metadata @> '{"country":"IN"}';   -- containment uses the GIN index

-- a more targeted JSONB operator class indexes only the keys/values you query by path:
CREATE INDEX idx_docs_meta_path ON docs USING GIN (metadata jsonb_path_ops);
```
GIN is also the engine behind full-text search (Chapter 16) and trigram fuzzy/`LIKE '%x%'` search:
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_name_trgm ON customers USING GIN (name gin_trgm_ops);
SELECT * FROM customers WHERE name ILIKE '%shet%';   -- leading wildcard CAN now use an index!
```
GIN is fast to read but slower to update; for write-heavy cases tune `fastupdate` and `gin_pending_list_limit`.

## 3. GiST and BRIN — geometry, ranges, and giant ordered tables
**GiST** powers spatial/range/overlap queries and nearest-neighbor:
```sql
-- exclusion constraint: no two bookings for the same room may overlap in time
CREATE EXTENSION btree_gist;
ALTER TABLE bookings ADD CONSTRAINT no_overlap
  EXCLUDE USING GIST (room_id WITH =, during WITH &&);   -- && = ranges overlap
```
This is genuinely magical — the database *enforces* "no double-booking" structurally, something application code struggles to do race-free.

**BRIN** is for very large tables physically sorted on a column (classic: time-series append logs). It stores just the min/max value per block range, so it's *tiny* (kilobytes for gigabytes of data):
```sql
CREATE INDEX idx_events_time_brin ON events USING BRIN (created_at);
-- great when rows are inserted roughly in created_at order, so each block covers a tight time window
```
BRIN trades precision for size: if the data isn't physically ordered on the column, BRIN is useless. On well-ordered big data it's a spectacular space/speed win.

## 4. Strategic patterns

**Composite column order (revisited with intent).** Lead with equality columns, then the range/sort column:
```sql
-- WHERE customer_id = ? AND created_at > ? ORDER BY created_at
CREATE INDEX ON orders (customer_id, created_at);   -- equality first, then range/sort
```
This lets one index satisfy the filter *and* provide the sort order (no extra Sort node — verify in `EXPLAIN`).

**Partial indexes for skewed data.** If 99% of rows are `status='done'` and you only ever query the 1% that aren't:
```sql
CREATE INDEX ON jobs (created_at) WHERE status <> 'done';   -- index is ~1% the size, far cheaper to maintain
```

**Covering indexes for index-only scans.** Add the columns a query *returns* with `INCLUDE` so the table never needs reading:
```sql
CREATE INDEX ON orders (customer_id) INCLUDE (total, created_at);
```
Confirm you get an `Index Only Scan` in `EXPLAIN`. (Note: index-only scans also need the table's visibility map reasonably up to date — i.e. recently vacuumed, Chapter 14.)

**Expression indexes for computed predicates.**
```sql
CREATE INDEX ON users (lower(email));            -- WHERE lower(email)=...
CREATE INDEX ON orders ((created_at::date));     -- WHERE created_at::date = '2026-01-01'
```

**Multi-column via separate indexes + BitmapAnd.** If columns are queried in varied combinations, Postgres can combine *several single-column indexes* with a Bitmap And/Or. Sometimes two single-column indexes beat one rigid composite — let `EXPLAIN` decide.

## 5. Finding and fixing index problems
```sql
-- unused indexes (candidates to drop — they cost writes and space for nothing):
SELECT indexrelid::regclass AS index, idx_scan
FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- duplicate/overlapping indexes waste resources; review with \di+ and pg_index.

-- bloated indexes (after many updates/deletes) — rebuild without locking:
REINDEX INDEX CONCURRENTLY idx_name;
```
> **Pitfalls round-up:** (1) over-indexing cripples write throughput — every index is write tax; (2) redundant indexes (a `(a)` index is redundant if you have `(a, b)`); (3) leading-wildcard `LIKE` needs trigram/GIN, not B-tree; (4) functions/casts on the column need expression indexes; (5) low-cardinality columns (e.g. boolean with 50/50 split) rarely benefit from their own index — a partial index is usually better.

## 6. Decision guide
```
scalar =, <, >, ORDER BY, prefix LIKE        → B-tree (default)
"contains" on array / JSONB / full-text       → GIN
fuzzy match / LIKE '%mid%' / similarity        → GIN with pg_trgm
geometry, ranges, overlap, nearest-neighbor    → GiST  (PostGIS, exclusion constraints)
huge append-only table, physically ordered     → BRIN
equality-only, must save space (niche)         → Hash
```

---

## Summary
- Postgres offers **B-tree, Hash, GIN, GiST, SP-GiST, BRIN** — pick by data shape and query type, not habit.
- **GIN** indexes "contains" queries on arrays, JSONB, full-text, and (with `pg_trgm`) fuzzy/leading-wildcard `LIKE`.
- **GiST** powers geometry, ranges, nearest-neighbor, and **exclusion constraints** (e.g. no-overlap booking enforcement); **BRIN** is a tiny index for huge, physically-ordered tables.
- Strategic patterns: **equality-then-range** composite order, **partial** indexes for skew, **covering (`INCLUDE`)** for index-only scans, **expression** indexes for computed predicates, and letting **BitmapAnd** combine single-column indexes.
- Audit with `pg_stat_user_indexes`; drop unused/redundant indexes; rebuild bloated ones with `REINDEX CONCURRENTLY`. Always verify with `EXPLAIN`.

## Test your understanding
1. Why is a B-tree the wrong tool for `WHERE tags @> ARRAY['urgent']`, and what index type fits?
2. How can `pg_trgm` + GIN make `ILIKE '%shet%'` use an index when a B-tree can't?
3. What does an exclusion constraint with GiST let you enforce that's very hard in application code?
4. When is BRIN a brilliant choice, and when is it useless?
5. For `WHERE customer_id = ? AND created_at > ? ORDER BY created_at`, what composite index column order avoids an extra Sort, and why?

## Hands-on exercise
1. Create an `int[]` or `jsonb` column, populate it, add a **GIN** index, and prove with `EXPLAIN` that a `@>` containment query uses it.
2. Install `pg_trgm`, add a GIN trigram index on a text column, and show `ILIKE '%substr%'` switches from Seq Scan to an index scan.
3. Build an `EXCLUDE USING GIST` constraint preventing overlapping time ranges, then try to insert an overlapping row and watch it rejected.
4. On a large time-ordered table, compare the size (`\di+`) of a BRIN index vs a B-tree on the same timestamp column.
5. Create a covering `INCLUDE` index and confirm an `Index Only Scan` in `EXPLAIN`; then find any `idx_scan = 0` indexes and reason about dropping them.
