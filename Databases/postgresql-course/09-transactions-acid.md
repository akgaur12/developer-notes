# Chapter 09 — Transactions & ACID

## Where you are
You can read and write data in sophisticated ways. Now you learn to make groups of changes **safe** even when things go wrong (crashes, errors) and when **many users act at once**. This is what makes a database trustworthy for money, inventory, bookings — anything where "half-done" is catastrophic.

> **The "why":** The real world has operations that must happen *all or nothing*. Transferring money is two writes (debit one account, credit another). If the system crashes between them, money must not vanish. A transaction makes those two writes a single indivisible unit.

---

## 1. ACID — the four guarantees
```
A — Atomicity    all statements in a transaction succeed, or none do (all-or-nothing)
C — Consistency  the transaction moves the DB from one valid state to another (constraints hold)
I — Isolation    concurrent transactions don't corrupt each other's view of the data
D — Durability   once committed, changes survive a crash (this is what the WAL guarantees)
```
PostgreSQL provides all four. The WAL (write-ahead log, Chapter 01) is the mechanism behind Durability: changes are written to the log and flushed to disk *before* commit returns, so a crash can replay the log to recover.

## 2. The basic transaction block
```sql
BEGIN;                                   -- start
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                                  -- make permanent
-- if anything goes wrong before COMMIT:
ROLLBACK;                                -- undo everything since BEGIN
```
Either both updates land, or neither does. Outside an explicit `BEGIN`, every statement is its own auto-committed transaction.

```
   BEGIN ──▶ stmt ──▶ stmt ──▶ stmt ──▶ COMMIT   (all persisted)
                         │
                         └──▶ error/ROLLBACK ──▶ (all undone, DB unchanged)
```

> **Practical habit (saves careers):** before a risky `UPDATE`/`DELETE` you're unsure about, wrap it in `BEGIN; ... ` run it, `SELECT` to verify, and only then `COMMIT;` — or `ROLLBACK;` if it's wrong. This is your undo button.

## 3. SAVEPOINTs — partial rollback
Within a transaction you can set checkpoints and roll back to them without abandoning the whole thing:
```sql
BEGIN;
INSERT INTO orders ...;
SAVEPOINT after_order;
INSERT INTO order_items ...;   -- if this fails:
ROLLBACK TO after_order;       -- undo just the items, keep the order, continue
COMMIT;
```
Note: in Postgres, once *any* statement errors inside a transaction, the whole transaction is poisoned (`current transaction is aborted`) until you `ROLLBACK` (optionally to a savepoint). Savepoints are how you recover gracefully.

## 4. The concurrency problems isolation prevents
When transactions overlap, several anomalies can occur. Knowing their names tells you what each isolation level protects against:
```
Dirty read           reading another transaction's UNcommitted change
Nonrepeatable read   re-reading a row and getting a different value (someone committed an update between)
Phantom read         re-running a query and getting different ROWS (someone committed an insert/delete)
Serialization anomaly  the result of concurrent transactions couldn't happen in ANY serial order
```

## 5. Isolation levels
SQL defines four; **Postgres implements three distinct behaviors** (its Read Uncommitted behaves like Read Committed — Postgres never allows dirty reads):
```
                    | dirty read | nonrepeatable | phantom | serialization anomaly
READ COMMITTED      |  no        |  possible     | possible|  possible      ← default
REPEATABLE READ     |  no        |  no           |  no*    |  possible      (*Postgres also blocks phantoms here)
SERIALIZABLE        |  no        |  no           |  no     |  no
```
Set per transaction:
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
...
COMMIT;
```
- **Read Committed (default):** each *statement* sees a fresh snapshot of committed data. Good for most apps.
- **Repeatable Read:** the whole *transaction* sees one consistent snapshot from its start (Postgres uses snapshot isolation, which also prevents phantoms). Great for multi-statement reports that must be internally consistent.
- **Serializable:** behaves as if transactions ran one-at-a-time. Postgres uses **Serializable Snapshot Isolation (SSI)**; if it detects a dangerous interleaving it aborts one transaction with a serialization error — so **your application must be prepared to retry**. This is the gold standard for correctness-critical logic.

> **Key consequence:** at `REPEATABLE READ` and `SERIALIZABLE`, transactions can fail at commit with a serialization error even though nothing was "wrong." Production code using these levels **must wrap the transaction in a retry loop.**

## 6. MVCC in one paragraph (full treatment in Chapter 14)
Postgres achieves isolation without making readers wait for writers (or vice versa) using **MVCC — Multi-Version Concurrency Control**. An `UPDATE` doesn't overwrite a row in place; it writes a *new version* and marks the old one as valid only up to a certain transaction. Each transaction sees the versions appropriate to its snapshot. Hence the famous property: **readers don't block writers, and writers don't block readers.** The cost is that obsolete row versions ("dead tuples") accumulate and must be cleaned up by `VACUUM` — the subject of Chapter 14.

## 7. Locking — when transactions DO wait
Writers to the *same row* must coordinate. Postgres locks automatically, but you can lock explicitly:
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;   -- lock these rows; others wait to modify them
SELECT * FROM jobs WHERE status='queued' ORDER BY id
  FOR UPDATE SKIP LOCKED LIMIT 1;                  -- great pattern for job queues: grab one nobody else has
```
- `FOR UPDATE` prevents others from modifying the selected rows until you commit — use it to avoid lost updates (read-modify-write races).
- `SKIP LOCKED` lets workers pull disjoint rows from a queue without blocking each other.

> **Deadlock:** two transactions each hold a lock the other wants. Postgres detects this automatically and aborts one with a deadlock error. Avoid it by always acquiring locks in a **consistent order** (e.g. always lock the lower account id first), and keep transactions short.

## 8. Practical guidance
- **Keep transactions short.** Long-running transactions hold locks and block `VACUUM` from cleaning dead tuples (a real production hazard — Chapter 14).
- **Never leave a transaction open** while waiting on user input or a slow external API.
- **Choose the lowest isolation level that's correct** for the operation; escalate only where needed, and add retries when you do.

---

## Summary
- **ACID** = Atomicity, Consistency, Isolation, Durability; the **WAL** delivers durability.
- `BEGIN ... COMMIT/ROLLBACK` makes changes **all-or-nothing**; **SAVEPOINTs** allow partial rollback within a transaction.
- Concurrency anomalies: **dirty read, nonrepeatable read, phantom, serialization anomaly**.
- Postgres offers **Read Committed (default), Repeatable Read (snapshot, no phantoms), Serializable (SSI)** — and the higher two require **retry-on-serialization-failure** in your app.
- **MVCC** lets readers and writers not block each other, at the cost of dead tuples needing `VACUUM`.
- **`FOR UPDATE`** / `SKIP LOCKED` give explicit row locking; avoid **deadlocks** by locking in a consistent order and keeping transactions short.

## Test your understanding
1. Explain Atomicity with the money-transfer example. What guarantees the change survives a crash after commit?
2. What's the difference between a nonrepeatable read and a phantom read?
3. Postgres's default isolation is Read Committed. Name one anomaly it still permits, and which level removes it.
4. Why must code using SERIALIZABLE (or REPEATABLE READ) include a retry loop?
5. What does `SELECT ... FOR UPDATE` do, and how would `SKIP LOCKED` help build a job queue?

## Hands-on exercise
With two `psql` sessions open side by side (this is the best way to *feel* isolation):
1. In session A: `BEGIN; UPDATE accounts SET balance = balance - 100 WHERE id = 1;` (don't commit). In session B, read that row — observe you see the *old* value (no dirty read). Then `COMMIT` in A and re-read in B.
2. Demonstrate a `ROLLBACK` undoing changes.
3. Use a `SAVEPOINT`, cause an error after it, `ROLLBACK TO` the savepoint, and `COMMIT` the surviving work.
4. In session A, `SELECT ... FOR UPDATE` a row and don't commit; in session B try to `UPDATE` the same row and watch it block until A commits.
5. (Stretch) Trigger a deadlock with two sessions locking two rows in opposite orders; observe Postgres abort one.
