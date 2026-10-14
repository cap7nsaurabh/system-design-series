# Databases: Deep Dive into SQL, NoSQL, Search & Time-Series Engines

Welcome to the Databases series. This section covers leading database technologies, their core architectural internals, storage engine mechanics, real-world failure post-mortems from top tech companies, and exhaustive query/method syntax references.

---

## 📚 Table of Contents

### 1. Relational (SQL) Databases
Relational engines built on strict ACID compliance, structured schemas, declarative SQL, and relational integrity.

* [01 MySQL: The Web's Relational Workhorse](./01-MySQL.md)
  * *Focus:* InnoDB buffer pool, WAL Redo/Undo logs, Clustered B+ Trees, Uber Schemaless migration, GitHub split-brain postmortem, Vitess sharding, Complete SQL & Window function reference.
* [02 PostgreSQL: The Object-Relational Titan](./02-PostgreSQL.md)
  * *Focus:* MVCC append-only tuples, VACUUM engine & bloat, HOT optimization, Instagram sharding, JSONB operator reference, Full-Text Search (tsvector/tsquery), Range types, RLS, Advisory locks, SKIP LOCKED work queues.
* [03 Oracle Database: The Enterprise Transaction Giant](./03-Oracle.md)
  * *Focus:* SGA/PGA memory architecture, SCN & Undo consistency, Oracle RAC Cache Fusion, Amazon migration off Oracle, MERGE (UPSERT), CONNECT BY hierarchical queries, Flashback queries, PL/SQL packages & triggers, Autonomous transactions.

### 2. Document & Key-Value NoSQL Databases
Non-relational databases engineered for flexible schemas, high write concurrency, and horizontal scaling.

* [04 MongoDB: The Document Database Giant](./04-MongoDB.md)
  * *Focus:* WiredTiger storage engine, document-level locking, BSON, eBay migration, complete CRUD & array modifiers, all 20+ aggregation pipeline stages, multi-document ACID transactions, Change Streams, GridFS.
* [06 Redis: The In-Memory Speed Demon & Multi-Model Store](./06-Redis.md)
  * *Focus:* Single-threaded event loop, RDB/AOF persistence, memory eviction algorithms, Twitter timeline architecture, blocking KEYS outage postmortem, exhaustive reference across Strings, Lists, Sets, ZSETs, Hashes, Bitmaps, HyperLogLog, Streams, Geo, and Lua distributed locks.
* [07 Amazon DynamoDB: Serverless Cloud-Native Key-Value & Document Engine](./07-DynamoDB.md)
  * *Focus:* Multi-Paxos storage partitions, partition capacity limits (10GB/1000WCU/3000RCU), Amazon Prime Day scale, AWS 2015 outage postmortem, RCU/WCU calculation math, Python boto3 SDK, Query vs Scan, TransactWriteItems, and PartiQL queries.

### 3. Distributed Wide-Column & Graph Engines
Specialized distributed architectures for extreme write throughput and complex relationship traversals.

* [05 Apache Cassandra: The Masterless Wide-Column Behemoth](./05-Cassandra.md)
  * *Focus:* Consistent hashing token ring, LSM-Tree write path (CommitLog/Memtable/SSTable), Tombstones, Discord ScyllaDB migration, Netflix multi-region survival, complete CQL reference, collections, distributed counters, LWT Paxos, SAI indexes, tunable consistency levels, and nodetool.
* [08 Neo4j: The Native Graph Database Powerhouse](./08-Neo4j.md)
  * *Focus:* Index-Free Adjacency ($O(1)$ physical pointer traversals), Causal Clustering Raft architecture, Panama Papers investigation, real-time financial fraud ring queries, complete Cypher reference (MATCH, OPTIONAL MATCH, variable-length paths, MERGE, WITH, DETACH DELETE, APOC, GDS PageRank/Louvain).

### 4. Search & Time-Series Engines
Specialized engines for full-text search, logging analytics, IoT telemetry, and financial tick data.

* [09 Elasticsearch: Distributed Search & Analytics Engine](./09-Elasticsearch.md)
  * *Focus:* Apache Lucene inverted indexes, FST dictionaries, immutable segments, Doc Values vs Fielddata, LinkedIn search, Pre-7.x Split-Brain postmortem, complete Query DSL (match, bool, nested, geo, aggregations, ILM).
* [10 TimescaleDB & InfluxDB: Time-Series Databases](./10-TimescaleDB-and-InfluxDB.md)
  * *Focus:* Hypertables & Chunk Exclusion, TSM engines, Cardinality explosion limits, Tesla fleet telemetry, Robinhood market tick charts, full TimescaleDB SQL (time_bucket, gap filling, continuous aggregates, columnar compression, data retention), InfluxQL/Flux syntax.

---

## 🎯 Database Selection & Decision Matrix

| Workload Requirement | Recommended Database | Primary Reason | Secondary Alternative |
|---|---|---|---|
| **E-Commerce Transactions / Financial Ledger** | PostgreSQL / MySQL / Oracle | Strict ACID, relational foreign keys, row locks | Amazon Aurora |
| **User Sessions / Rate Limiting / Caching** | Redis | In-memory sub-millisecond latencies, atomic increments | Memcached |
| **High Write IoT / Telemetry Logging** | Cassandra / TimescaleDB | LSM append-only write path, automated time partitioning | InfluxDB |
| **Catalog / Flexible Schema / E-Commerce Products** | MongoDB / DynamoDB | Nested document model, schema agility | PostgreSQL (JSONB) |
| **Full-Text Search / Log Aggregation** | Elasticsearch | Inverted indexes, BM25 scoring, fuzzy matching | OpenSearch |
| **Social Connections / Fraud Rings / Knowledge Graph** | Neo4j | Index-Free Adjacency ($O(1)$ multi-hop traversals) | Amazon Neptune |
| **Serverless Web Application on AWS** | DynamoDB | Zero server management, auto-scaling, pay-per-request | AWS Aurora Serverless |
