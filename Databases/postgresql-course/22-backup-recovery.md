# Chapter 22 — Backup & Recovery

## Where you are
Replication (Chapter 21) protects against hardware failure — but it faithfully copies your mistakes too. A `DROP TABLE` or a bad `UPDATE` replicates instantly to every standby. **Backups** are the independent, point-in-time safety net that lets you recover from human error, corruption, ransomware, and disasters. This is the chapter where "I administer Postgres" becomes "I can be trusted with data that matters."

> **The "why":** Replication answers "is the server up?"; backups answer "can I get the data *as it was* back?" They solve different problems. The only backup that counts is one you've **tested by restoring** — an untested backup is a hope, not a plan.

---

## 1. Two families of backup
```
LOGICAL backup   a dump of SQL/data that recreates objects and rows (pg_dump / pg_dumpall).
                 • portable across versions/architectures
                 • selective (single table/schema/db)
                 • slower to take & restore on big DBs; a snapshot at one instant
PHYSICAL backup  a byte-level copy of the data directory + WAL (pg_basebackup / file copy).
                 • fast for large clusters; whole-cluster
                 • enables Point-In-Time Recovery (PITR) when combined with archived WAL
                 • same major version / platform to restore
```

## 2. Logical backups with pg_dump / pg_dumpall
```bash
# one database, custom format (compressed, allows selective + parallel restore) — the best default:
pg_dump -Fc -d mydb -f mydb.dump

# plain SQL (human-readable, restore with psql):
pg_dump -d mydb -f mydb.sql

# just schema, or just data:
pg_dump --schema-only -d mydb -f schema.sql
pg_dump --data-only   -d mydb -f data.sql

# the WHOLE cluster incl. roles/globals (pg_dump skips roles & tablespaces!):
pg_dumpall -f cluster.sql
pg_dumpall --globals-only -f globals.sql      # just roles/tablespaces, to pair with per-db dumps
```
Restore:
```bash
# custom format → pg_restore (supports parallelism and selective restore):
createdb mydb_restored
pg_restore -d mydb_restored -j 4 mydb.dump            # -j 4 = 4 parallel jobs
pg_restore -d mydb_restored --table=orders mydb.dump  # restore just one table

# plain SQL → psql:
psql -d mydb_restored -f mydb.sql
```
> **The `pg_dumpall` gotcha:** `pg_dump` of a database does **not** include roles, passwords, or tablespaces — those are cluster-global. A per-database dump alone won't fully reconstruct your setup. Capture globals with `pg_dumpall --globals-only` alongside your per-db dumps. Many a restore has failed at 2 a.m. because the roles were missing.

A `pg_dump` is **consistent** — it runs in a single transaction snapshot (MVCC, Chapter 14), so it captures one coherent instant even while the database is live.

## 3. Physical backups and the magic of PITR
A logical dump captures *one moment* (when it ran). **Point-In-Time Recovery (PITR)** lets you restore to *any* moment — e.g. "the state at 14:32:05, one second before the bad `DELETE`." It combines a physical **base backup** with a continuous **archive of WAL**:
```
   base backup (Sunday 00:00)  +  every WAL segment since  =  restore to ANY second after Sunday 00:00
   ┌──────────────┐   ──WAL──▶ ──WAL──▶ ──WAL──▶ ──WAL──▶
   │ full snapshot│   replay WAL forward, then STOP at your chosen recovery target
   └──────────────┘
```
Set up:
```
# 1. continuously archive WAL (postgresql.conf):
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'   # (real setups push to object storage)

# 2. take periodic base backups:
pg_basebackup -D /backups/base_$(date +%F) -Ft -z -X stream
```
Recover to a target time:
```
# restore the base backup, then in the recovery config set:
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-06-19 14:32:05+05:30'
# Postgres replays WAL up to that instant and stops — the bad DELETE never happens.
```
PITR is the difference between "we lost everything since last night's dump" and "we lost three seconds." For any serious system, PITR (or a tool that manages it) is the standard.

## 4. Backup management tools (use these in production)
Hand-rolling `archive_command`, retention, and PITR is error-prone. Mature tools handle scheduling, parallelism, compression, encryption, retention, and (crucially) verified restores:
```
pgBackRest   the de-facto standard: parallel, incremental, compressed, encrypted, S3/object-store,
             integrated PITR and retention. Strongly recommended.
Barman       another well-regarded backup & recovery manager.
WAL-G        cloud-native continuous archiving, good for object storage.
```
Managed services (RDS, Cloud SQL) do this for you — but you must still **know your retention window and test restores**.

## 5. The 3-2-1 rule and what actually matters
```
3-2-1:  3 copies of the data, on 2 different media/systems, with 1 copy OFF-SITE.
```
Define your targets explicitly:
```
RPO (Recovery Point Objective): how much data can you afford to LOSE?  → drives backup frequency / WAL archiving
RTO (Recovery Time Objective):  how fast must you be back UP?          → drives backup type & restore rehearsal
```
> **The one rule that overrides all others: a backup you have not restored is not a backup.** Schedule **regular test restores** into a scratch environment and verify the data. Backups silently fail — bad credentials, full disks, missing WAL, missing roles — and you find out only when you need them, unless you rehearse. Automate restore drills.

## 6. What backups do NOT cover (and pairing with replication)
- A backup is a point in time; between backups you rely on **WAL archiving** to close the gap (RPO).
- **Replication ≠ backup** (it copies mistakes); **backup ≠ HA** (restoring takes time). Production needs **both**: replicas for fast failover, backups + PITR for recovery from logical errors and disasters.

---

## Summary
- **Logical** backups (`pg_dump`/`pg_dumpall`) are portable and selective but snapshot one instant; **physical** backups (`pg_basebackup`) are fast, whole-cluster, and the basis for PITR.
- Use **custom format** (`-Fc`) + `pg_restore` (`-j` parallel, selective). Remember **`pg_dump` excludes roles/globals** — also run `pg_dumpall --globals-only`.
- **PITR** = base backup + archived WAL → restore to *any* second; the standard for serious systems and the cure for "recover to just before the bad query."
- Use a **backup manager** (pgBackRest/Barman/WAL-G) rather than hand-rolling; follow **3-2-1**; define **RPO/RTO**.
- **Test restores regularly** — an untested backup is a hope. **Replication and backups are complementary, not substitutes.**

## Test your understanding
1. Why is replication not a substitute for backups, and vice versa?
2. What does `pg_dump` of a single database fail to include, and how do you capture it?
3. Explain PITR: what two ingredients does it need, and what problem does it uniquely solve?
4. Define RPO and RTO, and say which backup decisions each one drives.
5. What's the single most important backup practice, and why do backups fail silently without it?

## Hands-on exercise
1. Take a `pg_dump -Fc` of a database, drop the database, recreate it, and restore with `pg_restore` (try a selective single-table restore too).
2. Capture globals with `pg_dumpall --globals-only` and inspect what it contains.
3. Enable `archive_mode`/`archive_command` to a local directory, take a `pg_basebackup`, generate some WAL, and confirm WAL segments are being archived.
4. (Stretch) Perform a PITR: note a timestamp, run a destructive statement after it, then restore the base backup and replay WAL to `recovery_target_time` just before the destruction; verify the data is intact.
5. Write a one-paragraph backup plan for a hypothetical production DB: tool, schedule, retention, RPO/RTO, off-site copy, and how often you'll rehearse restores.
