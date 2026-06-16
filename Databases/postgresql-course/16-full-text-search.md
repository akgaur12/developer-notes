# Chapter 16 — Full-Text Search

## Where you are
`LIKE '%word%'` is fine for tiny data but is slow (can't use a normal index) and dumb (it doesn't know "running" relates to "run", or rank results by relevance). Postgres has a real **full-text search (FTS)** engine built in — good enough that many apps never need Elasticsearch. This chapter teaches the core model and when Postgres FTS is (and isn't) the right tool.

> **The "why":** Searching human language isn't substring matching. "Mice" should match a search for "mouse"; "the" and "a" shouldn't matter; results should be *ranked*. FTS preprocesses text into normalized tokens and matches/ranks against them.

---

## 1. The two core types
```
tsvector  a document processed into normalized, deduplicated lexemes (+ positions)
tsquery   a search query processed into terms combined with & (and) | (or) ! (not)
@@        the match operator: does this tsvector match this tsquery?
```
```sql
SELECT to_tsvector('english', 'The mice were running quickly');
-- → 'mice':2 'quick':5 'run':4   (stop-word "the/were" removed; "running"→"run", "quickly"→"quick")

SELECT to_tsvector('english', 'The mice were running') @@ to_tsquery('english', 'run');
-- → true   ("running" was normalized to "run", so it matches)
```
The magic is **normalization** by a *text-search configuration* (e.g. `'english'`): lowercasing, removing stop words, and **stemming** words to their root. That's why "running" matches "run".

## 2. A first searchable table
```sql
CREATE TABLE articles (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    body  TEXT NOT NULL
);

-- search across title+body:
SELECT id, title
FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'postgres & index');
```
This works but recomputes the `tsvector` for every row on every query — fine for hundreds of rows, far too slow for many thousands. We need to **store and index** the vector.

## 3. Making it fast: a generated tsvector column + GIN index
The modern, clean approach — a `GENERATED` column (Chapter 03) that Postgres keeps in sync, plus a GIN index (Chapter 13):
```sql
ALTER TABLE articles
  ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))
  ) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN (search);

-- now searches are fast and index-backed:
SELECT id, title
FROM articles
WHERE search @@ to_tsquery('english', 'postgres & index');
```
```
flow:  text  ──to_tsvector──▶ stored tsvector column ──GIN index──▶ fast @@ matching
       query ──to_tsquery──▶ tsquery ──────────────────┘
```

## 4. Query helpers — friendlier than raw `to_tsquery`
Raw `to_tsquery` requires precise `&`/`|` syntax and errors on bad input (dangerous with user input). Use these instead:
```
plainto_tsquery('english', 'postgres index')      → 'postgres' & 'index'   (ANDs the words)
phraseto_tsquery('english', 'query planner')       → 'query' <-> 'planner'  (adjacent, in order)
websearch_to_tsquery('english', 'postgres -mysql "query planner"')
        → web-style: quotes = phrase, - = exclude, or = OR. BEST for user-facing search boxes.
```
`websearch_to_tsquery` is the one to expose to end users — it never errors and understands familiar search syntax.

## 5. Ranking results by relevance
```sql
SELECT id, title,
       ts_rank(search, websearch_to_tsquery('english', 'postgres index')) AS rank
FROM articles
WHERE search @@ websearch_to_tsquery('english', 'postgres index')
ORDER BY rank DESC
LIMIT 20;
```
`ts_rank` (and `ts_rank_cd`, which factors in term proximity) score how well each document matches. You can also **weight** fields so a title hit outranks a body hit:
```sql
setweight(to_tsvector('english', title), 'A') ||
setweight(to_tsvector('english', body),  'B')   -- A > B > C > D in ranking influence
```

## 6. Highlighting matches
```sql
SELECT ts_headline('english', body, websearch_to_tsquery('english', 'postgres'))
FROM articles WHERE ...;
-- returns the snippet with matched terms wrapped in <b>…</b> (configurable) for search-result previews
```

## 7. Fuzzy/typo matching vs full-text (know the difference)
FTS matches *words* (after stemming) — it won't catch typos like "postgers". For typo tolerance, autocomplete, and `LIKE '%mid%'`, use **trigram** matching (`pg_trgm`, Chapter 13):
```sql
CREATE EXTENSION pg_trgm;
SELECT title, similarity(title, 'postgers') AS sim
FROM articles WHERE title % 'postgers' ORDER BY sim DESC;   -- % = similar-enough; fuzzy match
```
Often the best UX combines both: trigram for typo-tolerant prefix/autocomplete, FTS for relevance-ranked document search.

## 8. When Postgres FTS is enough — and when it isn't
```
Postgres FTS is GREAT when:
  • search is one feature of an app already on Postgres (no extra system to run)
  • document counts up to the low millions
  • you want search transactionally consistent with your data (no sync lag)

Consider a dedicated engine (Elasticsearch/OpenSearch, Typesense, or pgvector for semantic) when:
  • very large corpora with heavy search traffic and complex relevance tuning
  • advanced features: faceting at scale, fuzzy+linguistic blends, distributed sharding
  • semantic / vector similarity search (embeddings) — though pgvector (Chapter 23) keeps that in Postgres too
```
The pragmatic default: **start with Postgres FTS.** You avoid a whole second system, its sync pipeline, and its consistency headaches. Reach for a dedicated engine only when you've outgrown it for real, measured reasons.

---

## Summary
- FTS uses **`tsvector`** (normalized document) and **`tsquery`** (parsed query), matched with **`@@`**; a **text-search configuration** lowercases, strips stop words, and **stems** (so "running" matches "run").
- For speed, store a **`GENERATED ... STORED tsvector` column** and add a **GIN index**.
- Build queries with **`websearch_to_tsquery`** for user input (safe, familiar syntax); rank with **`ts_rank`**, weight fields with **`setweight`**, and highlight with **`ts_headline`**.
- **Trigram (`pg_trgm`)** handles typos/autocomplete/`%mid%` that FTS can't; combine them for great UX.
- **Default to Postgres FTS**; adopt a dedicated search engine only at large scale or for specialized needs.

## Test your understanding
1. Why does a search for "run" match a document containing "running," and what step makes that happen?
2. Why is computing `to_tsvector(...)` in the `WHERE` clause on every query a scaling problem, and what's the fix?
3. Which query-building function should you expose to end users and why?
4. What does `setweight` accomplish, and `ts_rank`?
5. A user types "databse" (typo) and gets no FTS results. Which Postgres feature addresses this, and how does it differ from FTS?

## Hands-on exercise
1. Create a table of articles (title + body) and insert a dozen rows of varied text.
2. Add a `GENERATED ... STORED` `tsvector` column combining title and body with `setweight` (title = A), and a GIN index.
3. Run a `websearch_to_tsquery` search, ordered by `ts_rank`, and inspect `EXPLAIN` to confirm the GIN index is used.
4. Add `ts_headline` to produce highlighted snippets.
5. Install `pg_trgm`, add a trigram index, and build a typo-tolerant autocomplete query with `similarity()`; compare its results to FTS for a misspelled term.
