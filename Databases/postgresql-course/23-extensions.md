# Chapter 23 — Extensions

## Where you are
Throughout this course you've used extensions in passing — `pg_trgm` for fuzzy search, `btree_gist` for exclusion constraints. Extensibility is Postgres's defining superpower (Chapter 01): you can bolt on entire new capabilities — geospatial, vector similarity, scheduling, cross-database queries — without leaving Postgres. This chapter surveys the extensions a professional should know, with `pg_stat_statements` first because it's the one you'll use constantly.

> **The "why":** A general-purpose database can't build in every specialized feature. Postgres instead exposes hooks for new types, functions, operators, and index methods, so the community ships them as installable extensions. The result: one database that can be a relational store, a document store, a geospatial engine, a vector database, and a job scheduler at once.

---

## 1. Installing and managing extensions
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;   -- enable in the current database
SELECT * FROM pg_available_extensions ORDER BY name; -- what's installable here
\dx                                                  -- what's currently installed
DROP EXTENSION pg_trgm;
```
Some extensions ship with Postgres ("contrib" modules — just `CREATE EXTENSION`); others must be installed at the OS level first (e.g. PostGIS, pgvector) and a few (like `pg_stat_statements`) must be loaded at startup via `shared_preload_libraries` then enabled.

## 2. pg_stat_statements — find your slow queries (use this first)
The most valuable operational extension. It records execution statistics for every query shape, so you know *what to optimize* instead of guessing. It pairs directly with `EXPLAIN` (Chapter 12) and tuning (Chapter 19).
```sql
-- one-time setup: add to shared_preload_libraries = 'pg_stat_statements', restart, then:
CREATE EXTENSION pg_stat_statements;

-- the killer query: your most time-consuming statements overall
SELECT calls,
       round(total_exec_time::numeric, 1)  AS total_ms,
       round(mean_exec_time::numeric, 2)   AS mean_ms,
       rows,
       query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```
> **The professional workflow:** `pg_stat_statements` tells you *which* queries cost the most cumulative time (often a fast query run a million times, not the obvious slow one). You then `EXPLAIN (ANALYZE, BUFFERS)` those and fix with indexes/rewrites. This "measure, then fix the top offender" loop is how real performance work is done — never optimize by hunch.

## 3. PostGIS — geospatial (the flagship extension)
Turns Postgres into a world-class geographic database — used for mapping, location search, routing, geofencing.
```sql
CREATE EXTENSION postgis;
ALTER TABLE stores ADD COLUMN geom geography(Point, 4326);   -- WGS84 lat/lon
UPDATE stores SET geom = ST_MakePoint(longitude, latitude)::geography;
CREATE INDEX idx_stores_geom ON stores USING GIST (geom);    -- spatial index (Chapter 13)

-- "stores within 5 km of this point", nearest-first:
SELECT name, ST_Distance(geom, ST_MakePoint(77.59, 12.97)::geography) AS meters
FROM stores
WHERE ST_DWithin(geom, ST_MakePoint(77.59, 12.97)::geography, 5000)
ORDER BY meters;
```
PostGIS adds geometry/geography types, hundreds of spatial functions, and GiST-indexed proximity/containment — an entire specialized database living inside Postgres.

## 4. pgvector — similarity search for AI/embeddings
Stores vector embeddings and finds nearest neighbors — the backbone of semantic search and Retrieval-Augmented Generation (RAG). It lets you keep embeddings *next to* your relational data, in one transactional store, instead of running a separate vector database.
```sql
CREATE EXTENSION vector;
CREATE TABLE documents (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    content   TEXT,
    embedding VECTOR(1536)        -- e.g. an embedding dimension from your model
);

-- approximate-nearest-neighbor index (HNSW is fast & high-recall):
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- find the 5 most semantically similar documents to a query embedding:
SELECT id, content
FROM documents
ORDER BY embedding <=> '[...query embedding...]'   -- <=> = cosine distance operator
LIMIT 5;
```
```
distance operators:  <-> L2/Euclidean   <#> negative inner product   <=> cosine distance
index types:         HNSW (fast, high recall, more memory)  •  IVFFlat (smaller, needs training)
```
Keeping vectors in Postgres means you can `JOIN` similarity results to metadata, filter by permissions, and stay transactional — often simpler than bolting on a dedicated vector store.

## 5. Other extensions worth knowing
```
pg_cron        run scheduled jobs (cron syntax) INSIDE Postgres — e.g. nightly
               REFRESH MATERIALIZED VIEW (Chapter 10) or partition maintenance (Chapter 18).
postgres_fdw   "Foreign Data Wrapper": query tables in ANOTHER Postgres server as if local
               (federation). Other FDWs reach MySQL, files, even APIs.
hstore         simple key→value string store (predates JSONB; JSONB is usually preferred now).
citext         case-insensitive text type (handy for emails/usernames without lower() everywhere).
uuid-ossp      extra UUID generators (modern Postgres has built-in gen_random_uuid()).
pgcrypto       hashing & encryption functions (Chapter 20).
pg_partman     automated partition creation & retention (Chapter 18).
pg_repack      rebuild bloated tables/indexes WITHOUT the exclusive lock of VACUUM FULL (Chapter 14).
TimescaleDB    (external) time-series superpowers built on partitioning — auto-partitioning,
               compression, continuous aggregates.
```

## 6. Cautions
- Extensions run with significant privileges — install only **trusted** ones, and know that some (PostGIS, TimescaleDB, pgvector) complicate **upgrades** and **managed-service compatibility** (not every host allows every extension; check first).
- A few require `shared_preload_libraries` and a **restart** to load (`pg_stat_statements`, `pg_cron`).
- Extensions are **per-database** (run `CREATE EXTENSION` in each database that needs it), and they're included/excluded by your backup strategy's scope — verify they're captured.

---

## Summary
- Extensions are Postgres's superpower: install new types, functions, operators, and index methods to make one database do specialized jobs. Manage with `CREATE EXTENSION`, `\dx`, `pg_available_extensions`.
- **`pg_stat_statements`** is the first one to enable — it ranks queries by cost so you optimize the real top offenders (measure → `EXPLAIN` → fix).
- **PostGIS** makes Postgres a full geospatial database (geometry types, spatial functions, GiST proximity search).
- **pgvector** adds vector similarity search (HNSW/IVFFlat, `<=>`/`<->` operators) for embeddings/RAG, keeping vectors transactional and joinable alongside relational data.
- Also know **pg_cron** (in-DB scheduling), **postgres_fdw** (federation), **citext**, **pgcrypto**, **pg_partman**, **pg_repack**, and **TimescaleDB**.
- Install only trusted extensions; some need a restart and complicate upgrades/managed hosting; they're per-database.

## Test your understanding
1. Why is `pg_stat_statements` the extension to reach for *first* when chasing performance, and how does it change your optimization workflow?
2. What does pgvector add, what does the `<=>` operator compute, and what's the advantage of keeping embeddings in Postgres rather than a separate vector DB?
3. Name an extension that requires `shared_preload_libraries` + restart.
4. What problem does `pg_repack` solve that `VACUUM FULL` causes (recall Chapter 14)?
5. You'd schedule a nightly materialized-view refresh entirely inside the database — which extension, and which earlier chapters does this connect?

## Hands-on exercise
1. List available extensions (`pg_available_extensions`) and installed ones (`\dx`).
2. Enable `pg_stat_statements` (set `shared_preload_libraries`, restart, `CREATE EXTENSION`), run a varied workload, then query the top 10 statements by `total_exec_time` and pick one to `EXPLAIN`.
3. If you can install it, enable `pgvector`, create a table with a `VECTOR` column, add an HNSW index, insert a few vectors, and run a `<=>` nearest-neighbor query.
4. Enable `citext`, make an email column case-insensitive, and prove `'A@x.com'` and `'a@x.com'` collide on a unique constraint.
5. (If available) Use `pg_cron` to schedule a `REFRESH MATERIALIZED VIEW` from Chapter 10 and confirm the job is registered.
