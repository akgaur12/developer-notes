# Chapter 21 — Replication & High Availability

## Where you are
A single Postgres server is a single point of failure and a single source of read capacity. **Replication** copies data to other servers — for resilience (if the primary dies, a replica takes over) and for scaling reads (send queries to replicas). This chapter explains the mechanisms and the real trade-offs. It builds directly on the WAL (Chapters 01, 09, 14) and pairs with backup/recovery (Chapter 22).

> **The "why":** Hardware fails, disks die, data centers lose power. High availability means the system survives those events with minimal downtime and no data loss. The foundation is the **WAL** — the same write-ahead log that gives durability also lets another server replay every change and stay in sync.

---

## 1. The foundation: shipping the WAL
Every change is written to the WAL before it touches data files. Replication works by getting that WAL stream to another server and replaying it there:
```
   PRIMARY (read/write)                         REPLICA / STANDBY (read-only)
   ┌────────────────────┐    WAL stream     ┌────────────────────────────┐
   │ transactions → WAL │ ───────────────▶  │ receive WAL → replay it →   │
   │ → data files       │   (streaming)     │ identical data files        │
   └────────────────────┘                   └────────────────────────────┘
        accepts writes                          serves read-only queries
```

## 2. Physical (streaming) replication
The standard built-in HA mechanism. The replica is a **byte-for-byte physical copy** of the primary, kept current by streaming WAL.
```
• whole-cluster: replicates EVERYTHING (all databases, all tables) — you can't pick.
• replica is read-only ("hot standby") — can serve SELECTs, offloading read traffic.
• same major version required on both sides.
• fast and low-overhead; the default choice for failover/HA.
```
Set up (modern Postgres, simplified):
```bash
# on the standby: clone the primary, then start in standby mode
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -R -X stream
# -R writes the standby signal + connection info automatically
```
The primary needs a replication role and `pg_hba.conf` entry; `wal_level = replica` (default) suffices.

## 3. Synchronous vs asynchronous — the central trade-off
```
ASYNCHRONOUS (default): primary commits and returns immediately; WAL flows to replica "soon after."
  + fast writes, replica can't slow the primary
  - on primary failure, the last few transactions not yet shipped are LOST (small data loss window)

SYNCHRONOUS: primary waits for the replica to confirm the WAL is written before COMMIT returns.
  + zero data loss on failover (the replica has every committed transaction)
  - slower writes (every commit pays a network round-trip); if the sync replica is down,
    commits can BLOCK unless you configure multiple candidates
```
Configured via `synchronous_commit` and `synchronous_standby_names`. Most setups run async for speed and accept a tiny loss window; correctness-critical systems use sync (often with ≥2 standbys so one being down doesn't halt writes). This mirrors the durability trade-off from Chapter 19 — you're choosing where on the speed/safety dial to sit.

## 4. Replication lag and read-your-writes
Async replicas are slightly behind the primary (**replication lag**, usually milliseconds, but it grows under load). Consequence: a user writes to the primary, then a read routed to a replica might not see it yet ("I just saved this — where did it go?"). Strategies:
```
• route reads that must be current to the primary; route the rest to replicas
• "read your writes": for a short window after a user's write, read from the primary
• monitor lag: SELECT * FROM pg_stat_replication;  (on primary) — watch replay_lag
```

## 5. Logical replication — selective and cross-version
Where physical replication copies bytes, **logical replication** copies *changes at the row level* via a publish/subscribe model (since PG10):
```sql
-- on the source (publisher):
CREATE PUBLICATION my_pub FOR TABLE orders, customers;   -- pick specific tables!

-- on the target (subscriber):
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=source dbname=app user=repl password=...'
  PUBLICATION my_pub;
```
```
LOGICAL replication advantages:
  • replicate SPECIFIC tables, not the whole cluster
  • replicate BETWEEN DIFFERENT major versions → enables near-zero-downtime upgrades
  • target can have its own additional tables/indexes and accept writes
  • consolidate many databases into one, or feed data warehouses / CDC pipelines
LOGICAL replication limits:
  • DDL is NOT replicated (schema changes must be applied manually on both sides)
  • sequences need manual handling; some edge cases require care
```
Logical replication is the modern tool for **major-version upgrades with minimal downtime** (replicate old→new, then switch over) and for change-data-capture.

## 6. Failover and automatic HA tooling
Built-in replication gives you a standby, but Postgres does **not** auto-promote it when the primary dies — promotion is manual (`pg_ctl promote` / `SELECT pg_promote()`) unless you add orchestration:
```
Patroni     the most popular HA orchestrator: uses a consensus store (etcd/Consul/ZooKeeper)
            to elect a leader and auto-promote a standby on failure, preventing split-brain.
repmgr      simpler replication management & failover tooling.
pg_auto_failover  Microsoft's monitor-based failover.
+ a connection router (HAProxy, PgBouncer, or a virtual IP) so clients reconnect to the new primary.
```
> **Split-brain — the danger to respect:** if a network partition lets two nodes both think they're primary, they accept conflicting writes and you get irreconcilable data. Automatic failover tools use a consensus/quorum store and **fencing** (ensuring the old primary is truly down/demoted) specifically to prevent this. This is *why* you use a battle-tested orchestrator rather than a homemade "ping and promote" script.

## 7. Putting together a typical production topology
```
   ┌─────────────┐   sync/async WAL   ┌─────────────┐
   │  PRIMARY    │ ─────────────────▶ │ STANDBY 1   │ (failover target)
   │ (writes)    │ ─────────────────▶ │ STANDBY 2   │ (read replica / 2nd failover)
   └─────────────┘                    └─────────────┘
          ▲ managed by Patroni (leader election, auto-promote) + etcd
          │
     HAProxy/PgBouncer  ←── apps connect here; routed to current primary / read replicas
          +  continuous backups & WAL archiving to object storage (Chapter 22)
```
Replication is **not** a backup (it faithfully replicates a `DROP TABLE` too) — you need both. That's Chapter 22.

---

## Summary
- Replication ships the **WAL** to other servers; the foundation of both HA and read scaling.
- **Physical (streaming) replication** copies the whole cluster byte-for-byte to a read-only **hot standby**; same major version; the default HA mechanism (`pg_basebackup -R`).
- **Synchronous vs asynchronous** is the core trade-off: async = fast but a small data-loss window on failover; sync = zero loss but slower commits (use ≥2 standbys to avoid blocking).
- **Replication lag** means replicas can be slightly stale → route must-be-current reads to the primary.
- **Logical replication** (publications/subscriptions) replicates *selected tables*, works *across major versions* (enabling low-downtime upgrades) and for CDC — but doesn't replicate DDL.
- Failover isn't automatic — use **Patroni**/repmgr + a consensus store and a connection router; respect **split-brain** and use fencing. Replication is **not** a substitute for backups.

## Test your understanding
1. How does WAL make replication possible, tying back to its role in durability?
2. State the precise trade-off between synchronous and asynchronous replication, and when you'd choose each.
3. What is replication lag and what user-visible bug does it cause? Name one mitigation.
4. Give two things logical replication can do that physical replication cannot, and one important limitation of logical replication.
5. What is split-brain, why is a homemade auto-promote script dangerous, and how do tools like Patroni prevent it?

## Hands-on exercise
(Two Postgres instances, e.g. two Docker containers/ports.)
1. Configure a replication role and `pg_hba.conf`, then clone the primary to a standby with `pg_basebackup -R` and start it; confirm it's read-only.
2. Insert rows on the primary and confirm they appear on the standby; check `pg_stat_replication` on the primary for lag.
3. Stop the primary and promote the standby (`pg_promote()`); confirm it now accepts writes.
4. On a fresh pair, set up **logical replication** for a single table with a publication/subscription and confirm only that table syncs.
5. Write (in comments) the topology and tooling you'd use for production HA, and one sentence on why replication still doesn't remove the need for backups.
