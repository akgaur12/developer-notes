# Chapter 24 — Operating Postgres in Production

## Where you are
You can design, query, tune, secure, replicate, and back up Postgres. This chapter is the operator's synthesis: how to **run** it day to day — in containers and Kubernetes, with monitoring, sane configuration management, safe upgrades, and a production-readiness checklist. It pulls together threads from nearly every prior chapter.

> **The "why":** Most production incidents aren't exotic — they're a disk filling with WAL, autovacuum falling behind (Chapter 14), connections exhausting (Chapter 19), or an upgrade gone wrong. Operating well means anticipating these *known* failure modes with monitoring and routine, not heroics.

---

## 1. Running Postgres in Docker
Fine for development and small/single-node production, with two non-negotiables: **persist the data** and **manage config**.
```bash
docker run --name pg \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=app \
  -v pgdata:/var/lib/postgresql/data \      # ← named volume: data survives container removal
  -p 5432:5432 \
  -d postgres:18 \
  -c shared_buffers=1GB -c max_connections=100   # pass config flags, or mount a postgresql.conf
```
- **The cardinal Docker mistake:** no volume → the container is ephemeral and your data dies with it. Always mount a volume (or bind mount) for `PGDATA`.
- Pin the **major version** in the image tag (`postgres:18`, not `postgres:latest`) — a surprise major-version jump on restart can break things.
- Set resources (`--memory`, `--cpus`) deliberately; an unbounded container competing for host RAM can get OOM-killed mid-write.

## 2. Running Postgres on Kubernetes
Postgres is **stateful**, so the naive "just run a Deployment" is wrong. The building blocks:
```
StatefulSet         stable network identity + stable per-replica storage (vs Deployment's churn)
PersistentVolumeClaim   durable storage that survives pod rescheduling — the data's real home
Headless Service    stable DNS for each pod (primary vs replicas)
Secrets             credentials (never bake passwords into images or manifests)
readiness/liveness probes   use pg_isready; don't route traffic to a not-ready pod
resource requests/limits    pin memory so the pod isn't OOM-killed; reserve CPU
anti-affinity       keep replicas on different nodes so one node loss ≠ total outage
```
> **Strong recommendation: use a Postgres Operator, not hand-rolled manifests.** Operators encode the hard operational knowledge — failover, backups, connection pooling, rolling upgrades — as automation:
> ```
> CloudNativePG   modern, widely adopted; integrates backups (Barman), PITR, pooling.
> Zalando postgres-operator, Crunchy PGO   other mature, battle-tested options.
> ```
> They run the replication + failover orchestration (Chapter 21), schedule backups (Chapter 22), and manage rolling minor-version upgrades. Self-managing stateful Postgres on K8s by hand is a classic way to learn painful lessons; let an operator carry that weight.

A common pattern: **PgBouncer (Chapter 19) as a sidecar or separate deployment**, with the app pointing at the pooler service rather than directly at Postgres — essential because pods scaling up can otherwise exhaust connections fast.

## 3. Monitoring — what to watch
You can't operate what you can't see. The built-in statistics views are the source of truth; export them to Prometheus/Grafana (via `postgres_exporter`) for dashboards and alerts.
```
pg_stat_activity       current connections & queries — find long/idle-in-transaction sessions (Ch 9/14)
pg_stat_replication    replication lag on the primary (Chapter 21)
pg_stat_user_tables    dead tuples (n_dead_tup), last_autovacuum, seq vs index scans (Chapter 14)
pg_stat_statements     top queries by total time (Chapter 23)
pg_stat_database       commits/rollbacks, cache hit ratio, deadlocks per database
pg_locks               lock contention & blocking chains
```
Key things to **alert** on:
```
• disk space (esp. the WAL/pg_wal directory — a stuck archive_command fills it and HALTS writes)
• replication lag exceeding a threshold
• transaction-ID age approaching wraparound (age(relfrozenxid)) — Chapter 14
• connection count near max_connections
• long-running / idle-in-transaction sessions
• cache hit ratio dropping; deadlock rate rising
```
> **The #1 silent killer: a failing `archive_command`.** If WAL archiving stalls (full archive disk, bad credentials), Postgres keeps WAL on the primary and `pg_wal` grows until the disk is full — then writes stop and the database effectively goes down. Always monitor archive success and `pg_wal` free space.

## 4. Configuration management & applying changes
```
Two kinds of settings:
  • reloadable (most planner/logging settings): apply with a reload, NO downtime
        ALTER SYSTEM SET random_page_cost = 1.1;
        SELECT pg_reload_conf();          -- or `pg_ctl reload`
  • restart-required (shared_buffers, max_connections, shared_preload_libraries):
        change, then restart the instance
SELECT name, setting, pending_restart FROM pg_settings WHERE pending_restart;  -- what needs a restart
SHOW config_file;                                                              -- where config lives
```
Keep config in version control (or in your operator's manifest), not as undocumented hand-edits — reproducibility matters when you rebuild a node.

## 5. Upgrades
```
MINOR upgrades (18.3 → 18.4): bug/security fixes, same on-disk format. Just install the new
       binaries and restart. Do these promptly — they include security patches. Low risk.
MAJOR upgrades (17 → 18): new features, may change internals. Two main paths:
  • pg_upgrade   — fast, in-place (uses hard links with --link); brief downtime. The usual choice.
  • logical replication (Chapter 21) — replicate old→new running in parallel, then switch over;
       enables near-zero-downtime upgrades and easy rollback. Best for can't-afford-downtime systems.
```
Always: read the release notes for breaking changes, **test the upgrade on a copy with production-like data**, ensure a fresh backup exists first, and `ANALYZE` the whole cluster afterward (statistics aren't carried over by `pg_upgrade`, so the planner is blind until you do — a common "why is everything slow right after upgrade?" cause).

## 6. The production-readiness checklist
```
DURABILITY & RECOVERY
  ☐ Automated backups + WAL archiving, with PITR (Chapter 22)
  ☐ Restores TESTED on a schedule (an untested backup isn't a backup)
  ☐ Off-site/separate-account backup copy (3-2-1)
AVAILABILITY
  ☐ At least one standby replica; failover orchestrated (Patroni/operator), not manual (Chapter 21)
  ☐ Connection pooler in front (PgBouncer, transaction mode) (Chapter 19)
PERFORMANCE & HEALTH
  ☐ Memory/planner settings tuned for the hardware (Chapter 19)
  ☐ Autovacuum healthy; scale_factor tuned for big hot tables; wraparound age monitored (Chapter 14)
  ☐ pg_stat_statements enabled; slow queries indexed (Chapters 12, 23)
SECURITY
  ☐ scram-sha-256 auth, TLS required, least-privilege app role, RLS where needed (Chapter 20)
  ☐ Secrets in a secret store, not in code/images
OBSERVABILITY
  ☐ Metrics dashboards + alerts on disk/pg_wal, lag, connections, wraparound, archive failures
  ☐ Slow-query and connection logging on
OPERATIONS
  ☐ Config in version control; documented restart vs reload procedures
  ☐ Upgrade path rehearsed; release notes reviewed; post-upgrade ANALYZE planned
```

---

## Summary
- In **Docker**, always persist `PGDATA` to a volume, pin the major version, and bound resources.
- On **Kubernetes**, Postgres is stateful: use **StatefulSet + PVC + Secrets + probes + anti-affinity**, and strongly prefer a **Postgres Operator** (CloudNativePG, Zalando, Crunchy) that automates failover, backups, and upgrades; front it with a pooler.
- **Monitor** the built-in `pg_stat_*` views (activity, replication, user_tables, statements, database, locks); **alert** on disk/`pg_wal`, replication lag, wraparound age, connection count, and especially a **failing `archive_command`** (the top silent killer).
- Know **reloadable vs restart-required** settings (`pg_settings.pending_restart`); keep config in version control.
- **Minor upgrades** are quick restarts (do them promptly); **major upgrades** use `pg_upgrade` or logical replication, always tested on a copy, with a backup first and a post-upgrade **`ANALYZE`**.
- Ship against a **production-readiness checklist** spanning durability, availability, performance, security, observability, and operations.

## Test your understanding
1. What's the cardinal mistake when running Postgres in Docker, and how do you avoid it?
2. Why is a Kubernetes `Deployment` wrong for Postgres, and what should you use instead — plus why prefer an Operator?
3. Why is a failing `archive_command` one of the most dangerous silent failures, and what does it eventually cause?
4. Which settings can you change with a reload vs which require a restart, and how do you check what's pending?
5. After a `pg_upgrade` major upgrade, queries are suddenly slow. What routine step was likely skipped, and why does it matter?

## Hands-on exercise
1. Run Postgres in Docker with a named volume; insert data, remove and recreate the container, and confirm the data survived. Then repeat *without* a volume to see the data lost.
2. Query `pg_stat_activity` and identify any long-running or `idle in transaction` sessions; query `pg_stat_user_tables` for dead-tuple counts.
3. Change a reloadable setting with `ALTER SYSTEM` + `pg_reload_conf()`, then change a restart-required one and find it in `pg_settings WHERE pending_restart`.
4. Check transaction-ID age with `SELECT max(age(relfrozenxid)) FROM pg_class;` and reason about how close to wraparound that is.
5. Write your own production-readiness checklist for a service you care about, marking which items are done, missing, or not-yet-needed — and (if exploring K8s) read the CloudNativePG quickstart and note how it handles failover and backups.
