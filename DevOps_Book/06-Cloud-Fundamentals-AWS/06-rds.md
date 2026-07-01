# Chapter 6 — RDS & Database Services

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why managed database services save operational overhead
- Launch and configure an RDS instance in private subnets
- Set up Multi-AZ for high availability and Read Replicas for read scaling
- Create manual snapshots and restore from them
- Understand when to choose Aurora, ElastiCache, or DynamoDB
- Select the right database service for a given use case

---

## 6.1 Why Managed Databases?

Running a database on a self-managed EC2 instance means you are responsible for:

- Installing and configuring the DB engine
- OS and DB engine patching
- Setting up automated backups
- Configuring replication and failover
- Monitoring disk usage and scaling storage
- Encryption at rest and in transit

**RDS (Relational Database Service)** handles all of that for you. AWS manages: OS patches, DB engine patches, automated backups and snapshots, Multi-AZ failover, and storage scaling.

You still manage: schema design, query optimization, indexes, connection pooling, and instance sizing. The split is "AWS handles the infrastructure; you handle the application layer."

> **Cost tradeoff**: RDS costs more per unit of compute than a raw EC2 instance running the same DB engine. You pay for the operational simplicity. For small projects or where you need full control, a self-managed DB on EC2 is reasonable. For production at scale, RDS or Aurora is almost always the right choice.

---

## 6.2 RDS Supported Engines

| Engine | AWS Managed | Aurora Available | Notes |
|---|---|---|---|
| **PostgreSQL** | Yes | Yes (Aurora PG) | Most popular for new projects |
| **MySQL** | Yes | Yes (Aurora MySQL) | Widely used, good free tier |
| **MariaDB** | Yes | No | MySQL fork |
| **Oracle** | Yes | No | Enterprise licensing required |
| **SQL Server** | Yes | No | Windows/enterprise workloads |
| **Aurora Serverless** | N/A | Yes | Auto-scales, pause when idle |

> **Starting a new project?** Default to PostgreSQL on RDS or Aurora PostgreSQL. It is open-source, feature-rich, well-supported, and free of licensing costs.

---

## 6.3 Launching an RDS Instance

RDS instances must be placed in a **DB Subnet Group** — a collection of subnets across at least two Availability Zones. Always place RDS in private subnets.

```bash
# Create a DB subnet group (RDS must span at least 2 AZs)
aws rds create-db-subnet-group \
  --db-subnet-group-name prod-db-subnets \
  --db-subnet-group-description "Production DB subnets" \
  --subnet-ids subnet-private-1a subnet-private-1b

# Create PostgreSQL RDS instance
aws rds create-db-instance \
  --db-instance-identifier prod-postgres \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 15.4 \
  --master-username dbadmin \
  --master-user-password $(aws secretsmanager get-secret-value \
      --secret-id db-password \
      --query SecretString \
      --output text) \
  --db-name appdb \
  --allocated-storage 20 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-db \
  --db-subnet-group-name prod-db-subnets \
  --multi-az \
  --backup-retention-period 7 \
  --no-publicly-accessible

# Get connection endpoint
aws rds describe-db-instances \
  --db-instance-identifier prod-postgres \
  --query 'DBInstances[0].Endpoint.Address'
```

Key parameters explained:

| Parameter | Notes |
|---|---|
| `--db-instance-class` | CPU/memory size. `db.t3.micro` is free tier; `db.t3.medium` for production. |
| `--storage-type gp3` | Newer, cheaper than gp2; IOPS and throughput configurable separately. |
| `--multi-az` | Creates a synchronous standby in another AZ for automatic failover. |
| `--backup-retention-period 7` | Keep automated backups for 7 days (max 35). |
| `--no-publicly-accessible` | DB not reachable from the internet — access only from within VPC. |

> **Never put database passwords in CLI commands.** Use Secrets Manager or Parameter Store, as shown above. Passwords in shell history or process lists are a serious security risk.

---

## 6.4 RDS Multi-AZ (High Availability)

Multi-AZ creates a synchronous standby replica in a second Availability Zone. If the primary instance fails, RDS automatically promotes the standby.

```
Primary Instance (AZ-a)  ←── synchronous replication ──► Standby Instance (AZ-b)
        │
        └── if primary fails: automatic failover (60–120 seconds)
            DNS endpoint stays the same — app reconnects automatically
```

Important details:

- The standby is **not readable** — it exists only for failover
- Failover is triggered by: AZ outage, DB instance failure, OS maintenance, manual failover
- Your application connects to the **endpoint DNS name** (not an IP) — DNS TTL is 5 seconds, so reconnection is fast
- Multi-AZ roughly doubles your RDS cost — budget accordingly

```bash
# Trigger a manual failover (test your app's reconnection logic)
aws rds reboot-db-instance \
  --db-instance-identifier prod-postgres \
  --force-failover
```

---

## 6.5 RDS Read Replicas

**Read Replicas** are asynchronous copies of the primary instance used to offload read-heavy queries.

- **Asynchronous replication**: slight lag (usually milliseconds, but can be higher under load)
- **Readable**: your application can send SELECT queries to replicas
- Up to 5 read replicas per primary instance
- Can be in the same region or different regions (cross-region replication)
- Can be promoted to standalone instance (useful for migrations)

```bash
# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-postgres-read \
  --source-db-instance-identifier prod-postgres \
  --db-instance-class db.t3.medium

# Application connection split:
# Writer: prod-postgres.xxxxx.us-east-1.rds.amazonaws.com
# Reader: prod-postgres-read.xxxxx.us-east-1.rds.amazonaws.com
```

> **Read Replica vs Multi-AZ**: These are independent features for different problems. Multi-AZ is for high availability (failover). Read Replicas are for read scaling and performance. You can (and often should) use both together.

---

## 6.6 RDS Backups and Snapshots

RDS has two backup mechanisms:

**Automated Backups**: enabled by default, stored in S3, support point-in-time recovery to any second within the retention period. Retention: 1–35 days.

**Manual Snapshots**: persist until you explicitly delete them — not subject to retention period. Use before risky operations like major version upgrades.

```bash
# Create a manual snapshot before a major change
aws rds create-db-snapshot \
  --db-instance-identifier prod-postgres \
  --db-snapshot-identifier prod-postgres-pre-migration-20240115

# List snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier prod-postgres

# Restore from snapshot to a NEW instance (cannot restore in-place)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier prod-postgres-pre-migration-20240115
```

> **Restore creates a new instance**: you cannot overwrite the existing DB. After restore, update your application's connection string, or use a CNAME/alias to switch traffic. This also means restores take time — plan for downtime during incident recovery.

---

## 6.7 Amazon Aurora

**Aurora** is AWS's proprietary MySQL/PostgreSQL-compatible database engine built specifically for the cloud.

Key differences from standard RDS:

- **Performance**: AWS claims 5x faster than MySQL, 3x faster than PostgreSQL for the same hardware
- **Storage**: decoupled from compute; auto-scales from 10GB to 128TB in 10GB increments
- **Durability**: 6 copies of data across 3 AZs (2 copies per AZ)
- **Replication**: up to 15 low-latency read replicas (vs 5 for standard RDS)
- **Aurora Global Database**: replicate across multiple regions with under 1 second replication lag
- **Aurora Serverless v2**: scales CPU and RAM in under 1 second — pay per ACU-second when active

```bash
# Aurora exposes two endpoint types:

# Cluster endpoint (writer) — always points to primary
my-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com:5432

# Reader endpoint — load-balanced across all read replicas
my-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com:5432
```

> **When to choose Aurora over RDS**: Aurora is more expensive but delivers better performance, more replicas, and faster failover (~30 seconds vs ~60–120 for Multi-AZ RDS). Choose Aurora for production workloads where performance and reliability matter. Use standard RDS for smaller workloads or when cost is the primary constraint.

---

## 6.8 ElastiCache — In-Memory Caching

**ElastiCache** provides managed in-memory data stores. In-memory means microsecond latency — orders of magnitude faster than reading from a relational database.

Two engines:

- **Redis**: supports data structures (lists, sets, hashes), pub/sub messaging, persistence, clustering, and replication. Most commonly used.
- **Memcached**: simple key-value caching, multi-threaded, no persistence. Fewer features, slightly simpler.

```bash
# Create a Redis replication group (primary + 1 replica)
aws elasticache create-replication-group \
  --replication-group-id prod-redis \
  --replication-group-description "Production Redis" \
  --cache-node-type cache.t3.medium \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --cache-subnet-group-name prod-cache-subnets
```

Common use cases:

| Use Case | Why Redis |
|---|---|
| Session storage | Fast reads, TTL support, data persists across app restarts |
| API response caching | Serve repeated queries from memory instead of hitting DB |
| Rate limiting | Atomic `INCR` operations with key expiry |
| Leaderboards | Sorted sets for real-time ranking |
| Pub/sub messaging | Fan-out events to multiple subscribers |

> **Typical caching pattern**: check Redis first → cache hit: return result → cache miss: query DB, store result in Redis with TTL, return result. This "cache-aside" pattern can reduce DB load by 80–90% for read-heavy workloads.

---

## 6.9 DynamoDB — Managed NoSQL

**DynamoDB** is AWS's fully managed key-value and document database — no servers to provision, no clusters to manage.

- **Single-digit millisecond latency** at any scale
- Automatically replicates data across 3 AZs
- **On-demand mode**: pay per read/write request — no capacity planning
- **Provisioned mode**: reserve capacity, cheaper at predictable scale
- Global Tables: multi-region, multi-active replication

```bash
# Create a table (partition key: userId)
aws dynamodb create-table \
  --table-name users \
  --attribute-definitions AttributeName=userId,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Put item
aws dynamodb put-item \
  --table-name users \
  --item '{
    "userId":  {"S": "u123"},
    "name":    {"S": "Alice"},
    "email":   {"S": "alice@example.com"},
    "created": {"N": "1700000000"}
  }'

# Get item by primary key
aws dynamodb get-item \
  --table-name users \
  --key '{"userId": {"S": "u123"}}'

# Query items (requires partition key)
aws dynamodb query \
  --table-name users \
  --key-condition-expression "userId = :uid" \
  --expression-attribute-values '{":uid": {"S": "u123"}}'
```

> **DynamoDB limitations**: no joins, no complex SQL queries, limited to queries by primary key or indexes. Design your access patterns first, then design your schema. This is the opposite of relational design.

---

## 6.10 Choosing the Right Database

| Need | Use |
|---|---|
| Relational data, SQL, ACID transactions | RDS PostgreSQL or MySQL |
| High-performance relational, large scale | Aurora PostgreSQL or MySQL |
| Auto-scaling relational, variable/low traffic | Aurora Serverless v2 |
| Key-value or document, low latency, massive scale | DynamoDB |
| Caching, sessions, pub/sub, rate limiting | ElastiCache Redis |
| Full-text search, log analysis | OpenSearch Service |
| Time-series metrics | Amazon Timestream |
| Graph relationships | Amazon Neptune |
| Data warehouse, analytics | Amazon Redshift |

> **Polyglot persistence**: most production systems use multiple database services. A typical application might use Aurora for the main relational data, DynamoDB for session tokens, ElastiCache for caching, and S3 for large objects. Choose the right tool for each access pattern rather than forcing everything into one database.

---

## Summary

- RDS removes the operational burden of managing database infrastructure — patches, backups, failover
- Launch RDS in private subnets within a DB Subnet Group spanning at least 2 AZs
- Multi-AZ provides automatic failover for HA; Read Replicas offload read traffic for performance
- Always use Secrets Manager for database credentials — never hardcode passwords
- Manual snapshots persist indefinitely and are essential before risky changes
- Aurora delivers higher performance and more replicas than standard RDS at higher cost
- ElastiCache Redis reduces database load dramatically for read-heavy workloads
- DynamoDB is the right choice when you need infinite scale and microsecond latency without schema constraints
- Match the database service to the access pattern — no single database fits all use cases

---

## Knowledge Check

1. What is the difference between Multi-AZ and Read Replicas? Can you use both at the same time?
2. You need to upgrade the PostgreSQL major version on a production RDS instance. What steps would you take to minimize risk and ensure you can roll back?
3. Your application queries the same product catalog data thousands of times per minute. The data changes only a few times per day. How would you reduce load on your RDS instance?
4. You are designing a system that stores user click events — billions of rows, no complex queries, just write-heavy with simple lookups by user ID. Which AWS database service is best suited and why?
5. An RDS instance fails. What happens with Multi-AZ enabled? How long is the expected downtime? What does your application need to do to reconnect?

---

## Hands-on Exercise

Practice end-to-end RDS operations:

1. Create a DB Subnet Group using your private subnets from Chapter 4
2. Create a security group `rds-sg` that allows inbound PostgreSQL (port 5432) from your app server security group only
3. Launch a free-tier RDS PostgreSQL instance (`db.t3.micro`) in private subnets with Multi-AZ disabled (free tier limitation) and 7-day backup retention
4. Connect to the instance via a bastion host EC2 in the public subnet using `psql`
5. Create a table, insert 3–5 rows of test data, and run a SELECT query
6. Take a manual snapshot named `my-db-snapshot-exercise`
7. Restore the snapshot to a new instance named `my-db-restored`
8. (Optional) Create a Read Replica and connect to it read-only from your application

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="05-s3.md">← Previous: S3 — Simple Storage Service</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="07-elb-and-autoscaling.md">Next: ELB & Auto Scaling →</a>
</div>
