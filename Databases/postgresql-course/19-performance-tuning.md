# Chapter 19 — Performance Tuning & Connection Pooling

## Where you are
You can write fast queries (Chapters 11–13), read plans (Chapter 12), and keep the engine healthy (Chapter 14). Now we zoom out to the **whole instance**: the memory and planner settings that govern performance, and — critically — **connection pooling**, which is the difference between a Postgres that scales and one that falls over under load. This chapter ties together threads from Chapters 01, 09, 12, and 14.

> **The "why" (recall Chapter 01):** Postgres uses a **process per connection**. Each backend costs memory and scheduling overhead. An app that opens hundreds or thousands of direct connections doesn't go faster — it grinds to a halt. Understanding this one fact prevents the most common production scaling failure.

---

## 1. The memory settings that matter most
These live in `postgresql.conf` (or are set via `ALTER SYSTEM ... ; SELECT pg_reload_conf();`). Defaults are conservative — tuned for tiny machines — so a real server is almost always under-configured out of the box.
```
shared_buffers         Postgres's own cache of data pages. Start ~25% of RAM.
                       (The OS file cache also caches data, so not 100%.)
effective_cache_size   NOT an allocation — the planner's ESTIMATE of total cache (PG + OS).
                       Set ~50–75% of RAM so the planner favors indexes (Chapter 12).
work_mem               memory PER SORT/HASH OPERATION, per query. Raises in-memory sorts/joins
                       above spilling to disk — but it's multiplied by concurrent operations!
maintenance_work_mem   memory for VACUUM, CREATE INDEX, etc. Set generously (e.g. 256MB–1GB);
                       speeds up index builds and vacuum.
```
> **The `work_mem` trap:** `work_mem` is allocated *per sort/hash node, per concurrent query* — not globally. A generous 256MB `work_mem` with a query using 4 sorts across 100 connections can demand 100 GB and trigger the OOM killer. Size it as: roughly `(RAM available for queries) / (max_connections × expected sorts per query)`. Raise it selectively per-session for known heavy analytical queries rather than globally.

## 2. Planner settings (correctness-neutral, plan-shaping)
```
random_page_cost       cost of a random page read. Default 4.0 assumes spinning disks.
                       On SSD/NVMe set ~1.1 — otherwise the planner under-uses indexes (Chapter 12).
effective_io_concurrency  number of concurrent I/O ops the storage can handle (raise on SSD).
default_statistics_target  sampling detail for ANALYZE (default 100). Raise for columns with
                       skewed distributions where the planner mis-estimates.
```
The single highest-impact planner fix on modern hardware is lowering `random_page_cost` to match SSDs — it's the most common reason "Postgres won't use my index."

## 3. WAL and checkpoint settings (write throughput & durability)
```
max_wal_size / min_wal_size   how much WAL accumulates between checkpoints. Larger = fewer,
                               smoother checkpoints (less write-spike stalling) at the cost of
                               longer crash recovery.
checkpoint_completion_target   spread checkpoint writes over this fraction of the interval
                               (default 0.9) to avoid I/O spikes.
synchronous_commit             on (default) = wait for WAL flush before COMMIT returns (durable).
                               Setting it 'off' boosts throughput but risks losing the last
                               fraction of a second of committed transactions on a crash.
                               A deliberate, rarely-correct trade-off — know it exists.
wal_compression                compress full-page WAL images; often a net win.
```

## 4. Connection pooling — the scaling linchpin
Because each connection is a process, the right number of *active* connections is surprisingly small — often roughly `(CPU cores × 2–4) + effective_spindle_count`, not "hundreds." Beyond that, connections fight over CPU and memory and **everything slows down**.

The solution is a **connection pooler** sitting between your app and Postgres: the app opens many cheap connections to the pooler, which multiplexes them onto a small set of real Postgres connections.
```
   many app connections                 pooler            few real PG connections
   ┌──────────────┐                  ┌─────────┐            ┌───────────────┐
   │ app worker 1 │──┐               │         │── conn 1 ──│               │
   │ app worker 2 │──┼──(500 conns)─▶│ PgBouncer│── conn 2 ──│  PostgreSQL   │
   │     ...      │──┤               │         │── ...    ──│  (e.g. 25)    │
   │ app worker N │──┘               └─────────┘            └───────────────┘
```

**PgBouncer** is the standard. Its key setting is the **pool mode**:
```
session     a server connection is tied to a client for the whole session.
            Safe (all features work) but least efficient — limited multiplexing.
transaction a server connection is assigned only for the duration of a transaction,
            then returned to the pool. MUCH higher concurrency. The usual choice.
            ⚠ but: features tied to a session (some prepared statements, SET, advisory
            locks, LISTEN/NOTIFY, session-level temp tables) can break — your app/driver
            must be compatible.
statement   returns the connection after each statement. Most aggressive; forbids
            multi-statement transactions. Niche.
```
> **Transaction pooling is what unlocks high concurrency**, but it changes the contract: anything that assumes a persistent session may misbehave. Configure your driver accordingly (e.g. disable client-side prepared-statement caching or use protocol-level support). This is the most important operational decision when scaling a Postgres app. (PgCat and the built-in pooling in some managed services are alternatives; PgBouncer remains the reference.)

## 5. A pragmatic tuning workflow
```
1. Right-size memory: shared_buffers ~25% RAM, effective_cache_size ~50–75% RAM,
   maintenance_work_mem generous, work_mem conservative-but-not-tiny (then tune per-query).
2. Match the planner to hardware: random_page_cost ~1.1 on SSD.
3. Put a connection pooler (PgBouncer, transaction mode) in front; cap real PG connections low.
4. Find the actual slow queries with pg_stat_statements (Chapter 23), EXPLAIN them (Chapter 12),
   and fix with indexes/rewrites BEFORE touching more knobs.
5. Keep autovacuum healthy (Chapter 14) — no amount of tuning saves a bloated, never-vacuumed table.
```
> **The most important principle:** the biggest wins are almost always **a missing index or a bad query**, not a config tweak. Use `pg_stat_statements` + `EXPLAIN` to find the real culprit first. Config tuning matters, but it's the second move, not the first. (Tools like PGTune give sane starting config for your RAM/workload — a fine baseline, not a substitute for query work.)

---

## Summary
- Postgres ships with **conservative defaults**; size **`shared_buffers`** (~25% RAM) and **`effective_cache_size`** (~50–75% RAM, planner estimate only), keep **`maintenance_work_mem`** generous.
- **`work_mem` is per-operation, per-query** — over-setting it globally invites OOM; tune it per-session for heavy queries.
- Set **`random_page_cost` ~1.1 on SSD** so the planner uses indexes; raise statistics targets for skewed columns.
- WAL/checkpoint settings trade write-spike smoothness and durability (`synchronous_commit`) against throughput and recovery time.
- **Connection pooling (PgBouncer, transaction mode) is the scaling linchpin** — Postgres's process-per-connection model means few real connections are best; pooling multiplexes many app connections onto few server ones, but transaction mode changes session-feature behavior.
- **Fix slow queries/missing indexes first** (`pg_stat_statements` + `EXPLAIN`); config tuning is the second move.

## Test your understanding
1. Why is `effective_cache_size` not a memory allocation, and how does it influence the planner?
2. Explain the `work_mem` trap: why can a "reasonable" value cause an out-of-memory event?
3. On an SSD server, which single planner setting most often fixes "Postgres ignores my index," and to what value?
4. Why does opening thousands of direct connections *slow down* Postgres rather than speed it up?
5. What does PgBouncer's transaction pooling gain you, and what category of features can it break?

## Hands-on exercise
1. Inspect current settings: `SHOW shared_buffers; SHOW work_mem; SHOW random_page_cost; SHOW effective_cache_size;`.
2. On a query with a big sort, set a higher `work_mem` for the session (`SET work_mem='256MB';`) and use `EXPLAIN (ANALYZE, BUFFERS)` to see an external-merge (disk) sort become an in-memory sort.
3. Lower `random_page_cost` (`SET random_page_cost = 1.1;`) and find a query whose plan flips from Seq Scan to Index Scan.
4. (Conceptual/setup) Stand up PgBouncer in front of your instance in transaction mode; point a client at it and confirm connectivity.
5. Write down, for a hypothetical 16 GB / 8-core SSD server, your starting values for the five settings in this chapter and a one-line justification for each.
