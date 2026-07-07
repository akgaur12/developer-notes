# Best Practices

Across the last fifteen chapters you learned dozens of individually correct recommendations: embed data that's read and updated together, put equality fields before range fields in a compound index, filter before you `$lookup`, avoid transactions when embedding would do, use `w: "majority"` for durability, and lock a deployment down with least-privilege RBAC. Each one made sense in the context of the chapter that introduced it. What's been missing is the **view from above** — all of these recommendations gathered in one place, organized by theme instead of by chapter number, so you can run through them quickly before a design review, a code review, or a production launch. This chapter is that reference: the checklist a senior engineer runs a MongoDB design or deployment against before signing off on it. Treat it as a document you return to repeatedly — before you finalize a schema, before you ship a new aggregation pipeline, before you flip a new cluster into production — not as a one-time read.

---

## Learning Objectives

By the end of this chapter, you will be able to:

- Recite a concise, defensible checklist of best practices spanning schema design, indexing, aggregation, CRUD, transactions, availability, scaling, security, and operations.
- Explain the reasoning behind each practice well enough to adapt it when your situation doesn't match the textbook case.
- Run a structured pre-production review of a MongoDB deployment and identify the highest-severity gaps first.
- Recognize the handful of decisions (write concern, shard key, index design, RBAC scope) that are expensive to change after data and traffic accumulate, and get them right before launch.
- Distinguish practices that are "nice to have" from the small set that are load-bearing for correctness, durability, or security.
- Audit a real or hypothetical schema/index/deployment configuration against this chapter's consolidated checklist.

---

## Prerequisites for This Chapter

This chapter is a **synthesis** chapter. It assumes you have completed Chapters 1 through 15 and have working knowledge of everything it references — it does not re-teach any technique, it distills and cross-links what you've already learned into one operational reference. The major theme areas it draws from:

- **[Chapter 5: Data Modeling & Schema Design](./05-data-modeling-and-schema-design.md)** — embedding vs. referencing, named schema design patterns, schema validation.
- **[Chapter 6: Indexes Fundamentals](./06-indexes-fundamentals.md)** and **[Chapter 14: Performance Tuning & Query Optimization](./14-performance-tuning-and-query-optimization.md)** — index types, the ESR rule, and reading query plans.
- **[Chapter 4: CRUD Fundamentals](./04-crud-fundamentals.md)** — atomic update operators, bulk writes, cursors, and projections.
- **[Chapters 7–10: The Aggregation Pipeline](./07-aggregation-pipeline-fundamentals.md)** — pipeline stages, expressions, and advanced patterns like `$merge` and `$setWindowFields`.
- **[Chapter 11: Transactions & ACID](./11-transactions-and-acid.md)** — single-document atomicity, multi-document transactions, read/write concern.
- **[Chapters 12–13: Replication, High Availability & Sharding](./12-replication-and-high-availability.md)** — replica sets, failover, shard keys, and read preference.
- **[Chapter 15: Security](./15-security.md)** — authentication, RBAC, encryption, network security, and auditing.

If any of these feel unfamiliar, a quick re-read before continuing will make this chapter much more useful — every bullet below has a full chapter behind it if you need the complete explanation.

---

## 1. Schema Design Best Practices

*(Builds on Chapter 5: Data Modeling & Schema Design)*

- **Default to embedding for data that is read together, updated together, and bounded in size.** Only reach for referencing when the embedded side would grow unbounded, needs to be queried independently, or is shared across many parent documents.
- **Recognize and apply named patterns deliberately, not accidentally.** The Subset Pattern (keep only the most recent N sub-items embedded, reference the rest), the Extended Reference Pattern (duplicate a few frequently-read fields from a referenced document to avoid a join on the hot path), and the Bucket Pattern (group time-series readings into fixed-size buckets) each solve a specific, recognizable access-pattern problem — reach for the pattern that matches your actual read/write shape rather than reinventing an ad hoc structure.
- **Model around your application's real query patterns, not around normalized relational instinct.** A schema copied from a relational ER diagram, with every relationship expressed as a foreign-key reference, throws away MongoDB's core advantage: colocating related data for atomic, single-document access.
- **Enforce a validation schema (`$jsonSchema`) on every production collection**, even a loose one. Validation catches malformed documents at write time instead of turning "what shape is this field, really?" into an archaeology exercise across millions of documents six months later.
- **Avoid unbounded arrays.** An array that grows without limit (all of a user's activity history, forever) eventually pushes the document past the 16MB BSON document size limit and degrades every operation that touches the document, because MongoDB must rewrite the whole document region on most updates. Cap it, bucket it, or move the tail into a separate collection.
- **Version your schema changes deliberately** — add new fields as optional, backfill in the background, and never require an atomic all-documents migration in a live collection when you can instead let old and new document shapes coexist behind an application-level `schemaVersion` field.

```javascript
// Schema validation on a new collection: enforce shape, not just presence
db.createCollection("orders", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["customerId", "status", "total", "createdAt"],
      properties: {
        customerId: { bsonType: "string" },
        status: { enum: ["pending", "shipped", "delivered", "cancelled"] },
        total: { bsonType: "decimal", minimum: 0 },
        createdAt: { bsonType: "date" }
      }
    }
  },
  validationLevel: "moderate"   // validate new/modified docs, tolerate legacy shapes
});

// Extended Reference Pattern: duplicate the few fields the order list actually needs
// so rendering an order history never requires a $lookup into `customers`.
db.orders.insertOne({
  customerId: "C-4471",
  customerSnapshot: { name: "Aria Novak", tier: "gold" },  // denormalized, read-hot fields
  status: "pending",
  total: NumberDecimal("129.99"),
  createdAt: new Date()
});
```

---

## 2. Indexing Best Practices

*(Builds on Chapter 6: Indexes Fundamentals and Chapter 14: Performance Tuning & Query Optimization)*

- **Apply the ESR rule when designing compound indexes**: Equality fields first, then Sort fields, then Range fields. This ordering lets the index narrow to an exact equality match before walking a sorted or bounded range, which is almost always dramatically more selective than the reverse ordering.
- **Index for your actual, known query and sort patterns — never speculatively.** Every index is a real, ongoing cost: it must be updated on every insert/update/delete that touches an indexed field, it consumes disk, and it competes for RAM in the WiredTiger cache. An unused index is pure cost with zero benefit.
- **Prefer one well-ordered compound index over several overlapping single-field indexes.** The prefix rule means a compound index `{ a: 1, b: 1, c: 1 }` already serves queries on `{a}`, `{a,b}`, and `{a,b,c}` — a separate `{ a: 1 }` index alongside it is almost always redundant.
- **Always verify with `explain("executionStats")` before shipping a query or index change** — never assume an index is being used just because one "should" match. Compare `nReturned` to `totalDocsExamined`; a large gap means the index isn't selective enough even though it's technically in use.
- **Test index changes in a staging environment against production-representative data volume before applying them in production.** An index that builds instantly on a 10,000-document staging collection can take hours and consume significant I/O on a 500-million-document production collection — plan the maintenance window accordingly.
- **Use partial indexes to keep large collections' indexes lean** when queries only ever target a well-defined subset (e.g., `status: "active"`), and reach for TTL indexes instead of a manual cleanup job for anything that should self-expire.
- **Periodically audit and drop unused indexes.** `$indexStats` reports per-index usage counters — an index with zero reads over a meaningful window is a pure liability that should be dropped.

```javascript
// ESR in practice: equality on tenantId, sort on createdAt, range on amount
db.transactions.createIndex({ tenantId: 1, createdAt: -1, amount: 1 });

db.transactions.find({
  tenantId: "T-9",
  amount: { $gte: 100 }
}).sort({ createdAt: -1 })
  .explain("executionStats");
// Check: winningPlan.stage is IXSCAN, and nReturned ≈ totalDocsExamined

// Find indexes nobody is using
db.transactions.aggregate([{ $indexStats: {} }])
  .forEach(ix => print(ix.name, ix.accesses.ops));
```

---

## 3. Aggregation Pipeline Best Practices

*(Builds on Chapters 7–10: the Aggregation Pipeline)*

- **Filter as early as possible.** A `$match` at the start of the pipeline, on indexed fields, lets the aggregation engine use an index scan before any documents flow into later stages — every stage after that `$match` then processes fewer documents.
- **Project early to shrink documents flowing through the pipeline.** A `$project` (or `$unset`) that drops fields you don't need reduces the memory footprint of every subsequent stage, particularly before a `$group`, `$sort`, or `$lookup`.
- **Avoid unnecessary `$unwind`.** Unwinding an array multiplies document count by array length for every downstream stage; if you only need an aggregate over the array (a sum, a count, the first N elements), reach for array expression operators (`$sum` on the array directly, `$filter`, `$slice`, `$map`) instead of unwinding and re-grouping.
- **Put `$match` before `$lookup`, not after.** Filtering the base collection first means `$lookup` performs its join against a much smaller candidate set; filtering after `$lookup` means you paid the join cost for documents you were going to throw away anyway.
- **Materialize expensive, frequently-read rollups with `$merge` rather than recomputing them on every request.** A dashboard aggregation that scans millions of documents on every page load should instead run on a schedule and write its result to a small materialized collection that the dashboard reads directly.
- **Use `allowDiskUse: true` deliberately, not reflexively.** It lets `$group`/`$sort` spill to disk past the default 100MB in-memory limit per stage, which is sometimes genuinely necessary — but reaching for it automatically can mask a pipeline that should have filtered or indexed better upstream, and disk-spilled stages are markedly slower than in-memory ones.
- **Check `explain()` on aggregation pipelines too**, not just `find()` queries — look for `IXSCAN` on the initial `$match`/`$sort`, and watch for a `$lookup` that becomes an unindexed nested loop over the foreign collection (index the foreign key field it joins on).

```javascript
// Filter and project early; join against a much smaller set; avoid $unwind
// where a fold-style expression will do.
db.orders.aggregate([
  { $match: { status: "delivered", createdAt: { $gte: ISODate("2026-01-01") } } },  // early filter, indexed
  { $project: { customerId: 1, items: 1, total: 1 } },                              // shrink documents
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },
  {
    $group: {
      _id: "$customer.region",
      revenue: { $sum: "$total" },
      itemCount: { $sum: { $size: "$items" } }   // sum array size, no $unwind needed
    }
  },
  { $merge: { into: "revenue_by_region_daily", whenMatched: "replace" } }  // materialize the rollup
], { allowDiskUse: true });   // deliberate: this report scans a full day of orders
```

---

## 4. CRUD and Application-Code Best Practices

*(Builds on Chapter 4: CRUD Fundamentals)*

- **Use atomic update operators instead of read-then-write in application code.** `findOneAndUpdate({ _id }, { $inc: { stock: -1 } })` is a single atomic operation; reading `stock`, decrementing it in application code, then writing it back is a race condition under any concurrent load — two requests can both read the same value before either writes back.
- **Use bulk write operations for batches of writes** (`bulkWrite`, `insertMany`) instead of looping individual `insertOne`/`updateOne` calls from application code — bulk operations dramatically reduce round-trip overhead and let the server batch the work.
- **Never issue an unbounded query in application code.** Always pair `find()` with a `.limit()` (or paginate with a range-based cursor on an indexed field) — an unbounded query against a growing collection is a latent outage waiting for the collection to grow past "fits comfortably in memory."
- **Use projections to fetch only the fields you need.** Pulling entire documents when only two fields are used wastes network bandwidth and memory on both ends, and can prevent a query from being covered by an index.
- **Prefer `findOneAndUpdate`/`findOneAndDelete` over separate find + delete/update calls** when you need the affected document's state — this is both atomic and saves a round trip.
- **Set sensible connection pool sizes and always close cursors/sessions** — connection pool exhaustion and unclosed long-lived cursors are two of the most common causes of "the database is fine, but the app can't reach it" incidents.

```javascript
// Wrong: read-then-write race condition
const doc = await db.products.findOne({ _id: sku });
await db.products.updateOne({ _id: sku }, { $set: { stock: doc.stock - 1 } });

// Right: atomic conditional update — never oversells even under concurrency
await db.products.findOneAndUpdate(
  { _id: sku, stock: { $gte: 1 } },
  { $inc: { stock: -1 } }
);

// Bulk writes instead of a loop of individual calls
await db.orders.bulkWrite(
  updates.map(u => ({
    updateOne: { filter: { _id: u.id }, update: { $set: { status: u.status } } }
  })),
  { ordered: false }
);

// Never unbounded — always paginate
db.orders.find({ status: "pending" }).sort({ _id: 1 }).limit(100);
```

---

## 5. Transactions and Consistency Best Practices

*(Builds on Chapter 11: Transactions & ACID)*

- **Avoid transactions where embedding already gives you atomicity.** Every single-document write — including nested fields and array mutations — is atomic in MongoDB with no transaction required. Reach for a multi-document transaction only when you have two or more genuinely independent entities (e.g., separate `accounts` documents, or `products` and `orders` in different collections) that must change together.
- **Always drive transactions through `withTransaction()`**, not hand-rolled commit/abort logic — it implements the correct retry behavior for `TransientTransactionError` and `UnknownTransactionCommitResult` out of the box.
- **Treat retryable transaction errors as normal, expected conditions**, not application bugs — a replica set election or momentary network blip mid-transaction is routine in a distributed system, and the correct response is to retry, not to surface a 500 to the end user.
- **Keep transactions short-lived** — never make an external network call (a payment gateway request, an email send) from inside an open transaction, and route every operation inside the transaction through the session handle, not the top-level `db` object.
- **Choose read concern and write concern deliberately, matched to what the operation actually needs.** `readConcern: "snapshot"` and `writeConcern: "majority"` together are what make an ACID claim complete; weakening either weakens the corresponding guarantee (isolation or durability, respectively).
- **Use `w: "majority"` for any write where losing it on primary failover would be unacceptable** — financial transactions, order placement, anything with a real-world consequence if it silently disappears. Reserve weaker write concerns (`w: 1`) for genuinely disposable data (e.g., high-volume telemetry where an occasional lost write is an acceptable trade for throughput).

```javascript
// Deliberate write concern choice: majority for money-moving writes
db.payments.insertOne(
  { accountId: "A-1", amount: NumberDecimal("250.00"), status: "captured" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
);

// Deliberate, weaker write concern for high-volume, disposable telemetry
db.pageViewEvents.insertOne(
  { url: "/product/42", ts: new Date() },
  { writeConcern: { w: 1 } }
);
```

---

## 6. High Availability and Scaling Best Practices

*(Builds on Chapters 12–13: Replication & High Availability, Sharding & Scalability)*

- **Run a minimum of a three-node replica set in production**, never a standalone `mongod`. A standalone has no failover and, notably, cannot run multi-document transactions at all.
- **Choose write concern based on the durability you actually need**, not the default. `w: "majority"` guarantees a write survives a primary failover; `w: 1` (acknowledged only by the primary) can silently lose the most recent writes if the primary crashes before replicating them to any secondary.
- **Choose read preference deliberately for read scaling — and understand what you're trading away.** Routing reads to secondaries (`secondaryPreferred`) spreads read load, but secondaries can lag behind the primary (replication lag), so reads may return slightly stale data — acceptable for a dashboard, unacceptable for "did my write just succeed" confirmation reads, which should read from the primary or use causal consistency.
- **Choose a shard key deliberately, before you have production data volume, because changing it later is expensive.** A good shard key has high cardinality, distributes writes evenly (avoids monotonically increasing keys like `_id` or a timestamp, which create a hot chunk that all inserts pile onto), and matches your dominant query pattern so most queries can be targeted to a single shard rather than broadcast to all of them.
- **Plan for zone sharding when you have geographic or regulatory data-locality requirements** (e.g., EU customer data pinned to EU-region shards) — decide this during initial cluster design, not after data is already distributed arbitrarily.
- **Monitor replication lag continuously** — a secondary that falls far behind the primary is both a stale-read risk and, in an election, a candidate that may lose recent data if it becomes primary before catching up.

```javascript
// Good shard key: high cardinality, evenly distributed, matches query pattern
sh.shardCollection("shop.orders", { customerId: "hashed" });

// Read preference chosen deliberately per use case
db.orders.find({ customerId: "C-1" })
  .readPref("primary");              // "did my write just land" — must be current

db.orders.aggregate([{ $group: { _id: "$region", total: { $sum: "$amount" } } }])
  .readPref("secondaryPreferred");   // analytics dashboard — slight staleness is fine
```

---

## 7. Security Best Practices

*(Builds on Chapter 15: Security)*

- **Never expose an unauthenticated `mongod`/`mongos` instance to any network beyond `localhost`.** Authentication should be enabled (`--auth` / `security.authorization: enabled`) on every deployment, including staging and internal-only clusters — "internal" networks get breached too, and an unauthenticated MongoDB instance reachable from anywhere is one of the most common real-world data-breach root causes.
- **Apply least-privilege RBAC.** Application service accounts get exactly the roles they need (typically `readWrite` scoped to their specific database), never `root` or `dbOwner`; human operators get time-boxed elevated roles for specific tasks, not standing admin access.
- **Enforce TLS/SSL for all client-server and intra-cluster traffic**, not just external-facing connections — traffic between application servers and `mongod`, and between replica set members, should be encrypted in any environment that isn't a fully isolated single-host development setup.
- **Enable encryption at rest** for any deployment holding sensitive data, using the WiredTiger encrypted storage engine or your cloud provider's disk-level encryption (Atlas enables this by default).
- **Bind to specific network interfaces and use firewall/security-group rules to restrict access to known application hosts** — `bindIp` should never be `0.0.0.0` in production without a compensating network-level restriction in front of it.
- **Enable auditing on any deployment subject to compliance requirements**, and rotate credentials and X.509/TLS certificates on a defined schedule rather than treating them as "set once, forget forever."

```javascript
// Least-privilege role for an application service account:
// read/write on exactly one database, nothing else.
db.createUser({
  user: "orders_service",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "shop" } ]
});

// Never this for an application account:
// db.createUser({ user: "orders_service", roles: [ "root" ] });
```

```
# mongod.conf — never bind to 0.0.0.0 without a firewall in front of it
net:
  bindIp: 10.0.4.12,127.0.0.1
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/mongodb.pem
security:
  authorization: enabled
```

---

## 8. Operational and Production Best Practices

These practices don't belong to a single earlier chapter — they're what "running MongoDB in production" means once the schema, indexes, and pipelines are already correct.

- **Monitor the metrics that predict trouble before it becomes an outage**: replication lag, WiredTiger cache hit ratio, current connection count against the configured max, queued read/write operations (a rising queue depth is the earliest signal of a resource bottleneck), disk I/O and free space, and `explain()`-visible query plan regressions on your hottest queries. Waiting until users complain means debugging blind with no historical baseline.
- **Plan capacity ahead of growth, not reactively.** Track working-set size against available RAM (the WiredTiger cache should comfortably hold your hot working set), disk growth rate against provisioned storage, and connection count trends against your driver's pool configuration — extrapolate forward, don't just look at today's snapshot.
- **Have a tested backup and disaster recovery strategy, not just a backup.** Use `mongodump`/`mongorestore` for logical backups of smaller datasets or ad hoc exports, filesystem/volume snapshots for point-in-time backups of larger deployments, and MongoDB Atlas's continuous backup (continuous oplog-based backup enabling point-in-time restore) if you're on Atlas. Critically: **restore from your backups periodically as a drill** — a backup nobody has ever successfully restored is a hope, not a plan.
- **Test index changes and schema migrations in a staging environment that mirrors production data volume before applying them in production.** An index build, a validation rule rollout, or a large `$merge` job can behave completely differently at production scale than on a small staging dataset.
- **Roll out changes to a replica set one node at a time (rolling upgrades/restarts).** Upgrade or restart secondaries first, verify each rejoins healthy and catches up on replication lag, then step down the primary and upgrade it last — this keeps the replica set continuously available throughout the maintenance window instead of taking the whole set down at once.
- **Version your MongoDB server upgrades deliberately** — read the release notes for feature-compatibility-version (FCV) changes, upgrade FCV only after the whole replica set/cluster is confirmed running the new binary version, and never skip FCV bump verification, since it gates which on-disk format and feature set the cluster commits to using.
- **Keep a documented, rehearsed incident-response runbook** for the failure modes most likely to actually occur: primary failover, a runaway query saturating CPU, an out-of-disk secondary, and a credential/certificate expiry.

```bash
# Logical backup and restore — smaller datasets, or targeted collection exports
mongodump --uri="mongodb://user:pass@host:27017" --db=shop --out=/backups/2026-07-06
mongorestore --uri="mongodb://user:pass@host:27017" --db=shop /backups/2026-07-06/shop

# Rolling upgrade order on a 3-node replica set:
# 1. Upgrade/restart each secondary, one at a time, confirming rs.status() shows
#    it back to SECONDARY with lag near zero before moving to the next node.
# 2. rs.stepDown() on the primary, then upgrade/restart it last.
```

### Diagram: Pre-Production Launch Checklist

```mermaid
flowchart TD
    Start([New deployment ready for review]) --> Schema{Schema validated?\nEmbed/reference decisions justified?}
    Schema -- No --> SchemaFix[Revisit Ch 5 patterns\nadd $jsonSchema validation]
    Schema -- Yes --> Index{Indexes match\nreal query patterns?\nESR applied?}
    Index -- No --> IndexFix[Design compound indexes\nverify with explain()]
    Index -- Yes --> Agg{Aggregation pipelines\nfilter/project early?\nexpensive rollups materialized?}
    Agg -- No --> AggFix[Reorder stages\nadd $merge materialization]
    Agg -- Yes --> Txn{Write concern and\ntransaction usage appropriate?}
    Txn -- No --> TxnFix["Set w: majority for durable writes\nremove unneeded transactions"]
    Txn -- Yes --> HA{Replica set sized?\nShard key chosen deliberately?}
    HA -- No --> HAFix[3+ node replica set\nhigh-cardinality shard key]
    HA -- Yes --> Sec{Auth enabled?\nTLS everywhere?\nleast-privilege RBAC?}
    Sec -- No --> SecFix[Enable auth + TLS\nscope roles down]
    Sec -- Yes --> Ops{Monitoring, backups,\nrunbooks in place?}
    Ops -- No --> OpsFix[Wire up dashboards\ntest restore from backup]
    Ops -- Yes --> Launch([Cleared for production launch])

    SchemaFix --> Schema
    IndexFix --> Index
    AggFix --> Agg
    TxnFix --> Txn
    HAFix --> HA
    SecFix --> Sec
    OpsFix --> Ops
```

---

## Real-World Scenario

**Setup:** You're the senior engineer running the pre-launch review for a mid-sized application's new production MongoDB deployment — a subscription billing platform moving off a legacy system. The team demos the system, and you walk it theme by theme against this chapter's checklist.

**Schema design.** The `subscriptions` collection embeds each customer's active plan and payment method, and references a separate `invoices` collection for historical billing records — a reasonable embed/reference split per Chapter 5, and there's a `$jsonSchema` validator on both collections. This section passes.

**Indexing.** You ask to see the indexes on `invoices`, since the dashboard's "show a customer's last 12 invoices" query is one of the highest-traffic queries in the app. The only index present is the default `_id` index — the query is filtering on `customerId` and sorting by `issuedAt`, and it's currently running as a `COLLSCAN` against a collection projected to reach tens of millions of documents within a year. **This is Issue #1**: a missing compound index (`{ customerId: 1, issuedAt: -1 }`) on the platform's single most common read pattern. Left unaddressed, this query degrades linearly as the collection grows and will be the first thing to page someone at 2 a.m.

**Aggregation pipelines.** The monthly revenue-recognition report is a five-stage pipeline that correctly filters early on `status` and date range before `$group`ing — good. It is not materialized, though; it's recomputed from scratch on every dashboard page load, scanning the full month's invoices each time. This is flagged as a lower-priority improvement (materialize with `$merge` on a nightly schedule) rather than a launch blocker.

**CRUD and transactions.** Payment capture correctly uses `findOneAndUpdate` with a conditional filter to atomically mark an invoice as paid — no read-then-write race. But you notice the write concern on that same payment-capture write is the driver default, `w: 1`. **This is Issue #2**: an acknowledged-by-primary-only write concern on a financial write. If the primary crashes microseconds after acknowledging the write but before replicating it, a payment can be recorded as captured and then vanish on failover — for a billing system, this is a durability gap that must be closed with `writeConcern: { w: "majority" }` before launch, not after the first missing-payment support ticket.

**High availability and scaling.** The production cluster is a properly sized three-node replica set with a sensible shard key already planned for the `invoices` collection ahead of the projected data volume. This section passes.

**Security.** Production auth and TLS are correctly configured. But while reviewing the full topology, you notice the team's **staging cluster** — used for load testing with a full production-shaped dataset, including realistic (if synthetic) customer and payment records — has `security.authorization` disabled "temporarily, to make load testing easier," and is reachable from the office network with no firewall restriction. **This is Issue #3**: an exposed, unauthenticated cluster holding production-shaped sensitive data. This is exactly the kind of gap Chapter 15 calls out as a common real-world breach vector — "staging" is not a security exemption.

**Operations.** Monitoring dashboards for replication lag and connection count are live; a nightly `mongodump` backup runs, but nobody has ever run a test restore from one of those backup files. You flag this as a pre-launch requirement: run one full restore drill before go-live, not after the first incident that needs it.

**Outcome:** Three issues are caught before launch — the missing `invoices` compound index, the `w: 1` write concern on payment captures, and the unauthenticated staging cluster — each one a direct match to a checklist item in this chapter. All three are fixed in under a day once identified; each one would have been materially more expensive to discover in production, in exactly the pattern this chapter's checklist exists to prevent.

---

## Best Practices

The condensed top-10 cheat sheet — the fastest possible pass through this chapter:

1. **Embed data that's read and updated together; reference what's independent or unbounded** — and validate every collection's shape with `$jsonSchema`.
2. **Design compound indexes with the ESR rule** (Equality, Sort, Range) and verify every hot query with `explain("executionStats")` — never assume.
3. **Filter and project early in every aggregation pipeline**; avoid `$unwind` when an array expression will do; materialize expensive, frequently-read rollups with `$merge`.
4. **Use atomic operators (`findOneAndUpdate`, `$inc`, `$push`) instead of read-then-write**, and always bound your queries with `.limit()` or cursor-based pagination.
5. **Avoid transactions where embedding already gives you atomicity**; when you do need one, drive it through `withTransaction()` and keep it short-lived.
6. **Set write concern deliberately per operation** — `w: "majority"` for anything that must survive a failover, weaker only for genuinely disposable data.
7. **Run a minimum three-node replica set, choose your shard key before production data volume accumulates, and route reads via a deliberate read preference.**
8. **Enable auth and TLS everywhere, including staging — never expose an unauthenticated instance — and apply least-privilege RBAC to every account.**
9. **Monitor replication lag, cache hit ratio, connection count, and queued operations continuously**, and build the dashboard before launch, not after the first incident.
10. **Test backups by actually restoring them, test index/schema changes in a production-scale staging environment first, and roll out upgrades one replica set node at a time.**

---

## Common Mistakes

Synthesizing the most consequential anti-patterns from across the whole course:

- **Modeling schemas the relational way** — normalizing everything into separate collections joined by `$lookup` at read time — instead of embedding the data that's actually read and updated together, and then reaching for transactions to paper over the consistency problem that decision created.
- **Adding indexes reactively, one at a time, chasing individual slow queries forever**, instead of designing a small set of compound indexes up front that cover the application's known access patterns, applying the ESR rule deliberately.
- **Recomputing expensive aggregations on every request** instead of materializing them with `$merge` on a schedule, and reaching for `allowDiskUse: true` as a reflex instead of fixing a pipeline that should have filtered earlier.
- **Using the default write concern everywhere**, including for financial or otherwise consequence-bearing writes, without ever asking "what happens to this write if the primary crashes one millisecond after acknowledging it?"
- **Choosing a monotonically increasing shard key** (an auto-incrementing ID or a raw timestamp), creating a hot chunk that all writes pile onto and defeating the entire purpose of sharding for write scaling.
- **Treating "staging" or "internal network" as an implicit security exemption** — running any deployment, anywhere, without authentication and TLS enabled, on the assumption that network obscurity is a substitute for access control.
- **Deploying without a monitoring dashboard or a tested backup restore**, and discovering both gaps for the first time during an actual incident, when there is no time to build either calmly.

---

## Summary

- **Schema design**: embed for togetherness and boundedness, reference for independence and unbounded growth, and validate every collection's shape.
- **Indexing**: apply the ESR rule, index for real query patterns only, and verify every important query with `explain()`.
- **Aggregation**: filter and project early, avoid unnecessary `$unwind`, materialize expensive rollups, and use `allowDiskUse` deliberately rather than reflexively.
- **CRUD**: use atomic operators over read-then-write, bulk writes over loops, and always bound your queries.
- **Transactions**: avoid them where embedding suffices, use `withTransaction()` with correct retry handling, and choose read/write concern to match the guarantee you actually need.
- **High availability and scaling**: never run a standalone `mongod` in production, choose write concern and shard keys deliberately, and route reads with an explicit read preference.
- **Security**: authenticate and encrypt everywhere, including staging, with least-privilege RBAC on every account.
- **Operations**: monitor the leading indicators (replication lag, cache hit ratio, connections, queue depth), plan capacity ahead of growth, test backups by restoring them, and roll out changes gradually with staging validation first.
- The **Real-World Scenario** showed exactly how this checklist catches real, planted issues — a missing index, a weak write concern on a financial write, and an exposed unauthenticated staging cluster — before they become production incidents.

---

## Knowledge Check

1. A colleague argues that embedding a customer's full order history directly in the customer document "avoids the need for transactions entirely." What's wrong with this reasoning, and which two best practices from this chapter does it violate?
2. You're designing a compound index for a query that filters on `region` (equality), filters on `createdAt` (range), and sorts by `priority` (equality-independent sort field). Using the ESR rule, what field order should the index use, and why?
3. Why is `w: "majority"` the right default for a payment-capture write, but potentially the wrong choice for a high-volume clickstream-logging write? Frame your answer in terms of the guarantee each write concern actually provides.
4. A team materializes a revenue rollup with `$merge` on a nightly schedule instead of recomputing it on every dashboard load. What trade-off are they making, and under what circumstance would recomputing on every request actually be the better choice?
5. Explain why an unauthenticated staging cluster is a genuine security risk even though it's "not production" — reference the Real-World Scenario's Issue #3 in your answer.
6. Name three metrics you would want visible on a monitoring dashboard before launching a new production MongoDB deployment, and explain what a bad trend in each one would indicate.

---

## Hands-On Exercise

You've been handed the following description of a deployment ahead of its production launch. Audit it against this chapter's checklist and write down every violation you find, along with which section of this chapter it maps to and why it matters.

**The deployment:**

- A `users` collection stores each user's profile plus an embedded `loginHistory` array that appends one entry per login, with no cap — some long-tenured users already have arrays with over 40,000 entries.
- The `orders` collection has a single-field index on `status` only. The application's most common query filters on `customerId` and sorts by `orderDate` descending.
- The nightly analytics job runs a nine-stage aggregation pipeline that starts with a `$lookup` into the `customers` collection, then applies a `$match` on `region` and `orderDate` afterward.
- The checkout flow reads a product's `stock` field, checks it in application code, and if sufficient, issues a separate `updateOne` to decrement it.
- The replica set is a single primary with one secondary and no third node or arbiter.
- All production writes use the driver's default write concern; nobody on the team has explicitly discussed write concern in a design review.
- The database user configured for the application's backend service has the `dbOwner` role on the whole database "so the team doesn't have to keep adjusting permissions as new collections get added."
- There is a nightly `mongodump` cron job; no one has run `mongorestore` from any of its output since it was set up eight months ago.

For each bullet, identify: (a) which section of this chapter it violates, (b) the concrete risk if left as-is, and (c) the specific fix you'd propose. Then rank your findings by severity, as you would in an actual pre-launch review — not every violation is equally urgent.

---

## Further Reading

- [Production Notes](https://www.mongodb.com/docs/manual/administration/production-notes/) — the official baseline configuration checklist for production deployments.
- [Production Considerations for Transactions](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/) — operational limits and best practices for running transactions safely.
- [Data Modeling Introduction](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/) — the official reference for embedding vs. referencing decisions.
- [Backup and Restore Overview](https://www.mongodb.com/docs/manual/core/backups/) — `mongodump`/`mongorestore`, filesystem snapshots, and backup strategy options.
- [Security Checklist](https://www.mongodb.com/docs/manual/administration/security-checklist/) — the official pre-production security hardening checklist, directly complementing Section 7 of this chapter.

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./15-security.md">← Previous: Security</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./17-common-mistakes-and-pitfalls.md">Next: Common Mistakes & Pitfalls →</a>
</div>
