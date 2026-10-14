# Redis: The In-Memory Speed Demon & Multi-Model Engine

## Overview
Redis (Remote Dictionary Server), created by Salvatore Sanfilippo (antirez), is an open-source, in-memory networked data store used as a primary database, cache, streaming engine, and message broker. Renowned for its sub-millisecond latencies, Redis executes hundreds of thousands of operations per second per CPU core by maintaining all datasets entirely in RAM while offering optional asynchronous and synchronous disk persistence.

---

## Architectural Deep Dive: Single-Threaded Event Loop & In-Memory Internals

```
+-------------------------------------------------------------------------+
|                        Redis Engine Architecture                        |
+-------------------------------------------------------------------------+
 Non-blocking I/O Multiplexing (epoll / kqueue / select)
                        │
                        ▼
 Single-Threaded Event Loop (ae.c File & Time Events)
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
 Command Execution   RAM Dicts     Background Threads (Redis 6.0+)
 (Atomic in-memory)  (ziplist/     - I/O socket read/write
                      skiplist)    - Asynchronous UNLINK / AOF fsync
                        │
       ┌────────────────┴────────────────┐
       ▼                                 ▼
 RDB Snapshots (fork COW)         AOF Log (Append-Only File)
```

### 1. Why Single-Threaded Command Execution Wins
- Eliminates CPU thread context switching and lock contention (no mutexes, deadlocks, or semaphore bottlenecks).
- Memory-bandwidth bound rather than CPU-bound: modern CPUs can execute sequential in-memory dictionary lookups far faster than network packets arrive.
- *Redis 6.0+ Enhancement:* Multi-threaded I/O handles network serialization/deserialization while keeping core state execution strictly single-threaded.

### 2. Memory Persistence Options
- **RDB (Redis Database Snapshot):** Point-in-time binary snapshot of memory dumped to disk via `BGSAVE`. The OS utilizes `fork()` with **Copy-On-Write (COW)**. Very fast restarts; risk of losing last $N$ minutes of data if crashed before snapshot.
- **AOF (Append-Only File):** Logs every mutating command sequentially.
  - `appendfsync always`: fsync on every write (slowest, zero data loss).
  - `appendfsync everysec`: fsync once per second (industry standard balance).
  - `appendfsync no`: OS decides when to flush (fastest, risky).
- **AOF Rewrite (`BGREWRITEAOF`):** Asynchronously consolidates redundant commands into the minimal set needed to reconstruct the dataset.

### 3. Memory Eviction Policies (`maxmemory-policy`)
When memory reaches `maxmemory`, Redis applies the configured policy:
- `noeviction`: Returns errors on write operations when memory is full (Default).
- `allkeys-lru` / `volatile-lru`: Evicts the Least Recently Used keys across all keys or keys with TTL.
- `allkeys-lfu` / `volatile-lfu`: Evicts the Least Frequently Used keys.
- `volatile-ttl`: Evicts keys with the shortest remaining time to live.

---

## Real World Stories & Incidents

### 1. Twitter's Home Timeline Architecture
Twitter generated users' home timelines via Redis Lists:
- When a user tweets, a background worker performs a **Fanout-On-Write**: pushing the `tweet_id` into the Redis List (`LPUSH`) of all their followers.
- Reading a timeline is a single `LRANGE timeline:user_id 0 20` lookup requiring $< 2\text{ms}$.
- To prevent unbounded memory consumption, Twitter bounds each user list with `LTRIM timeline:user_id 0 799`, retaining only the 800 most recent tweets.

### 2. The Blocking `KEYS *` Outage Nightmare
A Fortune 500 retailer crashed their e-commerce checkout system on Black Friday:
- A newly hired developer deployed an administrative cron job that executed `KEYS user:*` to count active sessions across 50,000,000 keys.
- Because Redis command execution is single-threaded, the `KEYS` command blocked the entire server for 42 seconds while traversing the dictionary.
- All web frontend health checks timed out, connection pools exhausted, and the application cascaded into total downtime.
- *Remedy:* Always use cursor-based **`SCAN`** in production; disable dangerous commands via `rename-command KEYS ""` in `redis.conf`.

### 3. GitHub's Cache Stampede & Redis Failover
GitHub experienced an incident where a Redis primary failed, and automated failover promoted a secondary replica that had slight replication lag.
- Cache misses spiked simultaneously across thousands of application threads (**Thundering Herd / Cache Stampede**), hitting the underlying MySQL cluster with millions of concurrent read queries.
- MySQL connections maxed out immediately. GitHub mitigated this by adding distributed mutexes and probabilistic early expiration (XFetch algorithm).

---

## Comprehensive Redis Commands Reference by Data Type

### 1. Strings (Text, Binary Blobs, Integers, Floats)

```text
# SET with expiration and conditional flags
# NX = Set only if key DOES NOT exist
# XX = Set only if key ALREADY exists
# EX = Expire in seconds, PX = milliseconds
SET session:1001 "user_payload" EX 3600 NX

# Retrieve values
GET session:1001

# Multi-Key Atomic Batch (Reduces network round-trips)
MSET config:theme "dark" config:currency "USD" config:locale "en-US"
MGET config:theme config:currency

# Atomic Numerical Increments (Counters, Rate Limiting)
INCR api:hits:user_42
INCRBY api:hits:user_42 10
INCRBYFLOAT wallet:balance:user_42 12.50
DECR inventory:stock:item_99
DECRBY inventory:stock:item_99 5

# String Manipulation
APPEND log:stream "[ERROR] Connection refused\n"
STRLEN session:1001
GETSET lock:resource "new_owner"   # Returns old value and sets new atomically
GETDEL token:temp_auth             # Atomically reads and deletes
```

---

### 2. Lists (Linked Lists / Quicklists — Queues & Stacks)

```text
# Producer: Push elements to head or tail
LPUSH task:queue "job_id_1" "job_id_2"
RPUSH task:queue "job_id_3"

# Consumer: Pop elements (LIFO Stack or FIFO Queue)
RPOP task:queue                     # Returns "job_id_3" (FIFO when paired with LPUSH)
LPOP task:queue                     # Returns "job_id_2"

# Blocking Pops for Real-Time Workers (0 = wait indefinitely until item arrives)
BRPOP task:queue 5                  # Waits up to 5 seconds for item to be pushed

# Inspection and Pagination
LLEN task:queue
LRANGE task:queue 0 9               # Return first 10 items without popping
LINDEX task:queue 2                 # Get item at index 2 (O(N) operation)

# Queue Trimming (Bounding memory)
LTRIM timeline:user_12 0 99         # Retains only the 100 most recent items

# Safe Queue Processing: Atomic Move
LMOVE source:queue processing:queue LEFT RIGHT
```

---

### 3. Sets (Unordered Unique Collections)

```text
# Add and Remove members
SADD user:101:tags "java" "redis" "docker" "microservices"
SREM user:101:tags "docker"

# Membership Testing (O(1))
SISMEMBER user:101:tags "redis"     # Returns 1 (true)
SMISMEMBER user:101:tags "java" "go" # Returns [1, 0]

# Set Cardinality and Retrieval
SCARD user:101:tags                 # Returns 3
SMEMBERS user:101:tags              # Warning: Use SSCAN if set has millions of elements

# Set Algebra (Intersections, Unions, Differences)
SADD user:102:tags "java" "python" "kubernetes"

# Common tags between users (Intersection)
SINTER user:101:tags user:102:tags  # Returns ["java"]

# Union across multiple users
SUNION user:101:tags user:102:tags

# Difference (Tags user 101 has that 102 doesn't)
SDIFF user:101:tags user:102:tags   # Returns ["redis", "microservices"]

# Store result of set operation into new key
SINTERSTORE common:tags:101:102 user:101:tags user:102:tags
```

---

### 4. Sorted Sets (ZSET — Leaderboards & Priority Queues)

Backed by a dual data structure: a Hash Table (for $O(1)$ lookups) and a Skip List (for $O(\log N)$ score-sorted range queries).

```text
# Add elements with floating-point scores
# Flags: NX (only new), XX (only update), GT (update only if new score > current)
ZADD leaderboard:weekly 1500 "player_alice"
ZADD leaderboard:weekly 2400 "player_bob" 1850 "player_charlie"

# Increment Score atomically
ZINCRBY leaderboard:weekly 300 "player_alice" # Alice's score becomes 1800

# Rankings (0-indexed)
ZRANK leaderboard:weekly "player_alice"      # Ascending rank
ZREVRANK leaderboard:weekly "player_bob"     # Descending rank (Rank 0 = 1st place)
ZSCORE leaderboard:weekly "player_bob"       # Returns "2400"

# Fetch Top 10 Players (Descending order with scores)
ZREVRANGE leaderboard:weekly 0 9 WITHSCORES

# Fetch Range by Score (e.g., players between 1500 and 2000 points)
ZRANGEBYSCORE leaderboard:weekly 1500 2000 WITHSCORES
ZCOUNT leaderboard:weekly 1500 2000          # Count matching elements

# Priority Queue Operations: Pop highest / lowest score
ZPOPMIN leaderboard:weekly 1                 # Pop lowest score
ZPOPMAX leaderboard:weekly 1                 # Pop highest score
BZPOPMAX leaderboard:weekly 5                # Blocking pop
```

---

### 5. Hashes (Field-Value Maps / Objects)

Ideal for representing entity objects (e.g., user profiles) without JSON serialization overhead:

```text
# Set multiple fields
HSET user:profile:1001 name "Sarah Connor" email "sarah@sky.net" age 28 role "admin"

# Retrieve fields
HGET user:profile:1001 email
HMGET user:profile:1001 name role nonexistent_field
HGETALL user:profile:1001                    # Retrieves all key-value pairs

# Field Existence and Deletion
HEXISTS user:profile:1001 age
HDEL user:profile:1001 role

# Atomic Numerical Increment on Field
HINCRBY user:profile:1001 login_count 1
HINCRBYFLOAT user:profile:1001 account_balance 50.25

# Inspection
HLEN user:profile:1001                       # Number of fields
HKEYS user:profile:1001                      # Array of field names
HVALS user:profile:1001                      # Array of values
```

---

### 6. Bitmaps (Bitwise Analytics & DAU Tracking)

Bitmaps allow tracking boolean state for hundreds of millions of users in megabytes of RAM:

```text
# Track Daily Active Users (User ID maps directly to bit offset)
# User 42 and User 85 log in on Day 2024-06-01
SETBIT dau:2024-06-01 42 1
SETBIT dau:2024-06-01 85 1

# Check if User 42 was active
GETBIT dau:2024-06-01 42                     # Returns 1
GETBIT dau:2024-06-01 99                     # Returns 0

# Count Total Active Users on that day
BITCOUNT dau:2024-06-01                      # Returns 2

# Bitwise Operations across multiple days (e.g., 2-day retention)
SETBIT dau:2024-06-02 42 1                   # User 42 logged in again next day

# AND operation: Find users who logged in on BOTH days
BITOP AND dau:active_both_days dau:2024-06-01 dau:2024-06-02
BITCOUNT dau:active_both_days                # Returns 1 (User 42)
```

---

### 7. HyperLogLog (Cardinality Estimation at Scale)

Fixed 12 KB memory footprint for estimating billions of unique events with $0.81\%$ standard error:

```text
# Add elements
PFADD visitor:unique:video_99 "192.168.1.1" "10.0.0.2" "172.16.0.4"
PFADD visitor:unique:video_99 "192.168.1.1" # Duplicate; returns 0

# Approximate Cardinality Count
PFCOUNT visitor:unique:video_99              # Returns 3

# Merge multiple HyperLogLogs into one (e.g., Weekly Unique Visitors)
PFMERGE visitor:weekly visitor:mon visitor:tue visitor:wed visitor:thu visitor:fri
PFCOUNT visitor:weekly
```

---

### 8. Streams (Append-Only Event Log & Consumer Groups)

Redis Streams model Kafka-like durable event logs with consumer groups:

```text
# Producer: Append event (* generates auto-timestamp ID)
XADD order_events * order_id 9901 user_id 42 amount 150.00 status "PLACED"

# Create Consumer Group (Starts listening from the beginning '$' or from start '0')
XGROUP CREATE order_events order_processors 0 MKSTREAM

# Consumer: Read new messages in consumer group
# > indicates messages never delivered to any other consumer in this group
XREADGROUP GROUP order_processors worker_1 COUNT 2 BLOCK 2000 STREAMS order_events >

# Acknowledge processed message
XACK order_events order_processors "1717240000000-0"

# Pending Message Inspection (Fault Tolerance for crashed consumers)
XPENDING order_events order_processors
XCLAIM order_events order_processors worker_2 60000 "1717240000000-0" # Reassign stuck message
```

---

### 9. Geospatial Indexes

```text
# Add Coordinates (Longitude, Latitude, Name)
GEOADD drivers:locations -122.4194 37.7749 "driver_bob" -122.4089 37.7833 "driver_alice"

# Distance between two members
GEODIST drivers:locations "driver_bob" "driver_alice" km

# Search within Radius (GEOSEARCH)
GEOSEARCH drivers:locations 
    FROMLONLAT -122.4194 37.7749 
    BYRADIUS 5 km 
    ASC 
    WITHCOORD WITHDIST WITHHASH
```

---

### 10. Pub/Sub Messaging

```text
# Subscriber Session:
SUBSCRIBE notifications:alerts
PSUBSCRIBE logs:*                           # Pattern subscription

# Publisher Session:
PUBLISH notifications:alerts "Service payment-gateway is degraded"
```

---

### 11. Transactions (MULTI/EXEC) & Lua Scripting

#### Optimistic Locking with WATCH:
```text
WATCH inventory:item:42
GET inventory:item:42
MULTI
DECR inventory:item:42
EXEC                                        # Fails (returns nil) if key changed by another client
```

#### Atomic Distributed Lock via Lua Script:
```lua
-- Acquire Lock: SET lock:resource "uuid_token" NX PX 30000
-- Release Lock Safely (Only delete if token matches to prevent releasing someone else's lock):
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```
Invoked via:
```bash
redis-cli --eval release_lock.lua lock:resource , uuid_token
```

---

### 12. Keyspace Management & Administration

```text
# Non-blocking deletion (Asynchronously reclaims memory in background thread)
UNLINK massive_key

# Key TTL management
EXPIRE session:1001 1800
TTL session:1001                             # Remaining seconds (-1 = no TTL, -2 = key not found)
PERSIST session:1001                         # Removes expiration, makes persistent

# Safe Production Key Scanning (Cursor-based)
SCAN 0 MATCH user:* COUNT 100

# Performance & Memory Profiling
INFO memory
INFO stats
SLOWLOG GET 10                               # Top 10 slowest queries
MONITOR                                      # Real-time stream of all processed commands (Dev only!)
MEMORY USAGE user:profile:1001
```
