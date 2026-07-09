# Chapter 16: Best Practices

Fifteen chapters ago, QuickCart's Redis footprint was a single `SET`/`GET` pair in a terminal window. By now it's a `session:{userId}` hash with a TTL, a `product:{sku}` cache, a `cart:{userId}` hash, a `leaderboard:daily` sorted set, a `ratelimit:{userId}:{endpoint}` counter, an `orders:events` stream, a `notifications:{userId}` Pub/Sub channel, a `stores:locations` geo index — backed by RDB snapshots and AOF, guarded by `maxmemory` and an eviction policy, made atomic with `MULTI`/`EXEC` and Lua, wrapped in a pooled and pipelined client, replicated and Sentinel-managed, sharded across a Cluster, tuned with `redis-benchmark`, watched with Prometheus, and locked down with ACLs and TLS. Every one of those decisions made sense in the chapter that introduced it. What's missing is the view from above: all of it, in one place, organized by theme instead of by chapter number, so a senior engineer can run a design or a deployment against it in twenty minutes instead of re-reading fifteen chapters.

This chapter is that reference. Treat it the way QuickCart's platform team treats it in this chapter's scenario: as a checklist you run *before* you sign off on a schema, a config change, or a new Redis-backed feature going to production — not as a one-time read.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Recite a concise, defensible checklist of Redis best practices spanning data modeling, persistence, memory management, atomicity, client usage, high availability, performance, observability, security, and operations.
- Explain the reasoning behind each practice well enough to adapt it when your workload doesn't match the textbook case.
- Run a structured pre-production review of a Redis deployment or a new feature built on Redis, and rank the gaps you find by severity.
- Recognize the handful of decisions (key schema, hash tags, persistence strategy, Cluster topology) that are expensive to change once real traffic and data volume accumulate.
- Distinguish practices that are "nice to have" from the small set that are load-bearing for correctness, availability, or data loss.
- Audit a real or hypothetical Redis deployment against this chapter's consolidated checklist and produce a written remediation list.

---

## Prerequisites

This is a **synthesis** chapter. It assumes you have completed Chapters 1 through 15 and have working knowledge of everything it references — it does not re-teach any technique, it distills and cross-links what you've already learned into one operational reference. The major theme areas it draws from:

- **[Chapter 2: Core Concepts](./02-core-concepts.md)** — the flat keyspace, `object-type:id` key naming, and why key design is Redis's version of schema design.
- **[Chapters 4–6: Strings, Lists & Hashes; Sets, Sorted Sets & Bitmaps; Streams & Pub/Sub](./04-strings-lists-and-hashes.md)** — choosing the right native data type for a problem.
- **[Chapter 7: Persistence — RDB & AOF](./07-persistence-rdb-and-aof.md)** — snapshotting, AOF rewriting, and durability trade-offs.
- **[Chapter 8: Transactions & Lua Scripting](./08-transactions-and-lua-scripting.md)** — `MULTI`/`EXEC`/`WATCH`, optimistic locking, `EVAL`/`EVALSHA`, Redis Functions.
- **[Chapter 9: Expiration, Eviction & Memory Management](./09-expiration-eviction-and-memory-management.md)** — TTLs, `maxmemory`, and eviction policy selection.
- **[Chapter 10: Client Libraries & Connection Management](./10-client-libraries-and-connection-management.md)** — connection pooling and pipelining.
- **[Chapter 11: Replication & High Availability](./11-replication-and-high-availability.md)** and **[Chapter 12: Redis Cluster & Sharding](./12-redis-cluster-and-sharding.md)** — Sentinel quorum, hash slots, hash tags, and topology design.
- **[Chapter 13: Performance Tuning & Benchmarking](./13-performance-tuning-and-benchmarking.md)** and **[Chapter 14: Monitoring & Observability](./14-monitoring-and-observability.md)** — `redis-benchmark`, the slow log, and what to watch in production.
- **[Chapter 15: Security](./15-security.md)** — ACLs, `requirepass`, TLS, and network hardening.

If any of these feel unfamiliar, a quick re-read before continuing will make this chapter much more useful — every bullet below has a full chapter behind it if you need the complete explanation.

---

## 1. Data Modeling & Key Design

*(Builds on Chapter 2: Core Concepts and Chapters 4–6: native data types)*

- **Adopt a single, enforced key naming convention across the whole team**: `object-type:id[:sub-id]`, e.g. `product:SKU-1001`, `cart:42`, `session:u1001`. Redis enforces none of this for you — a colliding or inconsistent key name is a silent bug, not an error.
- **Use colons as the separator, consistently**, and document the convention somewhere every engineer will actually read it (a `KEYS.md` or wiki page) — a mixed `product_1001` / `product:1001` / `Product:1001` codebase makes `SCAN`-based auditing and `redis-cli --scan --pattern` operations unreliable.
- **Choose the narrowest data type that fits the access pattern, not the one that's easiest to serialize.** A cart is a hash (`sku → qty`), not a JSON blob in a string — model it so Redis can mutate one field atomically (`HINCRBY`) instead of you doing a client-side read-modify-write on a whole blob.
- **Use hash tags (`{...}`) deliberately on every Cluster deployment where multi-key operations, transactions, or Lua scripts touch more than one key.** `cart:{42}:items` and `cart:{42}:meta` hash to the same slot because only the substring inside `{}` is hashed — without this, a Cluster-mode `MULTI`/`EXEC` or Lua script spanning both keys fails with a `CROSSSLOT` error.
- **Never let a single key grow unbounded.** A list, set, hash, or sorted set with millions of members is a "big key" (Chapter 13) — model time-series or unbounded-growth data by sharding across multiple keys (e.g., `leaderboard:daily:2026-07-06` per day) instead of one key that grows forever.
- **Attach a TTL to every key that represents transient state** (sessions, rate-limit counters, cache entries) at creation time, not as an afterthought — a forgotten TTL is how caches silently turn into unbounded memory leaks.
- **Avoid `KEYS *` and unbounded `SCAN` loops in application code entirely.** `KEYS` blocks the single-threaded event loop for the whole keyspace; use `SCAN` with a small `COUNT` and treat pattern-matching over the whole keyspace as an operational/debugging tool, not a request-path operation.
- **Prefer `EXPIRE`/`PEXPIRE`/`SET ... EX` over ad hoc "check a timestamp field and ignore stale rows" application logic** — let Redis's native expiration do the work it's designed for instead of reinventing TTL semantics in every service.
- **Version your key schema when a data type change is coming** (e.g., migrating `session:{id}` from a plain string to a hash): dual-write or use a prefix bump (`session:v2:{id}`) rather than mutating semantics under the same key name while old and new code paths coexist during a rollout.

```bash
# Hash tags: co-locate related keys onto the same Cluster slot
HSET cart:{42}:items sku:1001 2
HSET cart:{42}:meta  updated_at 1751808000
# Both keys hash on "42" only — a MULTI/EXEC or Lua script touching both
# stays on one slot and doesn't throw CROSSSLOT
```

---

## 2. Persistence & Durability

*(Builds on Chapter 7: Persistence — RDB & AOF)*

- **Decide durability requirements per Redis instance's role, not by copying one config everywhere.** A pure cache (recomputable from a source of truth) can run with persistence disabled entirely; a session store or job queue that would hurt to lose needs AOF; a system-of-record use case needs both.
- **Use RDB alone only when "lose the last few minutes of data on a crash" is genuinely acceptable** — it's the cheapest option operationally (single compact file, fast restarts) but has the largest data-loss window of any persistence strategy.
- **Enable AOF with `appendfsync everysec` as the default durability posture for anything that isn't a pure cache** — it bounds data loss to roughly one second of writes without the write-amplification and latency cost of `appendfsync always`.
- **Reserve `appendfsync always` for the rare case where losing even one second of writes is unacceptable**, and only after confirming (via benchmarking, Chapter 13) that your workload can absorb the fsync-per-write latency cost.
- **Run RDB and AOF together (hybrid persistence) for anything you'd call "production data".** RDB gives you fast full restarts and a stable point-in-time snapshot for backups; AOF gives you the tight recovery-point objective. Redis loads AOF preferentially on restart when both are enabled, giving you the best of each.
- **Size `save` snapshot intervals to your actual write rate, not the defaults blindly.** A high-write-throughput instance triggering frequent RDB forks can cause latency spikes and memory pressure from copy-on-write; tune the `save` points (or disable RDB and rely on AOF + a manual `BGSAVE` schedule) accordingly.
- **Treat every `BGSAVE`/AOF rewrite as a resource event, not a free operation.** Both fork the process; on a memory-constrained host with a large dataset, an ill-timed fork under load is a classic latency-spike root cause — monitor available memory headroom before scheduling additional forks.
- **Test the actual restart-and-recover path, not just "persistence is enabled."** Kill an instance, restart it, and confirm both that it starts and that the data it recovers matches expectations — an AOF file that Redis can't parse on load is a disaster discovered at the worst possible time otherwise.
- **Store RDB files and AOF directories on durable, monitored storage** (not an ephemeral container filesystem that disappears on redeploy) if the instance is expected to survive a restart with its data intact.

---

## 3. Memory Management

*(Builds on Chapter 9: Expiration, Eviction & Memory Management)*

- **Always set `maxmemory` explicitly.** An instance with no `maxmemory` cap will grow until the OS starts swapping or the OOM killer intervenes — neither is a controlled failure mode.
- **Choose the eviction policy based on whether every key is disposable or some are load-bearing.** `allkeys-lru`/`allkeys-lfu` for a pure cache where any key can be recomputed; `volatile-lru`/`volatile-lfu`/`volatile-ttl` when some keys (session, queue, or stream data with no TTL) must never be evicted and only TTL-bearing keys are fair game; `noeviction` when you'd rather see `OOM` command errors than silently lose data.
- **Prefer LFU over LRU for workloads with a long-tail access pattern** (a small hot set accessed constantly, a large cold tail accessed rarely) — LFU tracks actual access frequency and resists the "one big scan evicts my hot keys" failure mode that pure recency-based LRU is prone to.
- **Monitor `used_memory` against `maxmemory` continuously, and alert well before you hit the ceiling** (e.g., at 75% and 90%), not after evictions or `OOM` errors start appearing in application logs.
- **Track `evicted_keys` and `expired_keys` as separate signals.** A rising `evicted_keys` rate on an instance meant to hold non-disposable data is a correctness incident in progress, not a performance quirk.
- **Use `MEMORY USAGE <key>` and `--bigkeys`/`--memkeys` (via `redis-cli`) proactively to find oversized keys** before they become the reason an eviction pass or a `DEL` causes a latency spike.
- **Right-size data structure encodings deliberately.** Small hashes/lists/sets use compact `listpack`/`intset` encodings automatically below configured thresholds (`hash-max-listpack-entries`, etc.) — know these thresholds and avoid crossing them by accident with unbounded growth, since the conversion to the full encoding costs both memory and, transiently, CPU.
- **Don't rely on `maxmemory-policy` as a substitute for TTLs on data that should simply expire.** Eviction is a last-resort safety valve for memory pressure, not a design pattern for expiring data on schedule.
- **Re-evaluate `maxmemory` sizing whenever the workload's key cardinality or value size profile changes materially** — a capacity plan sized for last year's product catalog doesn't automatically hold after a 5x SKU count increase.

---

## 4. Atomicity & Concurrency

*(Builds on Chapter 8: Transactions & Lua Scripting)*

- **Reach for a single atomic command first.** `INCR`, `HINCRBY`, `SET ... NX`, `GETSET`/`SET ... GET` cover a huge share of "I need this to not race" requirements without any transaction machinery at all — the single-threaded event loop already makes each individual command atomic.
- **Use `MULTI`/`EXEC` for a fixed, known sequence of commands that must execute back-to-back with no other client's commands interleaved**, understanding that Redis transactions don't roll back on a runtime error mid-queue — they queue-then-execute, they don't provide relational-style rollback semantics.
- **Use `WATCH` for optimistic locking when a transaction's commands depend on a value read beforehand** (classic check-then-act, like a conditional stock decrement) — `WATCH` aborts the `EXEC` if the watched key changed since the `WATCH`, forcing a retry instead of silently racing.
- **Prefer a Lua script (`EVAL`/`EVALSHA`) or a Redis Function over `WATCH`/`MULTI`/`EXEC` when the logic involves a read-then-conditionally-write decision with real branching**, or when you want to guarantee the whole sequence runs as one atomic server-side unit without a client-side retry loop at all — the script itself runs to completion without interleaving from any other client.
- **Keep Lua scripts short and fast.** A script blocks the entire event loop for its whole runtime, same as any other command — a script with an unbounded loop over a large collection is a latency incident waiting to happen, exactly like an unbounded `KEYS` or `LRANGE`.
- **Always implement a bounded retry loop around `WATCH`-based optimistic locking**, not an infinite `while(true)` — under high contention on a hot key, an unbounded retry loop can itself become a load problem.
- **In Cluster mode, only combine keys in a transaction or Lua script if they share a hash tag** and therefore live on the same slot — this is the concurrency-and-atomicity reason hash tags exist, not just a routing optimization.
- **Cache script SHAs and use `EVALSHA` in the hot path**, falling back to `EVAL` (and re-caching the SHA) on a `NOSCRIPT` error after a restart or `SCRIPT FLUSH` — don't resend the full script body on every call.
- **Avoid holding application-level locks (e.g., a `SETNX`-based mutex) longer than strictly necessary**, and always set a TTL on any lock key so a crashed holder can't wedge the lock forever.

---

## 5. Client Usage

*(Builds on Chapter 10: Client Libraries & Connection Management)*

- **Always use a connection pool sized to your actual concurrency, not "as large as possible."** Too few connections serializes unrelated requests behind each other; too many wastes server-side memory and file descriptors and can itself become the bottleneck under contention.
- **Pipeline batches of independent commands instead of issuing them one round trip at a time.** Pipelining amortizes network round-trip latency across many commands and is one of the single highest-leverage client-side performance changes available, with zero server-side config changes required.
- **Set explicit connect and command timeouts on every client**, and make sure your library's default isn't "wait forever" — a hung connection to a partitioned or overloaded Redis node should fail fast, not stack up waiting threads/requests behind it.
- **Implement retries with exponential backoff and jitter for transient failures** (connection reset, `LOADING`, a Sentinel failover in progress), not a tight immediate-retry loop that can pile onto an already-struggling instance.
- **Treat `MOVED`/`ASK` redirects (Cluster mode) and topology changes (Sentinel failover) as expected, routine events your client library handles, not exceptional errors your application code needs to catch ad hoc** — use a Cluster-aware or Sentinel-aware client rather than hand-rolling redirect/failover handling.
- **Avoid `MULTI`/`EXEC` or Lua scripts as a substitute for pipelining** when you don't actually need atomicity — pipelining alone (no transaction wrapper) is cheaper and sufficient for "send many commands, care about throughput, don't care about interleaving."
- **Close and recycle connections that repeatedly error**, and configure health checks so pool members don't silently degrade over long-lived connections.
- **Keep the client library up to date and match it to your Redis server's protocol version** (RESP2 vs. RESP3) — an outdated client missing newer command support or RESP3 push-message handling is a common source of subtle bugs against a newer server.
- **Instrument client-side command latency, not just server-side `INFO` stats** — the gap between the two often reveals network or connection-pool contention that server metrics alone won't show.

---

## 6. High Availability & Scaling

*(Builds on Chapter 11: Replication & High Availability and Chapter 12: Redis Cluster & Sharding)*

- **Run an odd number of Sentinel processes (minimum 3), spread across separate failure domains** (different hosts/AZs) — an even number or a single Sentinel process can't establish a reliable majority quorum for failover decisions.
- **Set the Sentinel `quorum` value deliberately, and understand it governs *agreement to start* a failover, not the number of Sentinels required to *complete* one** — a quorum of 2 out of 3 Sentinels is a common, reasonable baseline; don't set it equal to your total Sentinel count, which makes failover fragile to a single Sentinel being briefly unreachable.
- **Always route writes to the primary and reads to replicas deliberately in application code (or via a Sentinel/Cluster-aware client)**, and never assume a replica is safe for a read that must reflect the very latest write — replication is asynchronous by default, and stale reads are a real, expected possibility.
- **Design your Cluster hash-tag strategy before sharding, not after.** Every multi-key operation, transaction, and Lua script that needs to span keys must have those keys co-located via a shared `{tag}` — retrofitting hash tags onto a live, populated Cluster means a data migration, not a config change.
- **Choose a resharding-friendly key distribution up front** (enough natural cardinality in your hash-tag values that hash slots spread evenly) — a hash tag that collapses too many keys onto one tag (e.g., tagging by a single tenant when one tenant dominates traffic) recreates a hot-shard problem inside a horizontally scaled Cluster.
- **Test failover in a non-production environment before you need it in production.** Kill the primary, confirm Sentinel promotes a replica within your expected time window, confirm the client reconnects to the new primary without manual intervention — don't discover a misconfigured client library's failover handling during a real incident.
- **Monitor replication lag (`master_repl_offset` vs. each replica's offset) continuously**, and treat a growing lag as an early warning of network, CPU, or disk saturation on the replica — not just a background statistic.
- **Don't oversize a Cluster past what your workload's multi-key/transactional needs can tolerate.** More shards mean more `CROSSSLOT` surface area for any operation that wasn't designed with hash tags in mind — validate application code against Cluster mode in staging before scaling out, not just against a single-node deployment.
- **Keep Cluster and Sentinel configuration under version control and applied consistently across nodes**, the same discipline as any other production infrastructure — a manually patched single node's config is a well-known source of "why does failover behave differently on this one replica" incidents.

---

## 7. Performance

*(Builds on Chapter 13: Performance Tuning & Benchmarking)*

- **Never run a genuinely O(N) command against an unbounded collection on the request path.** `KEYS *`, `LRANGE key 0 -1` on a large list, `SMEMBERS` on a large set, `HGETALL` on a large hash, and `SORT` without `LIMIT` all block the single-threaded event loop for every other client while they run — page results instead (`LRANGE` with bounded offsets, `HSCAN`/`SSCAN`/`ZSCAN` with cursors).
- **Diagnose hot keys (a single key receiving disproportionate traffic) and big keys (a single key holding disproportionate data) as two distinct problems with two distinct fixes**: hot keys need traffic spreading (client-side caching, key sharding, read replicas), big keys need data-model surgery (splitting one huge structure into many smaller ones).
- **Use `redis-cli --bigkeys` and `MEMORY USAGE` regularly, not only when something already hurts**, to catch a growing big key before it causes a slow `DEL`, a slow eviction pass, or a Cluster resharding hot spot.
- **Check the slow log (`SLOWLOG GET`) as a standing habit, not just during an incident** — a command that's slow at today's data volume was probably fine last month, and the slow log is the earliest, cheapest signal that a query pattern has outgrown its data structure.
- **Benchmark with `redis-benchmark` against realistic command mixes and payload sizes**, not just the default `SET`/`GET` micro-benchmark — a workload dominated by `HGETALL` on medium hashes or Lua scripts behaves very differently than a plain key-value workload.
- **Re-benchmark after any material schema, data-volume, or Redis-version change** — a data model that performed well at 1M keys can behave differently at 100M, and a Redis version bump can shift default behaviors worth re-validating.
- **Avoid `DEBUG` commands and administrative commands (`FLUSHALL`, `FLUSHDB`, synchronous `SAVE`) on a live production instance without a clear operational reason and a change-managed window** — several of them block the event loop and are indistinguishable from an outage while they run.
- **Prefer `UNLINK` over `DEL` for large keys** — `UNLINK` reclaims memory asynchronously in a background thread instead of blocking the event loop for the full deallocation.
- **Use pipelining and batched commands (`MSET`, `MGET`, pipelined `HSET`) to reduce round trips** whenever a request needs multiple related keys, rather than issuing them as separate sequential round trips.

---

## 8. Observability

*(Builds on Chapter 14: Monitoring & Observability)*

- **Scrape `INFO` (via `redis_exporter` or equivalent) into Prometheus/Grafana on every instance**, not just the ones that have already caused a problem — you want history to compare against when something does go wrong.
- **Alert on `used_memory` approaching `maxmemory`, rising `evicted_keys` on non-disposable data, rising `blocked_clients`, and growing replication lag** as the four highest-signal leading indicators of an approaching incident.
- **Watch `instantaneous_ops_per_sec` and connected client counts for sudden discontinuities**, not just absolute thresholds — a sudden 10x spike or drop is often the first visible symptom of a client-side bug (a retry storm, a connection leak) well before it becomes a server-side outage.
- **Keep the slow log and `MONITOR` in your toolkit for targeted, time-boxed investigation, never as an always-on production tool** — `MONITOR` in particular has a measurable throughput cost and should only run briefly, deliberately, against a specific problem.
- **Enable and review keyspace notifications selectively** (Chapter 6/14) for use cases that genuinely need to react to expiration/eviction events, and be aware of the throughput cost of enabling them broadly on a busy instance.
- **Track command-level latency percentiles (`LATENCY HISTORY`, `LATENCY LATEST`), not just averages** — a single slow Lua script or big-key command can hide inside a healthy-looking average while still causing real tail-latency pain for a subset of requests.
- **Dashboard replication and Cluster topology health explicitly** (primary/replica roles, Sentinel-observed state, Cluster slot coverage) so a partial failover or a Cluster with unassigned slots is visible at a glance, not discovered from an application error.
- **Correlate Redis-side metrics with application-side latency and error rates** in the same dashboard or time window — an isolated Redis metric rarely tells the whole story of a user-facing incident by itself.

---

## 9. Security

*(Builds on Chapter 15: Security)*

- **Never run a Redis instance reachable from the public internet with no authentication.** This is, across every database technology covered in this course series, the single most common real-world exposure incident — an unauthenticated Redis instance is typically found and exploited within minutes of being exposed.
- **Use ACLs to grant least-privilege access per application/service**, not a single shared superuser credential — an application that only needs `GET`/`SET` on `product:*` keys should not also be able to run `FLUSHALL` or `CONFIG SET`.
- **Enable TLS for both client connections and replication/Cluster bus traffic** whenever the network path between Redis and its clients or peers isn't already fully trusted (e.g., crosses a VPC boundary, a public cloud network, or an untrusted subnet).
- **Rename or disable dangerous administrative commands** (`FLUSHALL`, `FLUSHDB`, `CONFIG`, `SHUTDOWN`, `DEBUG`) for connections that don't need them, via `rename-command` or ACL category restrictions — a compromised application credential shouldn't automatically mean a compromised whole instance.
- **Put Redis behind a network boundary (VPC, security group, firewall) as the primary control, with auth/TLS as defense in depth**, not the other way around — auth alone doesn't protect against a network-level attack surface that shouldn't exist in the first place.
- **Rotate credentials and TLS certificates on a defined schedule**, and make credential rotation an operational non-event by using ACL users (which can be added/removed independently) instead of a single shared `requirepass`.
- **Audit who can run `CONFIG SET`, `ACL`, and other administrative commands, and log/alert on their use** — these commands can silently change an instance's security posture (disabling protected mode, changing `requirepass`) without any data-plane signature that monitoring would otherwise catch.
- **Disable protected mode only when you have deliberately configured `bind` and firewall rules**, never as a blanket workaround for a connectivity problem during development that then ships to production unnoticed.

---

## 10. Operational Discipline

These practices don't belong to a single earlier chapter — they're what "running Redis in production, over time" means once the data model, persistence, memory, and security are already correct.

- **Test backup restoration on a real schedule, not just backup creation.** An RDB file or AOF backup that's never been restored is a hope, not a plan — run a periodic restore drill against a non-production instance and confirm the data matches expectations.
- **Capacity-plan for both memory and connection count ahead of growth**, not reactively after `maxmemory` alerts start firing — track key cardinality and average value size trends over time and project forward against your `maxmemory` ceiling and Cluster shard count.
- **Roll out configuration changes (persistence settings, `maxmemory-policy`, ACL changes, Cluster topology changes) to a staging environment with production-representative data and traffic first**, and apply them to one node/replica at a time in production, confirming health before proceeding to the next.
- **Run periodic game-day failover drills** — deliberately trigger a primary failure in a controlled environment and verify Sentinel promotion, client reconnection, and application-level behavior end to end, on a recurring cadence, not just once at initial launch.
- **Keep a documented, rehearsed incident-response runbook** for the failure modes most likely to actually occur: memory pressure and evictions on non-disposable data, a stuck or slow Lua script, a Sentinel quorum loss, a Cluster with unassigned hash slots, and a replica falling significantly behind after a network partition.
- **Version-control every piece of Redis configuration** (`redis.conf`, ACL definitions, Sentinel config, Cluster topology) the same way you version application code, and require review for changes to any of it.
- **Re-run this chapter's checklist before every new Redis-backed feature launch, not just once per deployment's lifetime** — a healthy cluster today can be handed a data-model or traffic-pattern change tomorrow that violates several of these practices at once.

---

## Pillars of a Production-Grade Redis Deployment

```mermaid
mindmap
  root((Production-Grade\nRedis Deployment))
    Data Modeling
      object-type:id keys
      Right-sized data types
      Hash tags for Cluster
      No unbounded keys
    Durability
      RDB for fast restarts
      AOF for tight RPO
      Restore drills
    Memory
      maxmemory always set
      Eviction policy matches role
      evicted_keys alerting
    Atomicity
      Single atomic commands first
      WATCH/MULTI/EXEC
      Lua for branching logic
    High Availability
      Sentinel odd quorum
      Cluster hash slots
      Read/write routing
    Performance
      No O(N) on hot path
      Hot/big key mitigation
      redis-benchmark discipline
    Observability
      INFO to Prometheus
      Slow log review
      Replication lag alerts
    Security
      ACLs least privilege
      TLS everywhere untrusted
      No public exposure
```

---

## Real-World Scenario

**Setup:** QuickCart's platform team is about to ship a new **real-time inventory-sync service**: warehouse scanners publish stock-level deltas, a consumer applies them to a per-SKU counter in Redis, and the storefront reads current stock for the "only 3 left!" badge and to block checkout on out-of-stock items. Before it goes live, the team runs this chapter's checklist against the design.

**Data modeling.** Stock levels are stored as `inventory:{sku}` strings with `INCRBY`/`DECRBY` for deltas — a good fit, since atomic counters are exactly what strings are for (Section 1). But the design also needs a per-warehouse breakdown (`inventory:{sku}:{warehouseId}`) plus a rollup total, and the review catches that these two keys don't share a hash tag. On the planned Cluster deployment, a Lua script that atomically adjusts the per-warehouse count *and* the rollup total would hit `CROSSSLOT` the moment it's deployed against a multi-shard Cluster. **Fix:** rekey to `inventory:{sku}:total` and `inventory:{sku}:wh:{warehouseId}`, so the shared `{sku}` hash tag co-locates every key for a given SKU onto one slot (Section 1, Section 6).

**Persistence.** The team's first instinct is to treat inventory data like the existing `product:{sku}` cache — no persistence, rebuild from the source-of-truth database if the instance restarts. The review pushes back: unlike the product cache, inventory deltas from warehouse scanners are **not fully replayable** after a restart — some deltas would be lost between the last DB sync and the crash, understating or overstating stock. **Fix:** enable AOF with `appendfsync everysec` on this instance, treating it as durable state, not a disposable cache (Section 2).

**Memory.** The new keys are small and low-cardinality (one per SKU, one per SKU-warehouse pair), so memory impact is negligible — this section passes without changes, but the team adds `inventory:*` to the existing `evicted_keys`-by-prefix dashboard breakdown so an eviction here specifically (as opposed to on the disposable product cache) trips an alert (Section 3, Section 8).

**Atomicity.** The initial implementation reads the current count, checks it against the requested checkout quantity in application code, then writes the decrement — a classic check-then-act race under concurrent checkouts. **Fix:** replace it with a small Lua script that reads, checks, and conditionally decrements atomically in one round trip, exactly the branching-logic case Section 4 calls out for Lua over `WATCH`/`MULTI`/`EXEC`.

**Client usage & performance.** The storefront's read path calls `GET inventory:{sku}:total` per product on a listing page rendering 40 products — 40 sequential round trips per page load today. **Fix:** switch to a pipelined `MGET`-equivalent batch read (Section 5, Section 7), cutting round trips from 40 to 1.

**High availability.** Because this data is now durable and load-bearing (not recomputable-on-restart like the product cache), the review confirms it sits on the same Sentinel-managed replica set as the cart data, with reads routed to the primary for checkout-time stock checks (where staleness would cause overselling) and replicas acceptable only for the non-blocking "X left" badge display (Section 6).

**Security.** The warehouse-scanner ingestion service gets its own ACL user scoped to `INCRBY`/`DECRBY`/`EVALSHA` on `inventory:*` only — it cannot run `FLUSHALL` or read `session:*` keys even if its credential leaked (Section 9).

**Operational discipline.** The launch is gated on one additional item the checklist surfaces: a failover drill specifically exercising this new key range hasn't been run since it moved onto the shared replica set. The team schedules one before go-live rather than treating the existing cart-service drill as sufficient coverage (Section 10).

**Outcome:** The checklist catches a Cluster-breaking hash-tag omission, a durability strategy copied wrongly from an unrelated cache, and a checkout-race condition — all before the feature reaches production traffic, plus two efficiency and access-control improvements that would have shipped as technical debt otherwise.

---

## Best Practices

The condensed top-10 cross-cutting list — the fastest possible pass through the entire course:

1. **Design your key schema and hash-tag strategy before writing data, not after** — `object-type:id` naming and `{tag}` co-location are expensive to retrofit onto a populated Cluster.
2. **Match persistence to the instance's actual role**: disposable cache → none needed; anything you'd hurt to lose → AOF (`everysec`) plus RDB, tested by actually restoring it.
3. **Always set `maxmemory` and choose the eviction policy deliberately** — `volatile-*` when some keys must survive, `allkeys-*` only when every key is truly disposable.
4. **Default to single atomic commands; reach for `WATCH`/`MULTI`/`EXEC` for known sequences and Lua for branching read-then-write logic** — never a client-side check-then-act race.
5. **Pool connections, pipeline batched commands, and set real timeouts with backoff-based retries** — the single highest-leverage, lowest-risk client-side change available.
6. **Run Sentinel with an odd quorum (3+) or Cluster with a deliberate hash-slot/hash-tag design**, and route reads/writes correctly for your consistency requirement.
7. **Never run an O(N) command against an unbounded collection on the request path** — page with cursors, and treat hot keys and big keys as two distinct problems with two distinct fixes.
8. **Monitor memory headroom, eviction rate, replication lag, and slow-log entries continuously**, not just when something has already broken.
9. **Never expose an unauthenticated Redis instance to any untrusted network; use ACLs for least privilege and TLS wherever the network path isn't fully trusted.**
10. **Test failover and backup restoration on a recurring schedule, roll out config changes to staging first and node-by-node in production, and re-run this checklist before every new feature launch.**

---

## Common Mistakes

If you only avoid these five things, you'll avoid the large majority of real-world Redis production incidents:

- **Deploying an unauthenticated Redis instance reachable from an untrusted network** — Chapter 15's single most consequential lesson, and the most common real Redis exposure incident industry-wide.
- **Running an O(N) command (`KEYS *`, unbounded `LRANGE`/`SMEMBERS`/`HGETALL`, unlimited `SORT`) on the request path**, blocking the single-threaded event loop for every other client — the direct consequence of not internalizing Chapter 3's architecture and Chapter 13's performance guidance together.
- **Treating persistence as one-size-fits-all** — either enabling it nowhere (losing load-bearing data on a crash) or enabling `appendfsync always` everywhere without benchmarking the latency cost — instead of matching Chapter 7's RDB/AOF trade-offs to each instance's actual role.
<br>
- **Building multi-key transactions or Lua scripts without hash tags on a Cluster deployment**, discovering `CROSSSLOT` errors only after the data is already sharded and a hash-tag retrofit means a real migration (Chapters 8 and 12).
- **Skipping restore drills and failover drills** — having a backup that's never been restored and a Sentinel/Cluster topology that's never had a real failover rehearsed against it, then discovering both are broken during an actual incident instead of a scheduled drill (Chapters 7, 11, and this chapter's Section 10).

---

## Summary

- **Data modeling**: consistent `object-type:id` key naming, the narrowest data type that fits the access pattern, and hash tags designed in from the start for anything that will run on a Cluster.
- **Persistence**: match RDB/AOF strategy to each instance's role, and prove recovery works by actually restoring, not just enabling the config flag.
- **Memory**: `maxmemory` always set, eviction policy matched to which keys are disposable, and continuous monitoring of headroom and eviction rate.
- **Atomicity**: single atomic commands first, `WATCH`/`MULTI`/`EXEC` for known sequences, Lua for branching read-then-write logic — never a client-side race.
- **Client usage**: pooled connections, pipelined batches, real timeouts, and backoff-based retries.
- **High availability & scaling**: odd-quorum Sentinel or a hash-tag-designed Cluster, with reads and writes routed for your actual consistency requirement.
- **Performance**: no O(N) commands on the request path, hot/big keys treated as distinct problems, and `redis-benchmark` re-run after material changes.
- **Observability**: memory, eviction, replication lag, and slow-log metrics dashboarded and alerted on continuously.
- **Security**: no unauthenticated public exposure, least-privilege ACLs, TLS on untrusted network paths.
- **Operational discipline**: tested restores, tested failovers, staged rollouts, and a checklist you re-run before every new feature launch — exactly as QuickCart's platform team did in this chapter's scenario.

---

## Knowledge Check

1. Why does adding a hash tag (`{sku}`) to a set of related keys matter specifically on Redis Cluster, and what error do you get if you skip it and then run a multi-key Lua script across those keys?
2. A colleague argues their new feature's Redis data doesn't need AOF because "we can always rebuild it from the database." Under what condition is that actually true, and what property of the inventory-sync scenario in this chapter made it false there?
3. What's the difference between `volatile-lru` and `allkeys-lru` as eviction policies, and which would you choose for a Redis instance that holds both a rate-limiter counter (with TTL) and a job queue (no TTL)?
4. Explain why `WATCH`/`MULTI`/`EXEC` is the wrong tool for a "read the current stock, and if sufficient, atomically decrement it" operation under high concurrency, and what this chapter recommends instead.
5. Why is `KEYS *` dangerous specifically in a single-threaded server, in a way that wouldn't be as severe on a multi-threaded database?
6. Name the four leading indicators this chapter recommends alerting on before they become an outage, and explain what a bad trend in each one would predict.
7. A Sentinel deployment has exactly 2 Sentinel processes with quorum set to 2. What's wrong with this configuration, and what's the minimum recommended fix?
8. Why does this chapter recommend testing backup restoration and failover on a recurring schedule rather than once at initial launch?

---

## Hands-On Exercise

Pick a Redis deployment to audit — your own local/staging instance, a service you run at work, or a hypothetical deployment you sketch out for this exercise (e.g., a session store plus a job queue plus a small leaderboard, similar to QuickCart's early setup). Walk it against this chapter's full ten-section checklist and produce a written remediation list with three columns for every finding:

1. **Section/item** — which numbered checklist item it maps to (e.g., "Section 3: `maxmemory` not set").
2. **Current state** — what the deployment actually does today.
3. **Risk if left as-is, and proposed fix** — be concrete: not "improve security" but "enable ACLs and remove the shared superuser credential from the `orders-api` service."

At minimum, cover: key naming consistency, persistence strategy versus each instance's actual role, whether `maxmemory` and an eviction policy are set, whether any multi-key operation would break under Cluster hash-tag rules, connection pooling/pipelining/timeout configuration in your client code, Sentinel/Cluster topology (or the lack of one, if this is a single instance — note that as a finding itself if the workload doesn't tolerate single-instance downtime), whether the slow log and `INFO` metrics are actually being watched anywhere, ACL/auth/TLS/network exposure, and whether a backup restore or failover has ever actually been tested.

Rank your findings by severity, as a platform team would before a launch review — an unauthenticated public-facing instance and a missing `maxmemory` cap outrank a slightly inconsistent key-naming convention, even though both are legitimate findings.

---

## Further Reading

- [Redis Docs — Redis Enterprise / OSS Best Practices](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/) — the official memory optimization and operational guidance referenced across Sections 1–3.
- [Redis Docs — Redis Security](https://redis.io/docs/latest/operate/oss_and_stack/management/security/) — the official ACL, TLS, and network hardening reference behind Section 9.
- [Redis Docs — High Availability with Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/) — quorum sizing and failover mechanics behind Section 6.
- [Redis Docs — Scaling with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) — hash slots and hash tags in full detail, behind Sections 1 and 6.
- [Redis Docs — Latency Monitoring](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency-monitor/) — the slow log and `LATENCY` command family behind Sections 7 and 8.
- *Redis in Action* (Josiah Carlson) — worked, production-flavored examples of the data-modeling and atomicity patterns this chapter condenses.

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-security.md">← Previous: Security</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./17-common-mistakes-and-pitfalls.md">Next: Common Mistakes & Pitfalls →</a>
</div>
