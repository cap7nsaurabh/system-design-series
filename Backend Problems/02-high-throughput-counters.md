# High-Throughput Counters & Volatile Entities: Scaling Views, Likes, and Unique Reach

## 1. Executive Summary & Real-World Production Challenges

Resource counters and cardinality metrics under viral traffic conditions represent one of the most deceptively complex challenges in backend and data systems engineering. When a cultural phenomenon occurs—such as a MrBeast video launch, a Grand Theft Auto trailer drop, a World Cup final goal, or a presidential debate—backend systems encounter two divergent scaling problems:

```
                            THE VOLATILE METRICS DUALITY
                            
         TOTAL EVENT VELOCITY                          CARDINALITY ESTIMATION
   (Non-Idempotent / Cumulative)                   (Idempotent / Deduplicated)
  
   - YouTube Raw Views (150,000/sec)               - Instagram "Accounts Reached" (100M unique)
   - TikTok Video Plays & Replays                  - Daily Active Users (DAU / MAU)
   - Twitch Live Stream Concurrent Viewers         - Deduplicated Audience Reach across campaigns
   - Reddit Upvotes & X/Twitter Likes              - Unique IP addresses in DDoS mitigation
                   │                                               │
                   ▼                                               ▼
   Challenge: Lock Contention & Hotspots           Challenge: Memory Explosion & Set Unions
   Solution: Sharding, Buffering, Streaming        Solution: HyperLogLog, Count-Min Sketch
```

---

### Real-World Production Scenarios & Outages

#### 1. The Famous YouTube "301+ Views Freeze"
- **The Phenomenon:** In YouTube's legacy architecture, video view counts would reliably freeze at `301` for hours or days when a video went viral.
- **The Underlying Reality:** This was not a glitch; it was an intentional architectural boundary. For low-view videos ($< 300$), increments were written directly to master database rows. Once a video crossed 300 views, YouTube diverted the counter into an asynchronous verification pipeline. Background workers analyzed server logs, validated user watch durations, filtered bot farms, and cross-referenced IP addresses. Once validated, the true count jumped suddenly from $301$ to $250,000+$.
- **Modern Architecture:** Today, YouTube decouples the display into a **Dual-Engine System**: a fast-path real-time approximate counter for the public video page, and an immutable, fraud-audited ledger in Google Spanner for creator monetization and royalty disbursements.

#### 2. The PSY "Gangnam Style" 32-Bit Integer Overflow (December 2014)
- **The Event:** PSY's "Gangnam Style" became the first YouTube video to exceed $2,147,483,647$ views ($2^{31} - 1$), the maximum capacity of a signed 32-bit integer.
- **The Bug:** The view counter overflowed into negative numbers (displaying `-2,147,483,648`), exposing that YouTube's core column type was `INT4`.
- **The Fix:** Google engineers upgraded all storage columns across Vitess, BigQuery, and caching tiers to 64-bit signed integers (`BIGINT` / `INT8`), raising capacity to $9.22 \times 10^{18}$ views.

#### 3. Reddit Election Night Cassandra Upvote Meltdown
- **The Event:** During high-traffic election nights, millions of concurrent Reddit users upvoted breaking political threads.
- **The Failure:** Reddit originally persisted upvotes directly into Apache Cassandra rows. Because Cassandra handles updates and deletes as append-only **SSTables with tombstones**, the massive burst of counter column updates generated an unmanageable volume of SSTables. Compaction threads fell behind, disk I/O saturated at 100%, and read latencies spiked to tens of seconds, causing votes to disappear intermittently. Reddit re-architected upvotes into an in-memory Kafka ingestion queue feeding an asynchronous aggregation service written in Go.

---

## 2. The Core Technical Problem: Database Hotspots & Mutex Contention

If you implement a video view or post like counter using a standard relational query:

```sql
UPDATE videos 
SET views_count = views_count + 1 
WHERE video_id = 'v_mrbeast_viral';
```

At 100,000 concurrent requests per second, this naive query immediately triggers catastrophic system failure:

```
                            ROW-LEVEL MUTEX BOTTLENECK
                            
  [Thread 1] ──► Acquires Exclusive (X) Row Lock ──► Writes WAL ──► Flushes ──► Releases Lock
  [Thread 2] ──────► WAITS IN QUEUE...
  [Thread 3] ──────► WAITS IN QUEUE...
  [Thread N] ──────► WAITS IN QUEUE... (Connection pool exhausts in 150ms!)
```

### The Breakdown Mechanics:
1. **Row-Level Mutex Contention:** Relational storage engines (MySQL InnoDB, PostgreSQL) must serialize writes to the exact same physical disk page. An exclusive (`X`) lock is held for the duration of the update, page mutation, and Write-Ahead Log (WAL) append.
2. **Thread Starvation & Context Switching:** As thousands of worker threads wait for the lock, the operating system kernel spends 90% of CPU time performing thread context switches (`futex` system calls), reducing useful database work to near zero.
3. **Connection Pool Exhaustion:** Application connection pools (`HikariCP`, `pgbouncer`) drain within milliseconds. Non-counter queries (such as user logins and comments) are starved of connections, taking down the entire website.
4. **Replication Lag Avalanche:** Replicas executing the binary log (binlog) sequentially fall minutes behind the primary, resulting in stale reads worldwide.

---

## 3. Deep Dive: How YouTube Counts Video Views

YouTube's view-counting architecture processes billions of daily playback events while defending against view-fraud, bots, and metric inflation.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   YOUTUBE VIEW COUNTING PIPELINE                                       │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [Web / Mobile Client]
          │
          │ 1. Playback Heartbeat (every 5-10s with signed view token)
          ▼
 [Global Anycast Edge / Google Frontends (GFE)]
          │
          │ 2. Edge validation (Rate limiting, WAF, IP reputation scoring)
          ▼
 [YouTube Video Serving Service (C++ / Go)]
          │
          ├───► FAST PATH (Low Latency UI)
          │     In-Memory Distributed Buffer / Redis / Vitess Shards ──► Public Video Page (Approximate)
          │
          └───► SLOW PATH (Canonical Monetization & Verification)
                Produce raw event
                ▼
      [Google Cloud Pub/Sub (Kafka equivalent)]
                │
                ▼
      [Stream Processing Engine (MillWheel / Apache Beam)]
          │ - Deduplication window (30-second sliding session deduplication)
          │ - Watch-time validation (>= 30s duration or full short video)
          │ - Anti-Fraud ML Model (Detects headless browsers, botnets, loops)
          │
          ├───────────────────────────────────┐
          ▼ (Valid Views)                     ▼ (Flagged / Suspicious Views)
      [Bigtable Aggregator]              [Review Queue / BigQuery Sandbox]
          │                                   │
          │ Micro-batch aggregation           │ Offline ML batch auditing
          ▼                                   ▼
      [Google Cloud Spanner / Vitess] ◄───────┘ Nightly reconciliation adjustments
      (Canonical Financial Ledger)
```

### 1. What Qualifies as a YouTube "View"?
A view is **not** an HTTP request for video metadata or a page load:
- **Intentional Initiation:** The user must explicitly click play, or autoplay must run with clear viewport visibility and user focus.
- **Minimum Watch Duration:** The viewer must watch at least **30 seconds** of long-form content (or the full duration if the video is shorter than 30 seconds). Shorts use custom percentage-of-completion thresholds.
- **Anti-Loop / Anti-Spam Filtering:** A single user refreshing a browser tab 100 times does not yield 100 views. YouTube's deduplication engine caps incremental views per user session over a rolling time window.

### 2. The Dual-Engine Counter Architecture
- **Engine A: Fast-Path Counter (Approximate & Real-Time):** Displays on the video watch page. Reads from distributed in-memory caches and sharded database rows. Updates within seconds, providing immediate feedback to creators and viewers.
- **Engine B: Slow-Path Counter (Audited & Canonical):** Runs deep offline ML verification, IP cluster analysis, and click-fraud filtering. Feeds the YouTube Studio creator dashboard, partner ad revenue payouts, and public Billboard charts. If fraud is detected 12 hours later, the slow path applies negative adjustment deltas to the public counter.

---

## 4. Deep Dive: How Instagram Counts Views & Unique Accounts Reached

On Instagram, every Reel, Story, and Feed post displays two fundamentally distinct metrics:
1. **Views / Plays:** Total cumulative video playbacks (including repeat views by the same account).
2. **Accounts Reached (Unique Reach):** The deduplicated number of distinct Instagram accounts that saw the media item at least once during a time range.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         INSTAGRAM / META REAL-TIME ANALYTICS PIPELINE                                  │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

    [Instagram App / Reels Viewport]
                 │
                 │ Viewability Event (50% pixels visible >= 1 second)
                 ▼
    [Meta Edge Proxies / Scribe Logging Ingestion Engine]
                 │
                 ▼
    [Apache Kafka / Meta LogDevice]
    Topic: "reels_telemetry" (Partitioned by Hash(post_id))
                 │
                 ├─────────────────────────────────────────┐
                 ▼                                         ▼
    [Apache Flink Stream Processor]           [S3 / Hive Cold Storage]
    (Windowed In-Memory Processing)                        │
         │                                                 ▼
         ├── 1. Raw Plays: Count increments           [Meta Presto / Scuba]
         │      Flush: Batch every 5s to Cassandra     (Deep Historical Analytics)
         │
         └── 2. Unique Reach:
                HyperLogLog (HLL) Register Buffer
                Registers: 12 KB per post per time bucket
                Merge registers into Redis / RocksDB
         │
         ▼
    [Cassandra Cluster / Tao (Social Graph Cache)]
    (Stores aggregated hourly counters & HLL byte sketches)
         │
         ▼
    [Instagram Graph API / Insights UI]
    Sub-millisecond HLL Cardinality Estimation!
```

### The Scale Problem: Why Exact Deduplication Fails
Suppose an Instagram Reel from Cristiano Ronaldo reaches **100,000,000 users**:
- A naive exact deduplication approach using `COUNT(DISTINCT user_id)` requires storing 100 million 64-bit integer IDs in memory or a database set.
- $100{,}000{,}000 \times 8\text{ bytes} \approx 800\text{ MB}$ of memory **for a single viral post**!
- Meta hosts hundreds of billions of media objects. Storing 800 MB per post just to calculate unique reach would require exabytes of redundant RAM.
- **The Solution:** **HyperLogLog (HLL)** probabilistic sketches, which track reach up to billions of users with a fixed **12 KB memory footprint** and an error rate of $\approx 0.81\%$.

---

## 5. Architectural Approaches for Volatile Counters (HLD & LLD)

We now break down the core architectural patterns used to scale volatile counters, detailing both their **High-Level Design (HLD)** and **Low-Level Design (LLD)**.

---

## Approach 1: Sharded Database Counters (Slotting at the DB Layer)

### High-Level Design (HLD)

When strict relational database durability is mandatory (e.g., ticket inventory counts, bank balances, or auditable ledgers), we eliminate single-row lock contention by **sharding the single logical counter into $N$ physical rows (slots)**.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       SHARDED COUNTER ARCHITECTURE                                     │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                 Incoming Writes (10,000 RPS)
                              │
               ┌──────────────┼──────────────┐
               │ Hash(Random) % N (e.g., N = 20)
               ▼              ▼              ▼
         ┌───────────┐  ┌───────────┐  ┌───────────┐
         │  Slot 0   │  │  Slot 1   │  │ Slot N-1  │  (20 separate physical rows =
         │ count: 50 │  │ count: 48 │  │ count: 52 │   Lock contention reduced by 95%!)
         └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
               │              │              │
               └──────────────┼──────────────┘
                              │
                              ▼
                     Read: SELECT SUM(count)
                     (Result: 150 total views)
```

---

### Low-Level Design (LLD)

#### 1. Relational Schema Design

```sql
CREATE TABLE media_counter_shards (
    media_id VARCHAR(64) NOT NULL,
    shard_id INT NOT NULL,
    count BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (media_id, shard_id)
);

CREATE INDEX idx_media_shards ON media_counter_shards (media_id);
```

#### 2. Atomic Write & Read Operations

```sql
-- WRITE: Distribute writes randomly across 20 slots to reduce lock contention by 20x
INSERT INTO media_counter_shards (media_id, shard_id, count)
VALUES ('post_cr7_9021', FLOOR(RANDOM() * 20), 1)
ON CONFLICT (media_id, shard_id) 
DO UPDATE SET count = media_counter_shards.count + 1;

-- READ: Aggregate all 20 slots
SELECT SUM(count) AS total_count 
FROM media_counter_shards 
WHERE media_id = 'post_cr7_9021';
```

#### 3. Low-Level Concurrency & Cache Line Optimization
- **Slot Selection Strategy:** Instead of pure random selection (`RANDOM() % N`), high-performance systems use **Thread-Affinity Slotting** (`Thread.currentThread().getId() % N`). This ensures that worker threads on the same CPU core hit the same slot, maximizing CPU L1/L2 cache hits and preventing cross-core cache invalidation.
- **Tradeoffs:**
  - *Pros:* Works entirely within standard relational databases (PostgreSQL, MySQL). Provides ACID durability.
  - *Cons:* Reading the count requires an aggregation query (`SUM()`) across $N$ rows, which is slower than a single-row read. Dynamic resizing of $N$ under unexpected traffic spikes is difficult.

---

## Approach 2: Distributed In-Memory Buffer with Write-Back Caching (Redis)

### High-Level Design (HLD)

To handle hundreds of thousands of increments per second without touching disk storage, increments are absorbed in-memory by **Redis**, while an asynchronous worker flushes accumulated counter deltas to the persistent database in micro-batches.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              IN-MEMORY BUFFER & WRITE-BACK ARCHITECTURE                                │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [100,000 Client Requests/sec]
              │
              ▼
    [API Gateway / App Tier]
              │
              │ 1. Atomic INCRBY (Sub-millisecond RAM update)
              ▼
    [Redis In-Memory Cluster] ──► Reads: Instant GET video:101:views
              │
              │ 2. Add video_id to "dirty_videos" Set
              ▼
    [Asynchronous Write-Back Worker] (Runs every 5 seconds)
              │
              │ 3. Atomic GETSET delta to 0
              ▼
    [PostgreSQL / MySQL Primary]
    (Single batch update: UPDATE videos SET views = views + delta)
    *** 100,000 DB queries collapsed into 1 DB query every 5 seconds! ***
```

---

### Low-Level Design (LLD)

#### 1. Redis Command Pipeline (Python / Redis-Py)

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def record_view_fastpath(video_id: str, count: int = 1):
    pipe = r.pipeline(transaction=False)
    
    # 1. Increment the real-time public display counter
    pipe.incrby(f"video:{video_id}:views", count)
    
    # 2. Accumulate delta for the persistent DB write-back worker
    pipe.incrby(f"video:{video_id}:delta", count)
    
    # 3. Mark this video ID as dirty (needs flushing)
    pipe.sadd("dirty_video_counters", video_id)
    
    pipe.execute()

def get_public_view_count(video_id: str) -> int:
    val = r.get(f"video:{video_id}:views")
    return int(val) if val else 0
```

#### 2. Atomic Flush Worker (Solving the Lost Update Race Condition)
A common flaw in naive write-back implementations is reading the counter and then resetting it:
1. Worker reads `delta = 500`.
2. New requests arrive: Redis increments `delta` to `505`.
3. Worker sets `delta = 0`.
4. **Data Loss:** 5 views were erased!

The production solution uses the atomic `GETSET` (or Redis 6.2+ `GETDEL`) command:

```python
def sync_deltas_to_relational_db(db_connection):
    dirty_video_ids = r.smembers("dirty_video_counters")
    
    for raw_id in dirty_video_ids:
        video_id = raw_id.decode('utf-8')
        
        # ATOMIC SWAP: Atomically retrieve current delta and reset key to 0!
        # Any increments arriving during this command will accumulate into the new 0 value.
        delta = r.getset(f"video:{video_id}:delta", 0)
        
        if delta and int(delta) > 0:
            cursor = db_connection.cursor()
            cursor.execute(
                """
                UPDATE videos 
                SET views_count = views_count + %s, 
                    updated_at = NOW() 
                WHERE video_id = %s
                """,
                (int(delta), video_id)
            )
            db_connection.commit()
            
        # Remove from dirty set
        r.srem("dirty_video_counters", raw_id)
```

#### 3. Tradeoffs & Failure Modes
- **Pros:** Blazing fast. Absorbs $100{,}000+$ writes/sec per Redis node.
- **Cons:** If Redis crashes before the 5-second sync worker runs, uncommitted deltas in RAM are lost. (Mitigated using Redis **AOF with `appendfsync everysec`**).

---

## Approach 3: Stream Aggregation Pipeline with Micro-Batching (Kafka & Flink)

### High-Level Design (HLD)

For planetary scale (YouTube, TikTok, Netflix), counter increments are treated as an **immutable event stream**. Events are ingested through a distributed log (**Apache Kafka**) and aggregated into sliding or tumbling time windows by a stateful stream processor (**Apache Flink**).

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              STREAM AGGREGATION & TUMBLING WINDOW ARCHITECTURE                         │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [Edge Ingress] ──► [Kafka Topic: 'view-events'] (Partitioned by Hash(video_id))
                           │
                           ▼
          [Apache Flink Stateful Stream Workers]
          ┌───────────────────────────────────────────────────────┐
          │ KeyBy(event.video_id)                                 │
          │ Tumbling Window: 5 Seconds                            │
          │                                                       │
          │ Window Buffer:                                        │
          │   video_101: 42,100 events ──► Aggregate sum: 42,100  │
          │   video_102: 8,300 events  ──► Aggregate sum: 8,300   │
          └──────────────────────────┬────────────────────────────┘
                                     │
                                     │ Emits single batched increment record per window
                                     ▼
                   [Two-Phase Commit Sink Function]
                   ┌─────────────────┴─────────────────┐
                   ▼                                   ▼
          [Redis Cluster Cache]               [Google Spanner / Cassandra]
          (Sub-millisecond UI reads)          (Durable ACID Ledger)
```

---

### Low-Level Design (LLD)

#### 1. Flink Streaming Pipeline Implementation (Java)

```java
public class HighThroughputViewAggregator {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Exactly-once state checkpointing every 10 seconds
        env.enableCheckpointing(10000, CheckpointingMode.EXACTLY_ONCE);
        env.setStateBackend(new EmbeddedRocksDBStateBackend());

        DataStream<ViewEvent> stream = env.addSource(
            new FlinkKafkaConsumer<>("view-events", new ViewEventDeserializer(), kafkaProperties)
        );

        stream
            // 1. Filter bot events and invalid watch durations
            .filter(event -> event.getWatchTimeSeconds() >= 30 && !event.isSuspectedBot())
            // 2. Partition stream by video_id to ensure local state aggregation
            .keyBy(ViewEvent::getVideoId)
            // 3. Tumbling 5-second processing time window
            .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
            // 4. Reduce function: running count increment
            .reduce(new ReduceFunction<ViewEvent>() {
                @Override
                public ViewEvent reduce(ViewEvent v1, ViewEvent v2) {
                    v1.setCount(v1.getCount() + v2.getCount());
                    return v1;
                }
            })
            // 5. Sink batched totals to database & cache
            .addSink(new ProductionDatabaseIncrementSink());

        env.execute("YouTube-Scale View Aggregator");
    }
}
```

#### 2. Why Partitioning by `video_id` is Mandatory
In Kafka, if you publish events with a `null` key, messages are distributed round-robin across partitions. Every Flink consumer would receive a fraction of the views for `video_101`, requiring expensive cross-network shuffling. 

By setting `KafkaMessage.key = video_id`, all events for the same video land on the **exact same Kafka partition and Flink worker node**, enabling pure local in-memory summation with zero network shuffle overhead!

---

## Approach 4: HyperLogLog (HLL) for Massive Cardinality & Unique Reach

### High-Level Design (HLD)

HyperLogLog (invented by Philippe Flajolet in 2007) is a probabilistic cardinality estimation algorithm. It answers: **"How many unique users visited or viewed this entity?"** using a fixed, microscopic memory footprint (**12 KB**).

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               HYPERLOGLOG BIT ANATOMY & REGISTERS                                      │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 Stream Element: "user_id:98432"
          │
          │ 64-bit Uniform Hash (e.g., MurmurHash64A)
          ▼
 Hash Value (Binary):
 ┌───────────────────────────┬──────────────────────────────────────────────────────────────────┐
 │ 0 0 1 0 1 0 1 1 0 0 1 0 1 │ 0 0 0 0 0 1 0 1 1 0 1 1 0 1 0 1 0 0 1 0 1 0 1 1 1 0 ...        │
 └───────────────────────────┴──────────────────────────────────────────────────────────────────┘
  ▲                         ▲ ▲
  │  First 14 bits (b = 14) │ │  Remaining 50 bits (w)
  │  Determine Register ID   │ └──► Count leading zeros until first '1':
  │  Bucket index j = 1,381  │      Zeros = 5  ==> Run length rho(w) = 6
  └──────────────────────────┘
                │
                ▼
  Register Array M (m = 2^14 = 16,384 registers, each exactly 6 bits wide):
  ┌───────────┬───────────┬─────────────┬───────────┬───────────┬──────────────┐
  │ M[0] = 3  │ M[1] = 0  │ ...         │ M[1381]=6 │ ...       │ M[16383] = 4 │
  └───────────┴───────────┴─────────────┴───────────┴───────────┴──────────────┘
                                              ▲
                                              │ Updated: M[1381] = max(previous, 6)
```

---

### Low-Level Design (LLD)

#### 1. The Coin Toss Analogy & Mathematical Intuition
- If you flip a coin repeatedly until you see Heads, the probability of seeing $k$ consecutive Tails followed by Heads is:
  $$P(\text{Tails}^k, \text{Heads}) = \left(\frac{1}{2}\right)^{k+1}$$
- If someone tells you their longest sequence was $10$ consecutive Tails before Heads, your best estimate is that they flipped the coin approximately $2^{11} \approx 2,048$ times.
- By hashing user IDs into uniform pseudo-random binary strings and observing the longest sequence of leading zeros, we estimate unique cardinality!

#### 2. Why Exactly 12 Kilobytes? The Bit-Level Proof
1. **Number of Registers ($m$):** Redis sets $b = 14$, giving $m = 2^{14} = 16{,}384$ registers.
2. **Bits per Register:**
   - The remaining hash width is $64 - 14 = 50\text{ bits}$.
   - The maximum possible run of leading zeros in a 50-bit string is $50$.
   - The smallest integer power of 2 that holds the number 50 is $2^6 = 64$.
   - Therefore, each register is **exactly 6 bits wide**.
3. **Total Memory Calculation:**
   $$\text{Total Bits} = 16{,}384 \times 6 = 98{,}304\text{ bits}$$
   $$\text{Total Bytes} = \frac{98{,}304}{8} = 12{,}288\text{ bytes} = \mathbf{12\text{ KB}}$$

#### 3. The Estimation Formula: Harmonic Mean
To eliminate positive skew caused by random outlier hashes, HLL computes the **Harmonic Mean** of the register values:

$$E = \alpha_m \cdot m^2 \cdot \left( \sum_{j=0}^{m-1} 2^{-M[j]} \right)^{-1}$$

Where:
- $m = 16{,}384$
- $M[j]$ is the value stored in register $j$
- $\alpha_m \approx 0.7213$ (normalization constant for $m \ge 128$)
- **Standard Error:** $\sigma = \frac{1.04}{\sqrt{m}} = \frac{1.04}{\sqrt{16384}} \approx \mathbf{0.81\%}$.

#### 4. Microsecond Merging (`PFMERGE`)
Because each register stores the **maximum** run of zeros, two or more HLL sketches can be merged losslessly in microsecond CPU time without accessing the original events:

$$M_{\text{merged}}[j] = \max(M_A[j], M_B[j]) \quad \forall j \in [0, m-1]$$

This allows Instagram to maintain daily reach keys (`reach:2024-10-01`, `reach:2024-10-02`) and compute the **30-day deduplicated monthly reach** via a single `PFMERGE` command in $< 2\text{ms}$!

---

## Approach 5: Count-Min Sketch (CMS) for Heavy Hitters & Top-K Trends

### High-Level Design (HLD)

While HyperLogLog counts unique elements, **Count-Min Sketch (CMS)** estimates the **frequency** of events in a massive stream. It is used by Twitter/X, Twitch, and TikTok to track trending hashtags, viral search queries, and top chat emojis without storing millions of individual strings.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       COUNT-MIN SKETCH 2D MATRIX                                       │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 Event: "hashtag:#worldcup"
          │
          ├── Hash_1(event) % Width w ──► Row 0, Col 412 ──► Increment counter: C[0][412] += 1
          ├── Hash_2(event) % Width w ──► Row 1, Col 89  ──► Increment counter: C[1][89]  += 1
          ├── Hash_3(event) % Width w ──► Row 2, Col 1205 ──► Increment counter: C[2][1205] += 1
          └── Hash_d(event) % Width w ──► Row d, Col 634 ──► Increment counter: C[d][634] += 1
          
 To Query Frequency of "hashtag:#worldcup":
 Return MINIMUM across all hashed cells:
 Estimate = min( C[0][412], C[1][89], C[2][1205], C[d][634] )
 (Taking the MINIMUM eliminates hash collision over-estimations!)
```

---

### Low-Level Design (LLD)

#### 1. Python Implementation of Count-Min Sketch

```python
import hashlib
import math

class CountMinSketch:
    def __init__(self, epsilon: float = 0.001, delta: float = 0.01):
        # Width w = ceil(e / epsilon)
        self.w = int(math.ceil(math.e / epsilon))
        # Depth d = ceil(ln(1 / delta))
        self.d = int(math.ceil(math.log(1.0 / delta)))
        # 2D table initialized to 0
        self.table = [[0] * self.w for _ in range(self.d)]

    def _hash(self, item: str, seed: int) -> int:
        md5 = hashlib.md5(f"{seed}:{item}".encode('utf-8')).hexdigest()
        return int(md5, 16) % self.w

    def add(self, item: str, count: int = 1):
        for i in range(self.d):
            col = self._hash(item, seed=i)
            self.table[i][col] += count

    def estimate(self, item: str) -> int:
        # Frequency estimate is the minimum across all rows
        return min(self.table[i][self._hash(item, seed=i)] for i in range(self.d))
```

#### 2. Heavy Hitters (Top-K) Architecture
To find the top 10 trending items:
- Pair the Count-Min Sketch with a **Min-Heap of size K**.
- When an item is incremented in the CMS, query its estimated frequency.
- If the frequency exceeds the minimum element in the heap, insert or update the item in the heap.
- The heap maintains the real-time Top-K viral entities in $O(\log K)$ time.

---

## Approach 6: Single-Node Concurrency: Java `LongAdder` Internals

### High-Level Design (HLD)

On a single high-performance server (e.g., an API gateway or microservice aggregating in-memory counters), traditional atomic primitives like `AtomicLong` fail under high thread contention:

```
                          ATOMICLONG CAS CONTENTION COLLAPSE
                          
  [128 CPU Threads] ──► Compare-And-Swap (CAS) on a single 64-bit memory address
                              │
                              ▼
  - 1 Thread succeeds!
  - 127 Threads fail their CAS loop, spin furiously, burning 100% CPU on cache coherence buses!
```

Java's **`LongAdder`** (introduced in Java 8) solves this using **internal cell striping**:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       JAVA LONGADDER CELL STRIPING                                     │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [Thread 0, 1] ────────► Cell[0] (value: 412) ──► Padded to 64 bytes (1 Cache Line)
  [Thread 2, 3] ────────► Cell[1] (value: 389) ──► Padded to 64 bytes (1 Cache Line)
  [Thread 4, 5] ────────► Cell[2] (value: 405) ──► Padded to 64 bytes (1 Cache Line)
  [Thread 6, 7] ────────► Cell[3] (value: 420) ──► Padded to 64 bytes (1 Cache Line)
                              │
                              ▼
                 Sum(): Cell[0] + Cell[1] + Cell[2] + Cell[3] = 1,626
                 Zero lock contention! Throughput scales linearly with CPU cores!
```

---

### Low-Level Design (LLD)

#### 1. Cache Line Padding & False Sharing (`@Contended`)
In multi-core CPU architectures, memory is transferred between RAM and CPU caches in chunks of **64 bytes (a Cache Line)**. 

If two separate variables reside on the same 64-byte cache line, when CPU Core 1 writes to Variable A, it forcefully invalidates CPU Core 2's L1 cache line containing Variable B! This phenomenon is known as **False Sharing**.

Inside `LongAdder`, each `Cell` is annotated with `@jdk.internal.vm.annotation.Contended`:
```java
@jdk.internal.vm.annotation.Contended
static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return VALUE.compareAndSet(this, cmp, val);
    }
}
```
The JVM inserts 128 bytes of padding around each cell, guaranteeing that each cell occupies its own dedicated L1 cache line, eliminating cache-coherence bus saturation completely.

---

## 6. Comprehensive Architectural Comparison Matrix

| Architectural Pattern | Max Throughput (OPS) | Read Latency | Accuracy / Precision | Memory Overhead | Complexity | Primary Industry Use Case |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Sharded DB Counters** | $10,000 - 30,000$ | $10 - 50\text{ ms}$ | $100\%$ Exact | Low (Relational rows) | Low | Ticket inventory, bank balances |
| **Redis Write-Back Cache** | $100,000 - 250,000$ | Sub-$1\text{ ms}$ | $100\%$ Exact (lossy on crash) | Moderate (RAM) | Medium | Post likes, page views, upvotes |
| **Kafka + Flink Streaming** | $1,000,000+$ | $1 - 5\text{ sec}$ (Windowed) | $100\%$ Exact | High (Cluster footprint) | High | YouTube view pipeline, ad billing |
| **HyperLogLog (HLL)** | $500,000+$ | Sub-$2\text{ ms}$ | $\approx 0.81\%$ Error | **Strictly 12 KB** | Medium | Accounts Reached, DAU/MAU, Unique IPs |
| **Count-Min Sketch (CMS)** | $500,000+$ | Sub-$1\text{ ms}$ | Tunable over-estimate | Fixed Matrix ($w \times d$) | Medium | Trending hashtags, Top-K viral items |
| **Java `LongAdder`** | $50,000,000+$ | Nanoseconds | $100\%$ Exact | Bytes per process | Very Low | In-process server telemetry |

---

## 7. Real-World Production Post-Mortems & Anti-Patterns

### Anti-Pattern 1: The "COUNT(DISTINCT) Over 50 Million Rows" Trap
- **The Incident:** An engineering team attempted to compute the unique reach of a marketing campaign by running:
  ```sql
  SELECT COUNT(DISTINCT user_id) 
  FROM impression_events 
  WHERE campaign_id = 'summer_promo_2024';
  ```
- **The Crash:** As impressions crossed 60 million, PostgreSQL allocated huge `work_mem` hash aggregates. Memory exhausted, queries spilled to disk (producing 40GB of temporary files), and query latency exceeded 120 seconds, causing frontend timeouts.
- **The Fix:** Maintain a Redis HyperLogLog key per campaign. Increments execute via `PFADD` in sub-milliseconds; unique reach queries execute via `PFCOUNT` in under 1 millisecond.

### Anti-Pattern 2: The Redis Hot Key Shard Overload
- **The Incident:** During a viral tweet storm, an un-sharded Redis cluster key `tweet:98124:likes` was hit with 150,000 `INCR` commands/second. Because all commands for a single key target the exact same Redis hash slot and single-threaded CPU core, that specific Redis shard pinned at 100% CPU while the remaining 31 nodes sat idle.
- **The Fix:** Apply **Client-Side Key Striping**:
  ```python
  # Distribute hot key across 10 sub-keys
  sub_key = f"tweet:{tweet_id}:likes:{random.randint(0, 9)}"
  redis.incr(sub_key)
  ```

---

## 8. System Design Interview Blueprint: "Design View & Like Counter for YouTube"

When asked to design a high-throughput view counter in a system design interview:

1. **Clarify Scope & Counter Semantics:**
   - Are we counting total views or unique viewers?
   - What constitutes a view (e.g., 30s minimum watch time, bot filtering)?
   - Is eventual consistency acceptable (e.g., 5-10 second propagation delay)?

2. **Propose the Two-Tier Dual Engine Architecture:**
   - **Fast-Path Engine:** Redis In-Memory Increment with Write-Back to provide real-time UI counters.
   - **Slow-Path Verification Engine:** Kafka event stream $\to$ Flink tumbling window $\to$ Anti-Fraud ML $\to$ Google Spanner canonical ledger.

3. **Handle Unique Reach Separately:**
   - Explain why `COUNT(DISTINCT)` collapses at scale ($800\text{ MB}$ per viral post).
   - Propose **HyperLogLog (HLL)**, proving why **12 KB** memory guarantees $\approx 0.81\%$ standard error.
   - Demonstrate multi-day reach rollups via microsecond `PFMERGE` unions.

4. **Address Hotspots Proactively:**
   - Mention Hot-Key striping on Redis clusters.
   - Detail Java `LongAdder` cache line padding (`@Contended`) for high-throughput single-node ingress gateways.
