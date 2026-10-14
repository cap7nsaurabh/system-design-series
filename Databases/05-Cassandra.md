# Apache Cassandra: The Masterless Wide-Column Behemoth

## Overview
Apache Cassandra is a distributed, decentralized, highly available NoSQL database designed to handle petabyte-scale write-heavy workloads across commodity hardware and multiple cloud datacenters with zero single points of failure. Originally authored at Facebook by Avinash Lakshman (co-creator of Amazon Dynamo) and Prashant Malik, Cassandra marries the **distributed masterless architecture of Amazon Dynamo** with the **log-structured wide-column data model of Google Bigtable**.

---

## Architectural Deep Dive: The Peer-to-Peer Ring & LSM Storage Engine

```
+-------------------------------------------------------------------------+
|                  Cassandra Consistent Hashing Ring                      |
+-------------------------------------------------------------------------+
                    Node A (Tokens: 0 - 2^63/4)
                    /                          \
          Node D                                 Node B
 (Tokens: 3*(2^63/4) - 2^64-1)            (Tokens: 2^63/4 - 2^63/2)
                    \                          /
                    Node C (Tokens: 2^63/2 - 3*(2^63/4))
```

### 1. Consistent Hashing & Token Ring
- Data is distributed across cluster nodes using consistent hashing. The **Partition Key** is hashed via the `Murmur3Partitioner` into a 64-bit integer between $-2^{63}$ and $2^{63}-1$.
- Every physical node owns multiple **Virtual Nodes (vnodes)** distributed along this token ring.
- Replicas are placed by moving clockwise around the ring according to the keyspace's replication factor ($RF$).

### 2. Log-Structured Merge-Tree (LSM) Write Path (Blazing Fast Writes)
1. **CommitLog:** When a write arrives, it is immediately appended to an append-only sequential disk log (`CommitLog`) to ensure durability. Disk seeks are avoided.
2. **Memtable:** Simultaneously, the write is written to an in-memory sorted data structure (`Memtable`).
3. Once the Memtable fills up, it flushes to disk as an immutable **SSTable (Sorted String Table)**.
4. **Bloom Filter:** Each SSTable has an in-memory Bloom Filter to immediately rule out whether a partition key exists in that file without hitting disk.
5. **Compaction:** Background worker threads periodically merge multiple SSTables into a single larger SSTable, resolving duplicate records and discarding deleted items.

### 3. Tombstones: The Silent Performance Killer
- In Cassandra, a `DELETE` does **not** erase data immediately. It writes an append-only marker called a **Tombstone** with a deletion timestamp.
- Tombstones remain on disk until `gc_grace_seconds` (default 10 days) expires and a major compaction occurs.
- If an application creates millions of tombstones (e.g., polling queues or deleting rows frequently), queries scanning through tombstones will suffer severe latency spikes or trigger `TombstoneOverwhelmingException`.

---

## Real World Stories & Incidents

### 1. Discord's Journey from Cassandra to ScyllaDB
Discord stored trillions of chat messages across 177 Cassandra nodes:
- As servers with hundreds of thousands of members actively chatted and deleted messages, Cassandra's Java Virtual Machine (JVM) suffered from severe **Garbage Collection (GC) pauses**.
- Tombstone buildup from deleted channels and messages caused read latency spikes up to several seconds during p99 intervals.
- In 2022, Discord migrated to **ScyllaDB** (a C++ rewrite of Cassandra that uses an asynchronous seastar thread-per-core architecture), reducing tail latencies from seconds to single-digit milliseconds while shrinking the cluster from 177 nodes to 72 nodes.

### 2. Netflix: Zero-Downtime Multi-Region Resilience
In 2014, an entire AWS datacenter in `us-east-1` crashed during a major power outage:
- Netflix configured their Cassandra keyspaces with `NetworkTopologyStrategy` across three AWS regions: `us-east-1`, `us-west-2`, and `eu-west-1`.
- Because clients used `LOCAL_QUORUM` for reads and writes, traffic from the failed datacenter was automatically rerouted to `us-west-2`.
- Cassandra clusters continued serving streaming metadata without data corruption and with zero customer disruption.

### 3. Apple's Massive 160,000+ Node Fleet
Apple manages one of the largest Cassandra deployments in the world, running over 160,000 Cassandra instances to support Apple Music, iCloud, and Siri. They rely on Cassandra's linear scalability: doubling hardware strictly doubles write capacity.

---

## Comprehensive Cassandra CQL (Cassandra Query Language) Reference

### 1. Keyspace Operations & Replication Strategies

```sql
-- SimpleStrategy: Single Datacenter / Local Development only
CREATE KEYSPACE dev_keyspace 
WITH replication = {
    'class': 'SimpleStrategy', 
    'replication_factor': 1
};

-- NetworkTopologyStrategy: Multi-Datacenter Production Configuration
CREATE KEYSPACE telemetry_keyspace 
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'us-east-1': 3,
    'eu-central-1': 3
} 
AND durable_writes = true;

-- Alter and Drop Keyspace
ALTER KEYSPACE telemetry_keyspace WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east-1': 5, 'eu-central-1': 3};
DROP KEYSPACE IF EXISTS dev_keyspace;

USE telemetry_keyspace;
```

---

### 2. Table Creation, Keys, & Storage Options

```sql
-- User-Defined Type (UDT)
CREATE TYPE geo_point (
    latitude DOUBLE,
    longitude DOUBLE,
    elevation FLOAT
);

-- Table with Composite Partition Key, Clustering Keys, Collections, and Options
CREATE TABLE device_telemetry (
    tenant_id UUID,
    device_type VARCHAR,
    device_id UUID,
    recorded_at TIMESTAMP,
    temperature DOUBLE,
    battery_level FLOAT,
    location FROZEN<geo_point>,
    tags SET<TEXT>,
    error_logs LIST<TEXT>,
    sensor_readings MAP<TEXT, DOUBLE>,
    -- Composite Partition Key ((tenant_id, device_type)) + Clustering Keys (device_id, recorded_at)
    PRIMARY KEY ((tenant_id, device_type), device_id, recorded_at)
) 
WITH CLUSTERING ORDER BY (device_id ASC, recorded_at DESC)
AND compaction = {
    'class': 'TimeWindowCompactionStrategy', -- Ideal for time-series data
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': '1'
}
AND default_time_to_live = 2592000 -- 30 days automatic expiration
AND gc_grace_seconds = 86400;      -- 1 day before tombstones are cleared
```

---

### 3. Complete CRUD Operations

```sql
-- INSERT with TTL (Time To Live in seconds) and custom writetime
INSERT INTO device_telemetry (
    tenant_id, device_type, device_id, recorded_at, temperature, battery_level, location, tags
) VALUES (
    a1b2c3d4-e5f6-7890-abcd-ef1234567890,
    'THERMOSTAT',
    f47ac10b-58cc-4372-a567-0e02b2c3d479,
    '2024-06-01 12:00:00+0000',
    22.5,
    98.0,
    {latitude: 37.7749, longitude: -122.4194, elevation: 15.2},
    {'indoor', 'living_room'}
) 
USING TTL 86400; -- Expires in 24 hours

-- SELECT Queries (Must follow Partition Key -> Clustering Key rule)
SELECT device_id, recorded_at, temperature, TTL(temperature), WRITETIME(temperature)
FROM device_telemetry
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890
  AND device_type = 'THERMOSTAT'
  AND device_id = f47ac10b-58cc-4372-a567-0e02b2c3d479
  AND recorded_at >= '2024-06-01 00:00:00+0000'
  AND recorded_at <= '2024-06-01 23:59:59+0000'
LIMIT 50;

-- Per-Partition Limit (Return latest 3 records per partition)
SELECT * FROM device_telemetry
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890 AND device_type = 'THERMOSTAT'
PER PARTITION LIMIT 3;

-- Token Range Scan (Full table scanning without coordinator bottleneck)
SELECT * FROM device_telemetry
WHERE token(tenant_id, device_type) >= token(a1b2c3d4-e5f6-7890-abcd-ef1234567890, 'THERMOSTAT')
LIMIT 100;

-- DELETE Operations (Warning: Generates tombstones!)
DELETE temperature FROM device_telemetry
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890
  AND device_type = 'THERMOSTAT'
  AND device_id = f47ac10b-58cc-4372-a567-0e02b2c3d479
  AND recorded_at = '2024-06-01 12:00:00+0000';
```

---

### 4. Collection Manipulations (List, Set, Map)

```sql
-- SET: Add and remove elements without reading whole collection
UPDATE device_telemetry 
SET tags = tags + {'calibrated', 'v2_firmware'}
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890 AND device_type = 'THERMOSTAT' AND device_id = f47ac10b-58cc-4372-a567-0e02b2c3d479 AND recorded_at = '2024-06-01 12:00:00+0000';

UPDATE device_telemetry 
SET tags = tags - {'living_room'}
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890 AND device_type = 'THERMOSTAT' AND device_id = f47ac10b-58cc-4372-a567-0e02b2c3d479 AND recorded_at = '2024-06-01 12:00:00+0000';

-- LIST: Prepend and Append operations
UPDATE device_telemetry 
SET error_logs = error_logs + ['Sensor timeout: 504']
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890 AND device_type = 'THERMOSTAT' AND device_id = f47ac10b-58cc-4372-a567-0e02b2c3d479 AND recorded_at = '2024-06-01 12:00:00+0000';

-- MAP: Set key and delete key
UPDATE device_telemetry 
SET sensor_readings['humidity'] = 45.2
WHERE tenant_id = a1b2c3d4-e5f6-7890-abcd-ef1234567890 AND device_type = 'THERMOSTAT' AND device_id = f47ac10b-58cc-4372-a567-0e02b2c3d479 AND recorded_at = '2024-06-01 12:00:00+0000';
```

---

### 5. Distributed Counter Tables

Cassandra counters are specialized 64-bit integers with commutative replication:

```sql
CREATE TABLE video_view_counts (
    video_id UUID,
    channel_id UUID,
    view_count COUNTER,
    PRIMARY KEY (video_id, channel_id)
);

-- Counters can only be incremented or decremented
UPDATE video_view_counts 
SET view_count = view_count + 1 
WHERE video_id = e5d6c7b8-a901-2345-6789-abcdef012345 
  AND channel_id = 12345678-1234-1234-1234-123456789012;
```

---

### 6. Lightweight Transactions (LWT / Paxos Protocol)

Provides linearizable compare-and-set (CAS) consistency at the cost of 4 round trips:

```sql
-- Conditional Insert (Insert only if key doesn't exist)
INSERT INTO user_accounts (username, email, password_hash)
VALUES ('john_doe', 'john@example.com', 'hashed_pass')
IF NOT EXISTS;

-- Conditional Update
UPDATE bank_balances 
SET balance = balance - 100.00
WHERE account_id = 90210
IF balance >= 100.00;
```

---

### 7. Storage-Attached Indexing (SAI / Cassandra 4.0+)

Traditional secondary indexes in Cassandra scatter queries to every node. Modern **SAI** indexes are attached directly to SSTables:

```sql
-- Create SAI Index on non-partition column
CREATE CUSTOM INDEX idx_device_temperature ON device_telemetry (temperature)
USING 'StorageAttachedIndex';

-- Fast indexed lookup without ALLOW FILTERING
SELECT * FROM device_telemetry WHERE temperature > 35.0;
```

---

### 8. Batch Statements (Logged vs Unlogged)

```sql
-- LOGGED BATCH: Atomically applies all writes to multiple tables (Ensures eventual consistency)
BEGIN BATCH
    INSERT INTO users (user_id, username) VALUES (1, 'alice');
    INSERT INTO users_by_email (email, user_id) VALUES ('alice@example.com', 1);
APPLY BATCH;

-- UNLOGGED BATCH: Performance optimization for writes to the SAME partition (No atomic guarantee)
BEGIN UNLOGGED BATCH
    INSERT INTO user_events (user_id, event_time, action) VALUES (1, '2024-01-01 10:00:00', 'LOGIN');
    INSERT INTO user_events (user_id, event_time, action) VALUES (1, '2024-01-01 10:01:00', 'CLICK');
APPLY BATCH;
```

---

### 9. Tunable Consistency Levels

| Consistency Level | Reads Required | Writes Required | Tradeoff |
|---|---|---|---|
| `ANY` | N/A | 1 (even hinted handoff) | Highest write availability, data loss risk |
| `ONE` | 1 replica | 1 replica | Lowest latency, potential stale reads |
| `QUORUM` | $\lfloor RF/2 \rfloor + 1$ | $\lfloor RF/2 \rfloor + 1$ | Strong consistency across entire cluster |
| `LOCAL_QUORUM` | $\lfloor RF_{DC}/2 \rfloor + 1$ | $\lfloor RF_{DC}/2 \rfloor + 1$ | Strong consistency within local DC (Industry Standard) |
| `ALL` | All $RF$ replicas | All $RF$ replicas | Strongest, fails if single node is down |

---

### 10. Node Administration & Nodetool Commands

```bash
# Cluster health and ring topology
nodetool status
nodetool ring

# Flush Memtables to SSTables manually
nodetool flush telemetry_keyspace device_telemetry

# Trigger Anti-Entropy Repair (Synchronizes desynchronized replicas)
nodetool repair -pr telemetry_keyspace

# Check compaction status and active thread pools
nodetool compactionstats
nodetool tpstats

# Identify hot partitions and high tombstone reads
nodetool toppartitions telemetry_keyspace device_telemetry 60000
nodetool tablestats telemetry_keyspace.device_telemetry
```
