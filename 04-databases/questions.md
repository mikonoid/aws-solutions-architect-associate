# 04 — Databases: practice questions

> Format: question → options → correct answer with why the others are wrong.
> Try to answer first, then expand the answer.

---

### Q1. A production RDS MySQL database must automatically fail over if its AZ fails, with no data loss. What do you enable?

- A. A read replica in another AZ
- B. Multi-AZ deployment
- C. Automated backups with 35-day retention
- D. Storage autoscaling

<details><summary>Answer</summary>

**B.** **Multi-AZ** keeps a synchronous standby in another AZ and fails over automatically by flipping DNS.

- A — async replication, manual promotion, possible data loss.
- C — restore creates a new instance; not automatic failover.
- D — unrelated to availability.
</details>

---

### Q2. A reporting workload slows down the primary RDS PostgreSQL database. Reports can tolerate a few seconds of lag. What do you do?

- A. Enable Multi-AZ and point reports to the standby
- B. Create a read replica and point reports to it
- C. Increase backup retention
- D. Move reports to DynamoDB

<details><summary>Answer</summary>

**B.** **Read replicas** offload read traffic asynchronously.

- A — a Multi-AZ instance standby is not readable.
- C — unrelated.
- D — requires redesign.
</details>

---

### Q3. An existing RDS instance is unencrypted. Security requires encryption at rest. What is the process?

- A. Modify the instance and enable encryption
- B. Create a read replica with encryption enabled
- C. Take a snapshot, copy it with encryption, restore a new instance from the encrypted snapshot, and switch the application
- D. Enable TLS on the connection

<details><summary>Answer</summary>

**C.** RDS encryption is set at creation. Encrypt via an encrypted snapshot copy and restore.

- A — not possible in place.
- B — you cannot create an encrypted replica from an unencrypted instance.
- D — encryption in transit, not at rest.
</details>

---

### Q4. A team accidentally deleted rows 20 minutes ago. Automated backups are enabled. What do you do?

- A. Restore the latest manual snapshot onto the existing instance
- B. Use point-in-time restore to a time before the deletion, creating a new instance
- C. Promote the Multi-AZ standby
- D. Roll back with DynamoDB PITR

<details><summary>Answer</summary>

**B.** **PITR** restores to any second in the retention window. Restores always create a **new** instance.

- A — snapshots never restore onto an existing instance, and may be older.
- C — the standby has the same deleted data (synchronous).
- D — wrong service.
</details>

---

### Q5. A MySQL-compatible database needs up to 15 low-lag read replicas, storage replicated 6 ways across 3 AZs, and failover usually under 30 seconds. What do you choose?

- A. RDS for MySQL Multi-AZ
- B. Amazon Aurora MySQL
- C. DynamoDB
- D. Redshift

<details><summary>Answer</summary>

**B.** **Aurora** has 6-copy storage across 3 AZs and up to 15 Aurora Replicas with fast failover.

- A — fewer, higher-lag replicas, different storage architecture.
- C — not relational/MySQL-compatible.
- D — analytics.
</details>

---

### Q6. A global application needs its PostgreSQL database readable in three regions with sub-second replication, and the ability to promote another region within about a minute during a regional outage. What do you use?

- A. RDS cross-region read replicas
- B. Aurora Global Database
- C. DynamoDB Global Tables
- D. Multi-AZ DB cluster

<details><summary>Answer</summary>

**B.** **Aurora Global Database** replicates storage across regions typically under 1 second, with RTO around a minute.

- A — works, but higher lag and slower, more manual DR.
- C — NoSQL, not PostgreSQL.
- D — one region only.
</details>

---

### Q7. Thousands of concurrent Lambda invocations exhaust RDS database connections. What is the best fix?

- A. Increase the Lambda timeout
- B. Put RDS Proxy between Lambda and the database
- C. Add a read replica
- D. Switch to Multi-AZ

<details><summary>Answer</summary>

**B.** **RDS Proxy** pools and reuses connections, and supports IAM auth and Secrets Manager.

- A — makes it worse.
- C — writes and connection count are still a problem.
- D — availability, not connections.
</details>

---

### Q8. An application has unpredictable database load: near zero most of the day, sudden spikes during events. It's PostgreSQL-compatible. What minimizes cost while handling spikes?

- A. RDS on a large provisioned instance
- B. Aurora Serverless v2
- C. Redshift Serverless
- D. ElastiCache

<details><summary>Answer</summary>

**B.** **Aurora Serverless v2** scales capacity up and down quickly with demand.

- A — pays for peak capacity all day.
- C — data warehouse, not OLTP.
- D — cache, not a primary database.
</details>

---

### Q9. Analysts need to run complex SQL aggregations over petabytes of historical sales data. Some of it lives in S3 and should not be loaded. What fits?

- A. RDS for PostgreSQL with read replicas
- B. Redshift with Redshift Spectrum
- C. DynamoDB with GSIs
- D. ElastiCache

<details><summary>Answer</summary>

**B.** **Redshift** is the columnar MPP data warehouse. **Spectrum** queries S3 directly.

- A — OLTP engine, not built for PB analytics.
- C — key-value, poor for ad-hoc aggregations.
- D — cache.
</details>

---

### Q10. A gaming app needs a serverless database with single-digit-millisecond latency at millions of requests per second, and traffic is unpredictable. What do you choose?

- A. RDS with provisioned IOPS
- B. DynamoDB in on-demand capacity mode
- C. Redshift
- D. Neptune

<details><summary>Answer</summary>

**B.** **DynamoDB On-Demand** scales automatically and charges per request.

- A — relational instance, not serverless at this scale.
- C — analytics.
- D — graph database.
</details>

---

### Q11. A DynamoDB table already exists. A new access pattern needs to query by a different partition key. What do you add?

- A. A Local Secondary Index
- B. A Global Secondary Index
- C. DynamoDB Streams
- D. DAX

<details><summary>Answer</summary>

**B.** A **GSI** can use a different partition key and can be added to an existing table.

- A — LSIs keep the same partition key and can only be created with the table.
- C — change log, not a query index.
- D — cache, not a new access pattern.
</details>

---

### Q12. A read-heavy DynamoDB application needs microsecond read latency with minimal code changes. What do you add?

- A. ElastiCache for Redis
- B. DynamoDB Accelerator (DAX)
- C. DynamoDB Streams
- D. Global Tables

<details><summary>Answer</summary>

**B.** **DAX** is API-compatible with DynamoDB and gives microsecond cached reads.

- A — works, but requires cache logic in the app.
- C/D — not caching.
</details>

---

### Q13. Users in Europe and the US must both read and write the same DynamoDB data with low latency, and the app must keep working if one region fails. What do you use?

- A. DynamoDB Streams to a Lambda copying items
- B. DynamoDB Global Tables
- C. DAX in two regions
- D. Read replicas

<details><summary>Answer</summary>

**B.** **Global Tables** are multi-region, multi-active replicas.

- A — custom, error-prone.
- C — cache, not replication.
- D — DynamoDB has no read replicas; that's RDS.
</details>

---

### Q14. Session data for a web fleet must be stored in memory, survive a cache node failure, and support sorted sets for a leaderboard. What do you choose?

- A. ElastiCache for Memcached
- B. ElastiCache for Redis OSS/Valkey with replication and Multi-AZ
- C. DAX
- D. S3

<details><summary>Answer</summary>

**B.** Redis/Valkey supports data structures like sorted sets, replication, and automatic failover.

- A — no replication or persistence; simple key/value.
- C — only for DynamoDB.
- D — not in-memory.
</details>

---

### Q15. A team wants Redis data structures, but as the durable primary database with Multi-AZ durability, not just a cache. What do you choose?

- A. ElastiCache for Memcached
- B. MemoryDB
- C. DAX
- D. Redshift

<details><summary>Answer</summary>

**B.** **MemoryDB** is Redis/Valkey-compatible with a durable Multi-AZ transaction log.

- A — cache without persistence.
- C — DynamoDB cache.
- D — data warehouse.
</details>

---

### Q16. An application stores product catalogs and needs full-text search with relevance ranking and fuzzy matching. What do you add?

- A. DynamoDB Scan
- B. OpenSearch Service
- C. Redshift Spectrum
- D. Timestream

<details><summary>Answer</summary>

**B.** **OpenSearch** is built for full-text search and analytics.

- A — expensive and no relevance ranking.
- C/D — wrong workloads.
</details>

---

### Q17. Match the workload: (1) MongoDB migration, (2) Cassandra migration, (3) fraud detection over relationships, (4) IoT sensor metrics over time. Which set is right?

- A. DocumentDB, Keyspaces, Neptune, Timestream
- B. DynamoDB, DocumentDB, Redshift, OpenSearch
- C. Keyspaces, DocumentDB, Timestream, Neptune
- D. Aurora, Keyspaces, ElastiCache, Timestream

<details><summary>Answer</summary>

**A.** **DocumentDB** (MongoDB-compatible), **Keyspaces** (Cassandra-compatible), **Neptune** (graph), **Timestream** (time series).
</details>

---

### Q18. A DynamoDB table stores temporary session items that should be deleted automatically 24 hours after creation, without consuming write capacity. What do you use?

- A. A Lambda cron job that scans and deletes items
- B. DynamoDB TTL on an expiration attribute
- C. DynamoDB Streams
- D. PITR

<details><summary>Answer</summary>

**B.** **TTL** deletes expired items in the background at no cost.

- A — costs capacity and adds code.
- C — records changes, doesn't delete.
- D — backup feature.
</details>
