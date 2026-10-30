# 04 — Databases

> The exam asks you to match a data model and access pattern to a purpose-built
> database, then make it highly available, scalable, and secure. Know RDS/Aurora
> HA vs read scaling, DynamoDB design and caching, and when to pick each niche engine.

---

## 1. RDS

Managed relational engines: **MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2**.
AWS handles provisioning, patching, backups, and failover. No OS access (except **RDS Custom** for Oracle/SQL Server).

### Multi-AZ vs Read Replicas

| | Multi-AZ (instance) | Read Replica |
|---|---|---|
| Purpose | **High availability / DR** | **Read scaling** |
| Replication | **Synchronous** | **Asynchronous** |
| Readable standby? | No (standby only) | Yes |
| Failover | Automatic, DNS endpoint flips (~1–2 min) | Manual promotion to standalone DB |
| Location | Another AZ, same region | Same AZ, cross-AZ, or **cross-region** |
| Count | 1 standby | Up to 15 (MySQL, MariaDB, PostgreSQL); fewer for others |

- **Multi-AZ DB cluster** (MySQL/PostgreSQL): one writer + **two readable standbys** in 3 AZs, faster failover.
- A read replica can itself be Multi-AZ. Cross-region replicas help with DR and local reads.
- Converting Single-AZ to Multi-AZ is an online modification (snapshot and restore in the background).

> Gotcha: "Improve read performance" → read replicas. "Survive an AZ failure automatically" → Multi-AZ.

### Backups and restore

- **Automated backups**: daily snapshot + transaction logs, retention **0–35 days**, **point-in-time restore** (to the second, typically last 5 minutes).
- **Manual snapshots**: kept until you delete them. Can copy cross-region/account.
- **Restore always creates a new DB instance** (new endpoint).

### Security and operations

- Runs in your VPC. Control with **security groups**; put DBs in private subnets.
- **Encryption at rest** (KMS) must be chosen at creation. To encrypt an existing DB: snapshot → copy with encryption → restore.
  Read replicas of an encrypted DB are encrypted with the same/compatible key.
- **In transit**: TLS.
- **IAM database authentication** (MySQL, PostgreSQL) — tokens instead of passwords.
- **Secrets Manager** integration for password rotation.
- **Storage autoscaling** grows storage automatically when near full.
- **Performance Insights** and Enhanced Monitoring for diagnosis.

---

## 2. Aurora

AWS-built MySQL- and PostgreSQL-compatible engine.

- Storage: **6 copies across 3 AZs**, auto-grows up to 128 TiB, self-healing.
  Tolerates loss of 2 copies for writes and 3 for reads.
- **Up to 15 Aurora Replicas** with low lag. Any replica can be promoted on failover (usually < 30 s).
- Endpoints:
  - **Cluster (writer) endpoint** — always the primary.
  - **Reader endpoint** — load-balances across replicas.
  - **Custom endpoints** — a subset of replicas (e.g., analytics).
- **Aurora Global Database** — one primary region + up to 10 read-only secondary regions, replication typically **< 1 second**. Promote a secondary for DR (RPO ~1 s, RTO ~1 min).
- **Aurora Serverless v2** — capacity scales in fine increments (ACUs) for variable/unpredictable workloads; can mix with provisioned instances.
- **Backtrack** (Aurora MySQL) — rewind the DB in place without restoring a snapshot.
- **Cloning** — fast copy-on-write clone for test environments.
- Aurora I/O-Optimized for I/O-heavy workloads (predictable pricing).

> Note: "MySQL/PostgreSQL, highest availability and performance, many replicas, cross-region DR in seconds" → **Aurora / Aurora Global Database**.

---

## 3. RDS Proxy

- Fully managed **connection pooler** for RDS and Aurora.
- Solves "too many connections" from **Lambda** or bursty apps by sharing a warm pool.
- Reduces failover time (up to ~66%) by keeping client connections and routing to the new primary.
- Enforces **IAM authentication** and reads DB credentials from **Secrets Manager**.
- Runs only inside the VPC (not publicly accessible).

---

## 4. Redshift

- Managed **data warehouse** for **OLAP**: columnar storage, massively parallel processing (MPP), SQL.
- Cluster = **leader node** (planning, aggregation) + **compute nodes**. RA3 nodes separate compute from managed storage.
- **Redshift Spectrum** — query data directly in **S3** without loading it.
- Snapshots to S3; automated **cross-region snapshot copy** for DR.
- **Redshift Serverless** — no cluster to size; pay for RPU-hours when queries run.
- Data loading: `COPY` from S3, streaming ingestion from Kinesis/MSK, zero-ETL from Aurora.

> Gotcha: Redshift is for analytics over huge datasets, not for OLTP transactions.

---

## 5. DynamoDB

Serverless **key-value / document** NoSQL. Single-digit-millisecond latency at any scale. Multi-AZ by default.

- **Primary key**: partition key, or partition key + sort key.
- Max item size **400 KB** (put large blobs in S3 and store a pointer).
- Indexes:
  - **LSI** — same partition key, different sort key. **Created only at table creation.** Shares capacity with the table.
  - **GSI** — different partition/sort key. Can be added any time. Has its own capacity. Eventually consistent only.
- Capacity modes:
  - **On-Demand** — pay per request, for unpredictable/spiky traffic.
  - **Provisioned** — set RCU/WCU, add **auto scaling**; cheaper for predictable traffic.
  - 1 RCU = one strongly consistent 4 KB read/s (or two eventually consistent). 1 WCU = one 1 KB write/s.
- **DynamoDB Streams** — ordered item-change log (24 h) → Lambda triggers, replication, auditing.
- **Global Tables** — multi-region, **multi-active** (read/write in every region) replication.
- **TTL** — auto-delete expired items at no cost.
- **PITR** — continuous backups, restore to any second in the last **35 days**. On-demand backups too.
- **Transactions** — ACID across multiple items/tables.
- Export to S3 (no impact on table capacity) for analytics.

### DAX — DynamoDB Accelerator

- In-memory, write-through cache in front of DynamoDB. Read latency from ms to **microseconds**.
- **API-compatible** with DynamoDB — minimal code changes.
- Best for read-heavy, hot keys. Not for write-heavy or strongly consistent reads (those go to the table).

> Gotcha: "Microsecond reads from DynamoDB with minimal code changes" → **DAX**, not ElastiCache.

---

## 6. ElastiCache and MemoryDB

### ElastiCache (Redis OSS / Valkey / Memcached)

| | Redis OSS / Valkey | Memcached |
|---|---|---|
| Data structures | strings, hashes, lists, sets, sorted sets, streams | simple key/value |
| Persistence / backup | yes (snapshots) | no |
| Replication / Multi-AZ failover | yes | no |
| Multi-threaded | mostly single-threaded per shard | **multi-threaded** |
| Typical use | sessions, leaderboards, pub/sub, caching with HA | simple, horizontally scaled cache |

- Caching patterns: **lazy loading** (cache-aside, stale data possible) and **write-through** (always fresh, more writes). Use TTLs.
- Requires app code changes (unlike DAX).
- Redis AUTH/IAM auth, TLS in transit, encryption at rest.

### MemoryDB

- **Durable** Redis/Valkey-compatible **primary database** with a Multi-AZ transaction log.
- Microsecond reads, single-digit-millisecond writes.
- Choose when you want Redis data structures **as the system of record**, not just a cache.

---

## 7. OpenSearch Service

- Managed OpenSearch (Elasticsearch fork) + Dashboards.
- **Full-text search**, log analytics, observability, vector search.
- Common pipelines: CloudWatch Logs / Kinesis Firehose → OpenSearch; DynamoDB → OpenSearch for search on items.
- Serverless option available. Multi-AZ domains with dedicated master nodes for HA.

---

## 8. Other purpose-built databases

| Service | Model / compatibility | Choose when |
|---|---|---|
| **DocumentDB** | Document, **MongoDB-compatible** | Migrate MongoDB workloads; JSON documents. Storage like Aurora (6 copies, 3 AZs) |
| **Keyspaces** | Wide-column, **Apache Cassandra-compatible** (CQL) | Migrate Cassandra; serverless |
| **Neptune** | **Graph** (Gremlin, openCypher, SPARQL) | Social networks, recommendations, fraud detection, knowledge graphs |
| **Timestream** | **Time series** | IoT telemetry, DevOps metrics, built-in time-series functions and tiered storage |
| **QLDB** | Immutable, cryptographically verifiable **ledger** | **Retired (July 2025)**. Exams may still mention it for "verifiable history of changes"; AWS suggests Aurora PostgreSQL with audit tooling instead |

---

## 9. Quick picker

| Need | Choose |
|---|---|
| Relational, standard engine, managed | RDS |
| Relational, highest availability and performance | Aurora |
| Relational with unpredictable load | Aurora Serverless v2 |
| Cross-region DR in seconds, MySQL/PostgreSQL | Aurora Global Database |
| Too many DB connections from Lambda | RDS Proxy |
| Analytics / BI over TB–PB, SQL | Redshift (Serverless for spiky use) |
| Key-value at massive scale, serverless | DynamoDB |
| Multi-region active-active NoSQL | DynamoDB Global Tables |
| Microsecond DynamoDB reads | DAX |
| Cache in front of RDS / sessions | ElastiCache |
| Durable Redis as primary DB | MemoryDB |
| Full-text search / log analytics | OpenSearch |
| MongoDB | DocumentDB |
| Cassandra | Keyspaces |
| Relationships / graph traversal | Neptune |
| Time-series metrics | Timestream |

## 10. Quick recap

- Multi-AZ = sync HA, not readable (instance). Read replicas = async read scaling, can be cross-region.
- RDS automated backups 0–35 days, PITR; restore = new instance. Encrypt at creation.
- Aurora: 6 copies / 3 AZs, 15 replicas, reader endpoint, Global Database < 1 s lag.
- RDS Proxy = connection pooling + faster failover, great for Lambda.
- DynamoDB: 400 KB items, LSI at creation only, GSI any time, Streams, Global Tables, TTL, PITR 35 days.
- DAX = microseconds for DynamoDB, no code rewrite. ElastiCache = cache for everything else.
- Redshift = OLAP; Spectrum queries S3.

## 11. Questions

See [`questions.md`](questions.md)
