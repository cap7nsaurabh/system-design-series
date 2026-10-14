# 🗃️ Databases for System Design — The Foundation of Everything

> *"The database is not just storage; it is the single source of truth that determines the correctness of your entire system."*

---

## 📋 Table of Contents

1. [Database Categories Overview](#overview)
2. [Relational Databases (RDBMS)](#rdbms)
3. [ACID Properties Deep Dive](#acid)
4. [Transaction Isolation Levels](#isolation-levels)
5. [Indexes & B+ Trees](#indexes)
6. [NoSQL Databases](#nosql)
7. [CAP Theorem](#cap-theorem)
8. [PACELC Theorem](#pacelc)
9. [SQL vs NoSQL Decision Matrix](#sql-vs-nosql)
10. [Database Sharding](#sharding)
11. [Database Replication](#replication)
12. [Connection Pooling](#connection-pooling)
13. [N+1 Query Problem](#n-plus-1)
14. [Time-Series & Search Databases](#specialized)
15. [NewSQL](#newsql)
16. [Real-Life Scenario](#real-life-scenario)
17. [🔥 Big Tech Story: Discord's Database Migration](#big-tech-story)
18. [Trade-offs & Drawbacks](#trade-offs)
19. [Best Practices](#best-practices)
20. [Key Takeaways](#key-takeaways)

---

## 🗺️ Database Categories Overview

```
                        DATABASES
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
      Relational          NoSQL           NewSQL
      (RDBMS)                           (Best of both)
          │                 │
   MySQL, PostgreSQL,   ┌───┼───┬──────────┐
   Oracle, SQL Server   ▼   ▼   ▼          ▼
                     Key- Document Wide-  Graph
                     Value  DB   Column    DB
                       │    │    │          │
                    Redis MongoDB Cassandra Neo4j
                    DynamoDB     HBase
                             Firestore
```

Each category is optimized for different access patterns and consistency requirements.

---

## 🏛️ Relational Databases (RDBMS)

Relational databases organize data into **tables** (relations) with predefined **schemas**. Rows represent records; columns represent attributes. Tables relate to each other through **foreign keys**.

SQL (Structured Query Language) is the universal interface.

```sql
-- Example: E-commerce schema
CREATE TABLE users (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    email       VARCHAR(255) UNIQUE NOT NULL,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    total       DECIMAL(10,2) NOT NULL,
    status      ENUM('pending','paid','shipped','delivered'),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);
```

**Key strengths:**
- Strict schema ensures data integrity
- Powerful query language (JOINs, aggregations, subqueries)
- ACID transactions for correctness
- Mature ecosystem and tooling

**Popular RDBMS:**
| Database | Best For | Notable Users |
|---|---|---|
| PostgreSQL | General purpose, complex queries, extensibility | Instagram, Shopify, Notion |
| MySQL | Web applications, simple queries | Facebook (MySQL at scale), Twitter, Airbnb |
| Oracle | Enterprise, large-scale OLTP | Banks, telecoms |
| SQL Server | .NET ecosystem, Windows | Enterprises |
| SQLite | Embedded, mobile, testing | iOS apps, Electron apps |

---

## ⚗️ ACID Properties Deep Dive

ACID is the set of properties that guarantee database transactions are processed reliably.

### A — Atomicity

A transaction is **all or nothing**. Either all operations in the transaction succeed, or none of them do. There is no partial transaction.

**Example: Bank transfer**
```sql
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- Debit Alice
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- Credit Bob
COMMIT;
```

If the server crashes between the two UPDATE statements, atomicity ensures **neither** update is applied (rolled back on recovery). Alice's money doesn't disappear.

**How it's implemented:** Write-Ahead Log (WAL). Every change is written to the log before the actual data pages. On crash recovery, the database replays or rolls back based on the WAL.

---

### C — Consistency

A transaction brings the database from one **valid state** to another valid state. All data integrity constraints are satisfied before and after the transaction.

**Examples of constraints:**
- `NOT NULL`: a required field must not be empty
- `UNIQUE`: no two rows can have the same email
- `FOREIGN KEY`: order.user_id must reference an existing user
- `CHECK`: balance must be >= 0

```sql
CREATE TABLE accounts (
    id      BIGINT PRIMARY KEY,
    balance DECIMAL(10,2) CHECK (balance >= 0)  -- Consistency constraint
);

BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;  -- balance becomes -500
    -- CONSTRAINT CHECK FAILS: balance cannot be negative
ROLLBACK;  -- Transaction aborted; consistency maintained
```

**Note:** Consistency in ACID is different from Consistency in CAP theorem (common confusion!). ACID consistency = data integrity constraints. CAP consistency = all nodes see the same data at the same time.

---

### I — Isolation

Concurrent transactions execute **as if they were sequential**. One transaction's intermediate states are not visible to other transactions.

**Without isolation:**
```
Transaction A (reads balance, then updates):  Read: $1000 → Update: $1100
Transaction B (reads balance, then updates):  Read: $1000 → Update: $1200
Expected result: $1300. Actual result: $1200 (lost update!)
```

**Isolation levels** control the degree of visibility between concurrent transactions. More on this in the next section.

**How it's implemented:** Locks (pessimistic) or Multi-Version Concurrency Control (MVCC — optimistic). PostgreSQL and MySQL InnoDB use MVCC — readers don't block writers, writers don't block readers.

---

### D — Durability

Once a transaction is committed, it **remains committed** even in the event of a system failure (power loss, crash, OS error).

**How it's implemented:**
- Write to WAL (Write-Ahead Log) on disk before committing
- On crash recovery, replay committed WAL entries
- `fsync()` system call ensures data is on physical disk, not just OS buffer

**Tradeoff:** `fsync()` is slow. Some databases offer `synchronous_commit = off` (PostgreSQL) which sacrifices durability for performance — committed data can be lost in a crash. Use only when you can tolerate data loss (e.g., analytics counters).

---

## 🔐 Transaction Isolation Levels

SQL standard defines four isolation levels. They trade consistency for concurrency. Higher isolation = fewer anomalies = lower concurrency.

### Anomalies Explained First

**Dirty Read:** Reading data written by another transaction that hasn't committed yet. The other transaction might roll back, leaving you with phantom data.

```
T1: UPDATE orders SET status = 'shipped' WHERE id = 1;  -- not committed
T2: SELECT status FROM orders WHERE id = 1;  → 'shipped' (dirty read!)
T1: ROLLBACK;
-- T2 saw data that never officially existed
```

**Non-Repeatable Read:** Reading the same row twice and getting different values because another committed transaction modified it between your reads.

```
T1: SELECT price FROM products WHERE id = 5;  → $100
T2: UPDATE products SET price = $150 WHERE id = 5; COMMIT;
T1: SELECT price FROM products WHERE id = 5;  → $150 (different!)
```

**Phantom Read:** Running the same range query twice and getting different rows because another committed transaction inserted/deleted rows.

```
T1: SELECT COUNT(*) FROM orders WHERE user_id = 1;  → 5 orders
T2: INSERT INTO orders (user_id, ...) VALUES (1, ...); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE user_id = 1;  → 6 orders (phantom!)
```

### The Four Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| Read Uncommitted | ✅ Possible | ✅ Possible | ✅ Possible | Highest |
| Read Committed | ❌ Prevented | ✅ Possible | ✅ Possible | High |
| Repeatable Read | ❌ Prevented | ❌ Prevented | ✅ Possible | Medium |
| Serializable | ❌ Prevented | ❌ Prevented | ❌ Prevented | Lowest |

**Read Uncommitted:** Almost never used. Can read dirty (uncommitted) data. No practical use case.

**Read Committed:** Default in PostgreSQL and Oracle. Each read within a transaction sees the latest committed data. Most applications work fine here.

**Repeatable Read:** Default in MySQL InnoDB. Guarantees that if you read a row twice in the same transaction, you get the same value. MySQL's implementation also prevents phantom reads (via gap locks).

**Serializable:** Full isolation. Transactions behave as if run one at a time. Implemented via serializable snapshot isolation (SSI) in PostgreSQL. Use when correctness is critical (financial systems). Significant performance cost.

**Production recommendation:**
- Most applications: **Read Committed**
- Financial transactions, inventory management: **Repeatable Read** or **Serializable**
- Analytics (doesn't need isolation): can use **Read Uncommitted** or a replica

---

## 🌳 Indexes & B+ Trees

An index is a data structure that allows fast lookup of rows without scanning the entire table.

### Without Index (Full Table Scan)

```sql
SELECT * FROM users WHERE email = 'alice@example.com';
-- Without index: scan ALL rows comparing email. O(n)
-- 10 million users → 10 million comparisons
```

### With Index (B+ Tree Lookup)

```sql
CREATE INDEX idx_users_email ON users(email);
-- With index: B+ tree traversal. O(log n)
-- 10 million users → ~24 comparisons (log2(10M) ≈ 24)
```

### B+ Tree Structure

```
                    [40 | 70]              ← Root (internal node)
                   /    |    \
            [20|30]  [50|60]  [80|90]     ← Internal nodes
           /  |  \   /  |  \   /  |  \
         [10][25][35][45][55][65][75][85][95]  ← Leaf nodes (actual data)
          ↕   ↕   ↕   ↕   ↕   ↕   ↕   ↕   ↕
        ←──────────── linked list ──────────────→
```

**Key properties of B+ Trees:**
- All data is in **leaf nodes** (internal nodes only store keys for routing)
- Leaf nodes are connected as a **doubly linked list** → range scans are fast
- Tree is always **balanced** → O(log n) for all operations
- Each node is typically the size of a disk page (4-16KB) → minimizes disk I/O

**Why not use a hash index?**
- Hash index: O(1) for exact lookups, but **cannot do range queries** (`WHERE age BETWEEN 20 AND 30`)
- B+ Tree: O(log n) for exact lookups, **supports range queries** → B+ Tree wins for general purpose

### Types of Indexes

**Primary Index (Clustered Index)**
- The table data IS the index. Rows are physically sorted by the primary key.
- In MySQL InnoDB: every table has a clustered index. If no PK defined, InnoDB creates a hidden one.
- Only one per table (data can only be sorted one way)
- Range queries on PK are extremely fast (rows are adjacent on disk)

**Secondary Index (Non-Clustered)**
- Separate data structure pointing to primary keys
- Multiple allowed per table
- Lookup: secondary index → get PK → primary index → get row (two lookups)
- In InnoDB: secondary index leaves contain the PK value, not the row pointer

**Composite Index**
```sql
CREATE INDEX idx_name_email ON users(last_name, first_name, email);
-- Efficient for:
--   WHERE last_name = 'Smith'
--   WHERE last_name = 'Smith' AND first_name = 'John'
--   WHERE last_name = 'Smith' AND first_name = 'John' AND email = '...'
-- NOT efficient for:
--   WHERE first_name = 'John' (doesn't start with leftmost column)
```

Rule: **Left-prefix rule** — a composite index can only be used for queries that match a prefix of its columns.

**Covering Index**
If all columns needed by a query are in the index, the database can answer the query from the index alone (no need to read the actual row). Zero additional disk I/O.

```sql
CREATE INDEX idx_covering ON orders(user_id, status, created_at);
SELECT status, created_at FROM orders WHERE user_id = 123;
-- All needed columns (user_id, status, created_at) are in the index → index-only scan
```

**Index Best Practices:**
- Index columns used in `WHERE`, `JOIN ON`, `ORDER BY`, `GROUP BY`
- Don't over-index: each index slows writes (must update index on every insert/update/delete)
- Use `EXPLAIN` to verify indexes are being used
- Composite index column order matters: most selective columns first

---

## 🍃 NoSQL Databases

NoSQL ("Not Only SQL") databases sacrifice some relational features (strict schema, JOINs, ACID) for:
- Horizontal scalability
- Schema flexibility
- High write throughput
- Specialized data models

### 1. Key-Value Stores

The simplest data model: a hash map. Given a key, get a value.

**Examples:** Redis, DynamoDB, Amazon ElastiCache, Memcached

```
SET user:123 → {"name": "Alice", "email": "alice@ex.com"}
GET user:123 → {"name": "Alice", "email": "alice@ex.com"}
```

**Use cases:** Session storage, caching, shopping carts, real-time leaderboards
**Not suitable for:** Complex queries, relationships, partial updates

**DynamoDB internals:**
- Fully managed by AWS — no servers to manage
- Single-digit millisecond latency at any scale
- Every item has a **Partition Key** (determines which shard stores the item)
- Optional **Sort Key** (allows range queries within a partition)
- Global Secondary Indexes (GSI) for alternative access patterns
- On-demand or provisioned capacity modes
- DynamoDB Streams for change data capture

---

### 2. Document Databases

Stores data as JSON-like documents. Each document is self-contained.

**Examples:** MongoDB, Firestore, CouchDB, Amazon DocumentDB

```json
{
  "_id": "order_456",
  "user_id": "user_123",
  "items": [
    {"product_id": "prod_789", "quantity": 2, "price": 29.99},
    {"product_id": "prod_101", "quantity": 1, "price": 49.99}
  ],
  "shipping": {
    "address": "123 Main St",
    "city": "New York"
  },
  "total": 109.97,
  "status": "shipped"
}
```

**Benefits:**
- Schema flexibility (documents can have different fields)
- Embedded documents avoid JOINs for related data
- Natural fit for hierarchical data (order with line items)

**MongoDB internals:**
- Documents stored in BSON (Binary JSON)
- Indexed by `_id` (ObjectId by default)
- Supports rich query language, aggregation pipeline
- WiredTiger storage engine: MVCC for concurrency, compression
- Sharding: range-based or hash-based on shard key
- Replica sets for HA (one primary + multiple secondaries)

**Use cases:** E-commerce products (variable attributes), content management, user profiles, catalogs
**Not suitable for:** Data requiring complex JOIN-heavy queries, strong transactional consistency across documents

---

### 3. Wide-Column Stores

Stores data in tables, rows, and dynamic columns. Each row can have different columns. Optimized for massive write throughput and time-series-like data.

**Examples:** Apache Cassandra, HBase, Google Bigtable (original)

**Cassandra data model:**
```
Table: user_activities
Partition Key: user_id
Clustering Key: activity_time (DESC)

user_id  | activity_time        | action  | details
---------|---------------------|---------|--------
user_123 | 2024-01-15 10:00:00 | view    | product_789
user_123 | 2024-01-15 09:55:00 | search  | "running shoes"
user_123 | 2024-01-15 09:50:00 | click   | ad_456
user_456 | 2024-01-15 10:01:00 | purchase| order_789
```

**Cassandra internals:**
- **Consistent hashing** ring for distributing partitions across nodes
- **Tunable consistency:** `ONE`, `QUORUM`, `ALL` for reads and writes
- **Leaderless replication:** any node can accept writes (no single master)
- **LSM Tree storage engine:** writes go to memtable first → flushed to SSTables
- **Compaction** merges SSTables and removes tombstones (deleted data)
- **Replication factor:** copies of each partition (typically 3)

**Cassandra write path:**
```
Write Request → Commit Log (WAL) → Memtable (in-memory)
                                        ↓ (when full)
                                    SSTable (on disk)
                                        ↓ (periodically)
                                    Compaction
```

**Use cases:** Time-series data (IoT sensor readings, user activity), write-heavy workloads (100K+ writes/sec), global distributed data
**Not suitable for:** Complex queries, ad-hoc analytics, strong consistency requirements

---

### 4. Graph Databases

Models data as nodes (entities) and edges (relationships). Optimized for traversing connections.

**Examples:** Neo4j, Amazon Neptune, JanusGraph

```cypher
-- Neo4j Cypher query
MATCH (alice:User {name: "Alice"})-[:FOLLOWS]->(friend)-[:LIKES]->(post:Post)
WHERE post.created_at > datetime() - duration('P7D')
RETURN post.title, COUNT(*) AS mutual_likes
ORDER BY mutual_likes DESC
LIMIT 10
```

This query (find posts liked by Alice's friends in the last week) would require multiple expensive JOINs in SQL. In Neo4j, it's a natural graph traversal.

**Use cases:** Social networks (friend-of-friend queries), recommendation engines, fraud detection (unusual connection patterns), knowledge graphs
**Not suitable for:** Simple CRUD, bulk data processing

---

## 🔺 CAP Theorem

The CAP theorem (Brewer's theorem, 2000) states that a distributed data store can guarantee **at most two** of the following three properties simultaneously:

**C — Consistency:** Every read receives the most recent write or an error. All nodes see the same data at the same time.

**A — Availability:** Every request receives a non-error response (but it might not be the most recent data).

**P — Partition Tolerance:** The system continues to operate even when network messages between nodes are delayed or dropped.

> **Critical insight:** In any real distributed system, **network partitions WILL happen** (cables fail, network switches hiccup). Therefore, **P is not optional**. The real choice is: when a partition occurs, do you sacrifice **C** (remain available but possibly return stale data) or **A** (return errors to ensure data is current)?

```
           Consistency
               ▲
               │
            CA │  CP
               │
      ─────────┼─────────
               │
            AP │
               │
               └──────────► Availability
           Partition Tolerance (always required)
```

**Real database examples:**

| Database | CAP Classification | Behavior During Partition |
|---|---|---|
| PostgreSQL (single node) | CA | Not distributed; no partition |
| MySQL with sync replication | CP | Rejects writes until partition heals |
| Cassandra (default) | AP | Accepts writes; may return stale reads |
| HBase | CP | Halts until ZooKeeper quorum restored |
| DynamoDB (eventual) | AP | Remains available; eventually consistent |
| DynamoDB (strong) | CP | Consistent reads; may timeout during partition |
| CockroachDB | CP | Sacrifices availability for consistency |
| Zookeeper | CP | Requires quorum for writes |

**Real implications:**
- **Banking (transfers):** Choose CP. You'd rather get an error than incorrectly debit/credit an account.
- **Social media (likes, views):** Choose AP. Showing 9,998 instead of 10,000 likes for a few seconds is acceptable.
- **Shopping cart:** Typically AP. Amazon famously chose AP for carts — it's better to let users add items than show an error.
- **Inventory (last unit):** CP — overselling is catastrophic.

---

## 📊 PACELC Theorem

CAP only describes behavior during a partition. PACELC extends it to normal operation:

- **PAC:** During a Partition, choose between Availability and Consistency (CAP's insight)
- **ELC:** Even when the system is running normally (Else), there's a tradeoff between Latency and Consistency

A system optimizing for **consistency** must synchronously replicate data before acknowledging writes → **higher latency**.
A system optimizing for **latency** acknowledges writes before full replication → **lower consistency**.

| System | P: A or C | E: L or C |
|---|---|---|
| DynamoDB | A | L (eventual) |
| Cassandra | A | L |
| MongoDB | C | L (with strong read concern) |
| CockroachDB | C | C |
| PostgreSQL | C | C |
| Riak | A | L |

---

## 🤔 SQL vs NoSQL Decision Matrix

| Factor | Choose SQL | Choose NoSQL |
|---|---|---|
| **Data structure** | Structured, tabular, relational | Unstructured, hierarchical, flexible |
| **Query patterns** | Complex JOINs, aggregations, ad-hoc queries | Simple lookups by key, specific access patterns |
| **Consistency** | Strong ACID required | Eventual consistency acceptable |
| **Scale** | Vertical scale, moderate horizontal | Massive horizontal scale (PB of data) |
| **Write throughput** | Moderate (10K-100K writes/sec) | Extreme (100K-1M+ writes/sec) |
| **Schema changes** | Infrequent, well-defined schema | Frequent schema evolution needed |
| **Team expertise** | SQL skills available | NoSQL expertise available |
| **Use cases** | Financial systems, ERP, e-commerce core | IoT telemetry, user activity logs, catalogs, social graphs |

**Golden rule:** Start with PostgreSQL. It's extremely capable, supports JSON columns for flexibility, and scales remarkably well. Move to NoSQL only when you have a specific, proven need.

---

## 🔀 Database Sharding

Sharding is **horizontal partitioning** — splitting data across multiple database instances (shards), each owning a subset.

### Why Shard?

A single database server has limits:
- Storage: ~10–50TB practical limit
- Write throughput: ~100K writes/second
- Connections: ~5K-10K concurrent

When your data or traffic exceeds these, you shard.

### Sharding Strategies

**Range-Based Sharding**
```
Shard 1: user_id 1       – 1,000,000
Shard 2: user_id 1000001 – 2,000,000
Shard 3: user_id 2000001 – 3,000,000
```

- ✅ Simple, predictable, easy range queries
- ❌ Hot spots: newly registered users all go to the last shard
- ❌ Resharding requires moving large ranges

**Hash-Based Sharding**
```
shard = hash(user_id) % num_shards

hash("user_123") % 3 = 1 → Shard 1
hash("user_456") % 3 = 2 → Shard 2
hash("user_789") % 3 = 0 → Shard 0
```

- ✅ Even distribution
- ❌ Range queries require hitting all shards
- ❌ Adding shards requires rehashing ALL data

**Consistent Hashing**
The solution to rehashing on scale changes. Place shards on a virtual ring (0 to 2^32). Each key is also mapped to the ring. A key is owned by the nearest clockwise shard.

```
        Virtual ring (0 to 2^32)

        Shard C (hash=80)
              ●
    ●                   ●
  Shard B           Shard A
  (hash=40)         (hash=10)
              ●
           Key K
           (hash=60)
           → Goes to Shard C (next clockwise)
```

Adding a new shard: only keys between the new shard and its predecessor need to move. Typically only `1/n` fraction of keys move when adding the nth shard.

**Virtual nodes** (vnodes): Each physical shard is responsible for multiple positions on the ring (e.g., 150 virtual nodes per shard). This prevents uneven distribution when shards have different capacities.

### Shard Key Selection — Critical Decision

The shard key determines data distribution and query patterns.

**Bad shard key: `created_at` (timestamp)**
- All writes go to the "current time" shard → hot spot
- Historical queries span many shards

**Good shard key: `user_id` (hash)**
- Writes distributed across all shards
- User-centric queries hit exactly one shard

**Considerations:**
- Choose a key with high cardinality (many distinct values)
- Choose a key that distributes writes evenly
- Choose a key that keeps related data on the same shard (avoid cross-shard JOINs)
- Cross-shard transactions are extremely complex — design data model to avoid them

### Cross-Shard Queries — The Nightmare

```sql
-- Easy on a single DB, nightmare on shards:
SELECT u.name, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'US'
GROUP BY u.id
HAVING order_count > 5;
```

If users and orders are on different shards (or different users are on different shards), this requires:
1. Query all user shards for US users
2. Query all order shards for each user
3. Application-level aggregation

This is why **denormalization and data co-location** are critical in sharded systems.

### Resharding

When traffic grows further, you need to split existing shards. This is painful:
1. Start new shards
2. Backfill data from old shards to new shards
3. Switch writes to new sharding scheme
4. Clean up old data

Companies build complex internal tools for this. Vitess (MySQL shard management) automates much of it.

---

## 📡 Database Replication

Replication copies data from one database node (primary) to others (replicas/secondaries).

### Master-Slave (Primary-Replica) Replication

```
                 ┌──────────┐
  Writes ───────►│  Master  │
                 └────┬─────┘
                      │ Replicate (binlog/WAL shipping)
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      Replica 1   Replica 2   Replica 3
      ← Reads →   ← Reads →   ← Reads →
```

- All writes go to master
- Master's changes are replicated to replicas (via binary log in MySQL, WAL in PostgreSQL)
- Reads can be distributed across replicas
- If master fails → manual or automatic promotion of a replica

**Synchronous replication:** Master waits for at least one replica to confirm before acknowledging write. Zero data loss but higher write latency.

**Asynchronous replication (default):** Master acknowledges write immediately. Replica may lag by seconds. Data loss possible if master crashes before replica catches up. Used by most systems for performance.

**Semi-synchronous replication (MySQL):** Wait for ONE replica to confirm receipt (not apply). Compromise between safety and performance.

### Master-Master (Multi-Primary) Replication

Both nodes accept writes. Changes replicated bidirectionally.

```
      Writes          Writes
         │                │
    ┌────▼─────┐    ┌─────▼────┐
    │  Node A  │◄──►│  Node B  │
    └──────────┘    └──────────┘
```

**Problem:** Conflicts. Both nodes modify the same row simultaneously. Who wins?

**Conflict resolution strategies:**
- Last-Write-Wins (timestamp-based) — simplest, but clock skew can cause issues
- Application-defined resolution (e.g., merge logic)
- Avoid writes to the same rows on different nodes (geographical partitioning)

**Use cases:** Multi-region writes where users are geographically partitioned (US users write to US node, EU users write to EU node, minimal overlap). Geographic databases like Shopify's use this pattern.

### Replication Lag

Even async replication introduces lag. During high write load, replicas can fall behind by seconds to minutes.

**Problem:** If you write data and immediately read from a replica, you might get stale data.

```
Write to Master: user.email = "new@example.com"
(Replication lag: 200ms)
Read from Replica: user.email = "old@example.com" ← Stale!
```

**Solutions:**
- Read from master for "read-your-writes" consistency (e.g., immediately after user makes a change)
- Track replication position; wait for replica to catch up before reading
- Use synchronous replication for critical data

---

## 🔌 Connection Pooling

Each database connection is expensive:
- Memory: ~5MB per connection in PostgreSQL
- TCP overhead
- Authentication overhead

With 1,000 application server threads each wanting a DB connection, you'd need 1,000 DB connections. PostgreSQL struggles with more than ~500 connections.

**Solution: Connection Pool** — a pool of pre-opened connections shared across threads.

```
1000 App Threads ──► [ Connection Pool: 50 connections ] ──► PostgreSQL
                     (Threads wait if all 50 are busy)
```

**PgBouncer (PostgreSQL connection pooler):**
```
Modes:
  Session pooling:    1 DB connection per client session
  Transaction pooling: 1 DB connection shared for duration of transaction
  Statement pooling:  1 DB connection per SQL statement (most efficient, no prepared statements)
```

PgBouncer transaction mode: 10,000 app connections → 100 actual DB connections. Dramatic reduction.

**RDS Proxy (AWS):** Managed connection pooling for Aurora/RDS. Automatically pools connections from Lambda functions (which don't support persistent connections well).

---

## 🔍 N+1 Query Problem

One of the most common performance anti-patterns.

**The problem:**
```python
# Get all orders and their user names
orders = db.query("SELECT * FROM orders LIMIT 100")  # 1 query

for order in orders:  # 100 iterations
    user = db.query(f"SELECT * FROM users WHERE id = {order.user_id}")  # 1 query each!
    print(f"Order {order.id} by {user.name}")

# Total: 1 + 100 = 101 queries! (N+1)
```

**Solution 1: SQL JOIN**
```sql
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
LIMIT 100;
-- 1 query total ✓
```

**Solution 2: Eager Loading (ORM)**
```python
# Django ORM
orders = Order.objects.select_related('user').all()[:100]
# Generates: SELECT orders.*, users.* FROM orders JOIN users ON ... LIMIT 100
# 1 query total ✓

# For many-to-many:
orders = Order.objects.prefetch_related('items').all()[:100]
# 2 queries: one for orders, one for all items WHERE order_id IN (...)
```

**Solution 3: DataLoader (GraphQL)**
Batches multiple individual loads into a single query, automatically.

---

## ⏱️ Time-Series Databases

Specialized for data that is keyed by timestamp. IoT sensor readings, application metrics, financial tick data.

**InfluxDB:**
- Data model: measurement, tags (indexed metadata), fields (values), timestamp
- Automatic downsampling and retention policies
- InfluxQL / Flux query language
- Optimized for time-range queries

**TimescaleDB:**
- PostgreSQL extension — you get SQL + time-series optimizations
- Automatic time-based partitioning (hypertables)
- Continuous aggregates for pre-computed rollups
- Best of both worlds for teams familiar with SQL

**When to use:** System metrics, IoT data, financial tick data, clickstream analytics with time ranges.

---

## 🔍 Search Databases (Elasticsearch)

Elasticsearch is a distributed search and analytics engine built on Apache Lucene.

**Inverted index:** For each word, stores the list of documents containing it.
```
"iPhone" → [doc1, doc4, doc7, doc23]
"Apple"  → [doc1, doc4, doc9, doc15]
Search "Apple iPhone" → intersection → [doc1, doc4]
```

**Key features:**
- Full-text search with relevance scoring (TF-IDF / BM25)
- Faceted search and aggregations
- Near real-time indexing (1 second delay)
- Distributed: shards across nodes
- Schema-free (auto-detects types)

**When to use:**
- Full-text search (product search, document search)
- Log analysis (ELK stack)
- Auto-complete / type-ahead
- Complex aggregations on large datasets

**NOT a primary database.** Elasticsearch lacks ACID transactions. Use it as a search layer, with PostgreSQL or another DB as the source of truth. Sync data to Elasticsearch via CDC or events.

---

## 🆕 NewSQL

NewSQL databases combine the ACID guarantees and SQL interface of relational databases with the horizontal scalability of NoSQL.

**CockroachDB:**
- PostgreSQL-compatible SQL
- Distributed ACID transactions using Raft consensus
- Automatic sharding and rebalancing
- Multi-region active-active (geo-partitioned replicas)
- Serializable isolation

**Google Spanner:**
- Google's globally distributed SQL database
- Uses TrueTime API (GPS + atomic clocks) for globally synchronized timestamps
- Enables externally consistent distributed transactions
- Used for Google Ads, Finance, Gmail
- Available as Cloud Spanner on GCP

**When to use NewSQL:** You need ACID + SQL + horizontal scale. Think fintech, global apps needing consistency. Significant cost and operational complexity vs traditional RDBMS.

---

## 💳 Real-Life Scenario: Fintech Startup Database Choices

**The startup:** A fintech company building a payment processing platform.

**Requirements:**
1. Payment transactions (must be ACID, no data loss)
2. User profiles and KYC documents (flexible schema, large documents)
3. Transaction analytics and reporting (aggregate over millions of transactions)
4. Fraud detection (graph of account relationships)
5. Real-time exchange rates (time-series)

**Database choices:**

**Payments → PostgreSQL**
- ACID transactions are non-negotiable for money movement
- Complex queries for reconciliation
- Serializable isolation for preventing double-charges

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- Lock row
    UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;
    -- Check rows affected = 1, else insufficient funds
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**User profiles with KYC documents → PostgreSQL with JSONB**
- KYC documents have variable structure (different countries require different fields)
- JSONB column stores flexible data while maintaining relational structure for querying

**Analytics → BigQuery (data warehouse)**
- Petabyte scale
- Columnar storage for fast aggregations
- SQL interface
- Nightly ETL from PostgreSQL

**Fraud detection → Neo4j**
- Query: "Find accounts within 3 hops of this fraudulent account"
- Impossible with SQL JOINs efficiently; trivial graph traversal in Neo4j

**Exchange rates → TimescaleDB**
- Append-only time series
- Need to query "rate at a specific point in time" efficiently

---

## 🔥 Big Tech Story: Discord's Database Journey

### The Beginning: MongoDB

Discord launched in 2015 and chose **MongoDB** for storing messages. Reasonable choice — JSON documents map naturally to messages, flexible schema for evolving message types.

By 2017, Discord had 100 million messages per day, storing a total of 100 billion messages. MongoDB struggled:
- A single MongoDB collection had 100+ billion documents
- RAM could no longer hold the working set
- Random disk I/O for queries was causing latency spikes
- Index sizes were enormous

### The Migration: Cassandra

Discord migrated to **Apache Cassandra** in late 2017. Their data model:

```
Table: messages
Partition key: (channel_id, bucket)  ← bucket = time-based grouping
Clustering key: message_id (DESC)    ← newest first within partition

channel_id | bucket | message_id | content | author_id
-----------+--------+------------+---------+----------
123        | 2017-1 | 999999     | "Hello" | alice_id
123        | 2017-1 | 999998     | "World" | bob_id
```

This worked beautifully:
- Reads for a channel's recent messages hit one partition → fast
- Cassandra's LSM tree storage excelled at writes
- Linearly scalable by adding nodes

**But problems emerged...**

Cassandra's write path (memtable → SSTable) means deleted data isn't immediately removed — it becomes a "tombstone." Discord delete messages at scale, so they had hundreds of billions of tombstones. Compaction was a significant overhead.

Latency spikes during compaction caused p99 latency to balloon from 5ms to 250ms at random times. Very difficult to predict or control.

### The Next Migration: ScyllaDB (2022)

ScyllaDB is a Cassandra-compatible database rewritten in C++, designed to eliminate Java garbage collection pauses and be more CPU cache friendly.

Discord wrote a famous blog post "How Discord Stores Trillions of Messages" describing the migration:

Key improvements:
- **Rust for the data service layer** wrapping ScyllaDB — type safety + performance
- **Better compaction strategies** in ScyllaDB to handle tombstones more efficiently
- **Row-level repair** instead of full token range repair (much faster)
- **p99 latency dropped from 40-125ms to 15ms** for their message history service

### The Lesson

Discord's journey illustrates a crucial system design truth: **there is no permanent solution — only solutions that work at your current scale.** 

MongoDB → Cassandra → ScyllaDB wasn't a failure; it was **deliberate evolution**. Each migration was undertaken when the current solution was genuinely limiting growth, not prematurely.

The cost of each migration was enormous (months of engineering effort). The lesson: **choose your initial database carefully** based on predicted scale, but design your data access layer to abstract the database so migrations are possible when needed.

---

## ⚖️ Trade-offs & Drawbacks

### Relational Databases
| Pro | Con |
|---|---|
| ACID guarantees | Vertical scaling hits limits |
| Powerful SQL | Schema migrations are painful at scale |
| Mature tooling | JOINs across shards are impossible |
| Strong consistency | Not designed for unstructured data |

### NoSQL Databases
| Pro | Con |
|---|---|
| Horizontal scale | Eventual consistency is hard to reason about |
| Flexible schema | Limited query capabilities |
| High write throughput | No JOINs → data denormalization required |
| Purpose-optimized | Less mature tooling |

### Sharding
| Pro | Con |
|---|---|
| Unlimited horizontal scale | Cross-shard transactions are complex |
| Parallel query execution | Cross-shard JOINs require app logic |
| | Resharding is painful |
| | Increased operational complexity |

---

## ✅ Best Practices

1. **Start with PostgreSQL.** It handles enormous scale with proper indexing, read replicas, and partitioning. Many companies run billions of rows in PostgreSQL.

2. **Use `EXPLAIN ANALYZE`** to understand your query plans. Never assume your query is fast without verification.

3. **Design indexes for your query patterns, not your table structure.** Create indexes based on `WHERE`, `JOIN`, `ORDER BY` clauses in your actual queries.

4. **Separate read and write workloads.** Use read replicas for analytics, reporting, and read-heavy API endpoints. Reserve primary for writes.

5. **Use connection pooling in production.** PgBouncer for PostgreSQL is non-negotiable at scale.

6. **Choose your shard key with extreme care.** Changing a shard key later requires full data migration. Model it correctly from the start.

7. **Avoid cross-shard operations.** Design your data model so that related data lives on the same shard. Denormalize aggressively for sharded systems.

8. **Use soft deletes instead of hard deletes when possible.** Add `deleted_at TIMESTAMP NULL` column. Hard deletes can cause Cassandra tombstone problems and complicate audit trails.

9. **Set appropriate connection limits and timeouts.** Unbounded connections can exhaust database resources. Always set `statement_timeout`, `lock_timeout`, and `idle_in_transaction_session_timeout` in PostgreSQL.

10. **Backup strategy: 3-2-1 rule.** 3 copies of data, 2 different storage media, 1 offsite location. Test your backups regularly by restoring to a test environment.

11. **Use database migrations tools.** Flyway, Liquibase, or Alembic for version-controlled schema changes. Never run raw SQL directly against production.

12. **Monitor slow queries.** Enable `log_min_duration_statement` in PostgreSQL to log queries exceeding a threshold. Long-running queries can hold locks and cause cascading failures.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **ACID** | Atomicity + Consistency + Isolation + Durability = correctness guarantees for transactions |
| **Isolation Levels** | Higher isolation = fewer anomalies = lower concurrency; Read Committed is the pragmatic default |
| **B+ Tree** | All data in leaves, linked list for range scans; O(log n) operations |
| **CAP Theorem** | P is non-optional; real choice is C vs A during network partitions |
| **SQL vs NoSQL** | Start with SQL; go NoSQL only for specific scale/flexibility needs |
| **Sharding** | Distribute data horizontally; consistent hashing prevents mass rehashing |
| **Replication** | Master for writes + replicas for reads; async replication means eventual consistency |
| **Connection Pooling** | Essential at scale; 1000 app threads → 50 DB connections |
| **N+1 Problem** | Always use JOINs or eager loading; never query in a loop |
| **Discord's Lesson** | Database choices evolve with scale; abstract your data layer for future migrations |

---

*Next: [05_Message_Queues_and_Event_Driven_Architecture.md](05_Message_Queues_and_Event_Driven_Architecture.md) — Decoupling services with Kafka, RabbitMQ, and event-driven patterns.*
