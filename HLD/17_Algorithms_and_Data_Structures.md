# ⚙️ Algorithms & Data Structures for System Design

> *"Knowing what algorithm to use — and why — is what separates a senior engineer from a principal engineer."*

---

## 📋 Table of Contents

1. [Rate Limiting Algorithms](#rate-limiting)
2. [Consistent Hashing (In Depth)](#consistent-hashing)
3. [Bloom Filters](#bloom-filters)
4. [Count-Min Sketch (Frequency Estimation)](#count-min-sketch)
5. [HyperLogLog (Unique Count at Scale)](#hyperloglog)
6. [Top-K / Heavy Hitters Problem](#top-k)
7. [Trie (Prefix Search & Autocomplete)](#trie)
8. [Geohash & QuadTree (Proximity Search)](#geospatial)
9. [LRU & LFU Cache Eviction](#cache-eviction)
10. [Leader Election Algorithms](#leader-election)
11. [Checksums & Merkle Trees](#merkle-trees)
12. [Skip Lists](#skip-lists)
13. [When to Use Which Algorithm](#decision-guide)
14. [Key Takeaways](#key-takeaways)

---

## 🚦 Rate Limiting Algorithms

Rate limiting protects services from being overwhelmed. The choice of algorithm determines the granularity of control and the behavior during bursts.

### Algorithm 1: Token Bucket

**Metaphor:** A bucket holds tokens. Each request consumes one token. Tokens are added at a fixed rate. If the bucket is empty, the request is rejected.

```
Bucket capacity: 100 tokens (max burst size)
Refill rate: 10 tokens/second

t=0s:   bucket=100 tokens
t=0s:   100 requests arrive → all accepted (bucket now empty)
t=1s:   10 requests arrive → only 10 tokens refilled → 10 accepted, 0 rejected
t=5s:   bucket has refilled to 50 tokens → 50 requests accepted
```

**Key property:** Allows **bursting** up to the bucket capacity, then smooths to the refill rate.

```python
import time
import threading

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity:    max tokens (max burst size)
        refill_rate: tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = threading.Lock()

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now

    def consume(self, tokens: int = 1) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True   # Request allowed
            return False      # Rate limited

# Usage
limiter = TokenBucket(capacity=100, refill_rate=10)
if limiter.consume():
    handle_request()
else:
    return "429 Too Many Requests"
```

**Distributed Token Bucket (Redis):**

```python
import redis
import time

r = redis.Redis()

def token_bucket_redis(user_id: str, capacity: int, refill_rate: float) -> bool:
    key = f"ratelimit:token:{user_id}"
    now = time.time()

    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(key)
                data = pipe.hgetall(key)

                tokens = float(data.get(b'tokens', capacity))
                last_refill = float(data.get(b'last_refill', now))

                # Refill tokens based on elapsed time
                elapsed = now - last_refill
                tokens = min(capacity, tokens + elapsed * refill_rate)

                if tokens < 1:
                    pipe.unwatch()
                    return False  # Rate limited

                tokens -= 1

                pipe.multi()
                pipe.hset(key, mapping={'tokens': tokens, 'last_refill': now})
                pipe.expire(key, int(capacity / refill_rate) + 1)
                pipe.execute()
                return True  # Allowed

            except redis.WatchError:
                continue  # Retry on concurrent modification
```

**✅ Use when:** You want to allow short bursts (API clients can send a burst of requests) but smooth out over time.

---

### Algorithm 2: Leaky Bucket

**Metaphor:** Requests enter a queue (the "bucket"). Requests are processed (leaked) at a fixed rate. If the queue is full, new requests are dropped.

```
Queue capacity: 100 requests
Processing rate: 10 requests/second

1000 requests arrive instantly:
  First 100 → queued
  Remaining 900 → DROPPED immediately
  
Queued 100 requests processed at 10/second over 10 seconds.
Output is smooth: exactly 10 requests/second, always.
```

**Key property:** **No bursting**. Output rate is always fixed. Smooths traffic.

```python
import queue
import threading
import time

class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        self.bucket = queue.Queue(maxsize=capacity)
        self.leak_rate = leak_rate

    def add_request(self, request) -> bool:
        try:
            self.bucket.put_nowait(request)
            return True   # Accepted (queued)
        except queue.Full:
            return False  # Dropped (bucket full)

    def start_processing(self, processor):
        """Process requests at fixed rate"""
        def _leak():
            while True:
                try:
                    request = self.bucket.get(timeout=1)
                    processor(request)
                except queue.Empty:
                    pass
                time.sleep(1 / self.leak_rate)  # Process 1 per (1/rate) seconds

        thread = threading.Thread(target=_leak, daemon=True)
        thread.start()
```

**✅ Use when:** You need a perfectly smooth, predictable outflow rate (e.g., sending to a downstream API with strict rate limits, sending SMS at exactly 100/second).

---

### Algorithm 3: Fixed Window Counter

**How it works:** Count requests in a fixed time window (e.g., current minute). If count exceeds limit → reject.

```
Window: every 1 minute (wall-clock aligned)
Limit: 100 requests/minute

00:00 - 01:00: user sends 99 requests → OK
                   ↑ window resets at 01:00
01:00 - 02:00: user sends 100 requests → OK

Attack: 
  00:59: send 100 requests → accepted (1 second left in window)
  01:00: window resets, send 100 more → accepted
  = 200 requests in 2 seconds! ← BURST ATTACK AT WINDOW BOUNDARY
```

```python
import redis
import time

def fixed_window(user_id: str, limit: int, window_seconds: int) -> bool:
    now = int(time.time())
    window_key = now // window_seconds  # e.g., at 10:05:30 with 60s window → key=1705312
    
    key = f"ratelimit:fixed:{user_id}:{window_key}"
    count = r.incr(key)
    
    if count == 1:
        r.expire(key, window_seconds)  # Set expiry on first request
    
    return count <= limit
```

**✅ Use when:** Simple use case, burst attacks at window boundaries are acceptable, or windows are very short.

---

### Algorithm 4: Sliding Window Log

**How it works:** Store the timestamp of every request. On each new request, count timestamps in the last N seconds.

```
Window: 1 minute (rolling)
Limit: 100 requests/minute

t=10:05:00: user has 50 requests in last 60 seconds
t=10:05:01: new request arrives
  count requests from 10:04:01 to 10:05:01 → still 51 → OK

t=10:05:30: user has 99 requests in last 60 seconds
t=10:05:31: new request → 100 → OK (exactly at limit)
t=10:05:32: new request → some from 10:04:32 may have fallen off the window
  count = depends on exact timestamps → precise!
```

```python
import redis
import time

def sliding_window_log(user_id: str, limit: int, window_seconds: int) -> bool:
    key = f"ratelimit:swlog:{user_id}"
    now = time.time()
    window_start = now - window_seconds

    with r.pipeline() as pipe:
        # Remove timestamps older than window
        pipe.zremrangebyscore(key, 0, window_start)
        # Count requests in window
        pipe.zcard(key)
        # Add current request timestamp
        pipe.zadd(key, {str(now): now})
        # Expire key after window duration
        pipe.expire(key, window_seconds + 1)
        
        results = pipe.execute()
        request_count = results[1]  # Count BEFORE this request

    return request_count < limit  # If adding this would exceed limit → reject

# Memory cost: O(requests_per_window) per user
# 100 req/min limit → max 100 timestamps per user per window
# At 1M users: 1M × 100 × 16 bytes = 1.6GB — potentially large
```

**✅ Use when:** Exact precision required, small user base, or you can shard Redis.

---

### Algorithm 5: Sliding Window Counter (Hybrid — Production Standard)

Approximates the sliding window using two fixed-window counters. Avoids storing per-request timestamps.

```
Current time: 10:05:45 (45 seconds into the current minute)

Prev window count (10:04:00-10:05:00): 70 requests
Curr window count (10:05:00-10:05:59): 30 requests

Overlap factor: how much of prev window is within our rolling 60s?
  = (60 - 45) / 60 = 15/60 = 0.25 (25% of prev window overlaps)

Approximate count = (prev_count × overlap) + curr_count
                  = (70 × 0.25) + 30
                  = 17.5 + 30 = 47.5 → round to 47 or 48

If limit is 100: 47 < 100 → OK
```

```python
import redis
import time
import math

def sliding_window_counter(user_id: str, limit: int, window_seconds: int) -> bool:
    now = time.time()
    curr_window = math.floor(now / window_seconds)
    prev_window = curr_window - 1
    elapsed_in_curr = now % window_seconds
    
    curr_key = f"ratelimit:swc:{user_id}:{curr_window}"
    prev_key = f"ratelimit:swc:{user_id}:{prev_window}"

    with r.pipeline() as pipe:
        pipe.get(prev_key)
        pipe.get(curr_key)
        results = pipe.execute()

    prev_count = int(results[0] or 0)
    curr_count = int(results[1] or 0)

    # Weight: how much of the prev window falls within our rolling window
    prev_weight = (window_seconds - elapsed_in_curr) / window_seconds
    estimated_count = (prev_count * prev_weight) + curr_count

    if estimated_count >= limit:
        return False  # Rate limited

    # Increment current window counter
    with r.pipeline() as pipe:
        pipe.incr(curr_key)
        pipe.expire(curr_key, window_seconds * 2)
        pipe.execute()

    return True
```

**Memory cost:** O(1) per user (just two counters). **Accuracy:** ~0.003% error rate in practice.

**✅ Use when:** Production rate limiting at scale. Best balance of accuracy, memory, and performance. **Used by Cloudflare, Figma, Stripe.**

---

### Rate Limiting Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Boundary Attack | Best Use Case |
|---|---|---|---|---|---|
| Token Bucket | O(1) | Exact | ✅ Allows bursts | N/A | General APIs, SDKs |
| Leaky Bucket | O(capacity) | Exact | ❌ Queues bursts | N/A | Smooth outbound rate |
| Fixed Window | O(1) | Approximate | ❌ Window edge burst | ⚠️ Vulnerable | Simple, low-risk |
| Sliding Log | O(requests) | Exact | N/A | ✅ No boundary | High-precision, small scale |
| Sliding Counter | O(1) | ~99.997% | N/A | ✅ No boundary | **Production standard** |

---

## 🔗 Consistent Hashing — Deep Dive

### The Problem with Naive Sharding

```
4 cache servers: S0, S1, S2, S3
Key → Server: hash(key) % 4

hash("user:alice") = 17 → 17 % 4 = 1 → Server S1
hash("user:bob")   = 22 → 22 % 4 = 2 → Server S2

Add Server S4 (now 5 servers):
hash("user:alice") = 17 → 17 % 5 = 2 → Server S2  ← MOVED!
hash("user:bob")   = 22 → 22 % 5 = 2 → Server S2  ← same
hash("user:carol") = 31 → 31 % 5 = 1 → Server S1  ← MOVED!

On average, (N-1)/N ≈ 80% of keys remap when adding one server.
= 80% cache miss rate → stampede on backend databases!
```

### The Consistent Hash Ring

Map both servers and keys onto a **circular hash ring** (0 to 2³²). Each key is assigned to the first server clockwise from its position.

```
Hash ring: 0 ──────────────────────────────────────── 2^32

Servers placed at their hash positions:
    S0 at position 100
    S1 at position 250
    S2 at position 400
    S3 at position 700

Ring (simplified):
         S2(400)
        /        \
    S1(250)      S3(700)
        \        /
         S0(100)

Key assignment (clockwise to first server):
  key at position 150 → S1 (next clockwise from 150)
  key at position 350 → S2 (next clockwise from 350)
  key at position 500 → S3 (next clockwise from 500)
  key at position 800 → S0 (wraps around, next is S0 at 100)
```

**Adding S4 at position 300:**
```
Before: keys 250-400 go to S2
After:  keys 250-300 go to S4 (new)
        keys 300-400 still go to S2

Only keys between S1 (250) and S4 (300) move.
Average keys moved when adding 1 server to N: 1/N of total.
For N=4: only 25% move (vs 80% with modulo hashing).
```

### Virtual Nodes — Solving Load Imbalance

With 3 physical servers, they might hash to adjacent positions → uneven load.

```
Physical servers:
  S0 → position 100   (owns keys 700-100, only 30% of ring)
  S1 → position 200   (owns keys 100-200, only 10% of ring)
  S2 → position 700   (owns keys 200-700, owns 50% of ring!!)

S2 is overloaded. S1 is underloaded.
```

**Solution: Virtual Nodes (VNodes)**

Each physical server gets multiple positions on the ring (50-150 virtual nodes each).

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes: list[str], vnodes_per_node: int = 150):
        """
        nodes: list of server names (e.g., ["s0", "s1", "s2"])
        vnodes_per_node: number of virtual positions per server
        """
        self.ring: dict[int, str] = {}    # position → server name
        self.sorted_positions: list[int] = []
        self.vnodes_per_node = vnodes_per_node

        for node in nodes:
            self._add_node(node)

    def _hash(self, key: str) -> int:
        """MD5 hash → 32-bit integer position on ring"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

    def _add_node(self, node: str):
        for i in range(self.vnodes_per_node):
            # Each virtual node has a unique position
            vnode_key = f"{node}:vnode:{i}"
            position = self._hash(vnode_key)
            self.ring[position] = node
            bisect.insort(self.sorted_positions, position)

    def add_node(self, node: str):
        """Add a new server. Only ~1/N keys remapped."""
        self._add_node(node)

    def remove_node(self, node: str):
        """Remove a server. Its keys shift to next clockwise server."""
        for i in range(self.vnodes_per_node):
            vnode_key = f"{node}:vnode:{i}"
            position = self._hash(vnode_key)
            del self.ring[position]
            idx = bisect.bisect_left(self.sorted_positions, position)
            self.sorted_positions.pop(idx)

    def get_node(self, key: str) -> str:
        """Find which server owns this key."""
        if not self.ring:
            raise Exception("No nodes in ring")
        position = self._hash(key)
        # Find first position >= key's position (clockwise)
        idx = bisect.bisect_right(self.sorted_positions, position)
        # Wrap around if beyond last position
        if idx == len(self.sorted_positions):
            idx = 0
        return self.ring[self.sorted_positions[idx]]

    def get_nodes_for_replication(self, key: str, replicas: int) -> list[str]:
        """Get N unique servers for replication (next N distinct servers clockwise)."""
        if not self.ring:
            return []
        position = self._hash(key)
        idx = bisect.bisect_right(self.sorted_positions, position)
        
        seen_nodes = set()
        result = []
        for _ in range(len(self.sorted_positions)):
            if idx == len(self.sorted_positions):
                idx = 0
            node = self.ring[self.sorted_positions[idx]]
            if node not in seen_nodes:
                seen_nodes.add(node)
                result.append(node)
                if len(result) == replicas:
                    break
            idx += 1
        return result

# Usage
ring = ConsistentHashRing(["server-1", "server-2", "server-3"])

# Key routing
server = ring.get_node("user:alice")    # → "server-2" (consistent)
server = ring.get_node("user:alice")    # → "server-2" (always same)

# Replication
servers = ring.get_nodes_for_replication("product:123", replicas=3)
# → ["server-2", "server-1", "server-3"] (3 different servers)

# Add new server: only ~1/4 of keys remapped
ring.add_node("server-4")
server = ring.get_node("user:alice")    # Might now → "server-4" (if its vnode is between the key and old server)
```

### Load Distribution with VNodes

With 150 vnodes per server, load distribution is statistically near-uniform:

```
With 3 servers × 150 vnodes = 450 positions on ring
Each server expected: 1/3 = 33.3% of keys
Actual with 150 vnodes: within ±2% of ideal (Central Limit Theorem)

With 3 servers × 3 vnodes = 9 positions (too few):
  S0: 40%, S1: 18%, S2: 42% — unacceptably skewed!
  → More vnodes → better distribution (but more memory for the ring index)
```

### Real-World Usage

| System | Use of Consistent Hashing |
|---|---|
| Amazon DynamoDB | Routes key-value requests to partition servers |
| Apache Cassandra | "Token Ring" — each node owns a range of tokens |
| Memcached | Client-side consistent hashing to select cache servers |
| Redis Cluster | 16,384 hash slots distributed across nodes (similar concept) |
| Nginx upstream | Consistent hashing upstream module for sticky routing |
| Load balancers | Consistent hashing for session affinity without sticky sessions |

---

## 🌸 Bloom Filters

### The Problem

**Question:** "Is `user:alice` in our set of banned users?"

**Naive approach:** HashSet in memory → fast O(1) but requires all elements in memory.

For 1 billion banned users at 8 bytes/user = 8GB RAM — expensive.

**Bloom filter:** Probabilistic data structure that answers "Is X in the set?" with:
- **Definitely NOT in set** (100% accurate — no false negatives)
- **Probably in set** (with a small false positive rate, e.g., 1%)

Memory: **~10 bits per element** (regardless of element size).  
1 billion users × 10 bits = 1.25 GB (vs 8 GB for a hash set).

### How It Works

```
Setup:
  - Bit array of M bits (all zeros initially)
  - K independent hash functions h1, h2, ... hK

Add element "alice":
  h1("alice") = 3   → set bit[3] = 1
  h2("alice") = 17  → set bit[17] = 1
  h3("alice") = 42  → set bit[42] = 1

Add element "bob":
  h1("bob") = 7    → set bit[7] = 1
  h2("bob") = 17   → bit[17] already 1 (shared with "alice", OK)
  h3("bob") = 61   → set bit[61] = 1

Query "alice":
  h1("alice") = 3  → bit[3] = 1 ✓
  h2("alice") = 17 → bit[17] = 1 ✓
  h3("alice") = 42 → bit[42] = 1 ✓
  All bits set → "Probably in set" ✓ (correct)

Query "carol" (NOT in set):
  h1("carol") = 3   → bit[3] = 1 ✓ (set by "alice")
  h2("carol") = 99  → bit[99] = 0 ✗
  Not all bits set → "Definitely NOT in set" (correct)

Query "dave" (NOT in set, but false positive):
  h1("dave") = 7   → bit[7] = 1 ✓ (set by "bob")
  h2("dave") = 17  → bit[17] = 1 ✓ (set by "alice" or "bob")
  h3("dave") = 61  → bit[61] = 1 ✓ (set by "bob")
  All bits set → "Probably in set" ← FALSE POSITIVE (dave was never added!)
```

### Implementation

```python
import math
import hashlib

class BloomFilter:
    def __init__(self, expected_elements: int, false_positive_rate: float = 0.01):
        """
        expected_elements:  how many elements you'll add
        false_positive_rate: e.g., 0.01 = 1% false positives
        """
        # Optimal bit array size
        self.m = self._optimal_m(expected_elements, false_positive_rate)
        # Optimal number of hash functions
        self.k = self._optimal_k(self.m, expected_elements)
        self.bit_array = bytearray(math.ceil(self.m / 8))
        print(f"Bloom filter: {self.m} bits ({self.m/8/1024:.1f} KB), {self.k} hash functions")

    def _optimal_m(self, n: int, p: float) -> int:
        """Optimal bit array size: m = -n*ln(p) / (ln(2))^2"""
        return int(-n * math.log(p) / (math.log(2) ** 2))

    def _optimal_k(self, m: int, n: int) -> int:
        """Optimal hash count: k = (m/n) * ln(2)"""
        return max(1, int((m / n) * math.log(2)))

    def _get_bit_positions(self, item: str) -> list[int]:
        """Generate k bit positions using double hashing (hash1 + i*hash2)"""
        h1 = int(hashlib.md5(item.encode()).hexdigest(), 16)
        h2 = int(hashlib.sha1(item.encode()).hexdigest(), 16)
        return [(h1 + i * h2) % self.m for i in range(self.k)]

    def _set_bit(self, pos: int):
        byte_idx, bit_idx = divmod(pos, 8)
        self.bit_array[byte_idx] |= (1 << bit_idx)

    def _get_bit(self, pos: int) -> bool:
        byte_idx, bit_idx = divmod(pos, 8)
        return bool(self.bit_array[byte_idx] & (1 << bit_idx))

    def add(self, item: str):
        for pos in self._get_bit_positions(item):
            self._set_bit(pos)

    def contains(self, item: str) -> bool:
        """True = probably in set; False = definitely NOT in set"""
        return all(self._get_bit(pos) for pos in self._get_bit_positions(item))

# Example: URL deduplication for a web crawler
bf = BloomFilter(expected_elements=1_000_000_000, false_positive_rate=0.001)
# Output: Bloom filter: 14377588178 bits (1718.1 MB), 10 hash functions
# 1% false positives for 1 billion URLs with just ~1.7 GB

bf.add("https://google.com")
bf.add("https://github.com")

bf.contains("https://google.com")   # True (probably in set — correct)
bf.contains("https://reddit.com")   # False (definitely not in set — correct)
```

### Optimal Parameters

```
For n = 1 million elements, 1% false positive rate:
  m = 9.585 million bits ≈ 1.14 MB
  k = 7 hash functions

For n = 1 billion elements, 0.1% false positive rate:
  m = 14.38 billion bits ≈ 1.7 GB
  k = 10 hash functions

Compare: HashSet for 1B elements = ~8 GB → Bloom filter saves ~78%
```

### Real-World Usage

| Use Case | System | Why |
|---|---|---|
| URL deduplication | Google Crawler, web scrapers | Skip already-crawled pages |
| Cache filtering | Redis, Cassandra | Check Bloom before hitting disk |
| Spam filter | Email providers | Check if sender IP is in blocklist |
| Password breach check | HaveIBeenPwned | Check if password is in breach list (1B+ hashes) |
| SSTable filtering | Cassandra, RocksDB | Skip SSTable files that don't contain a key |
| Ad deduplication | AdTech | "Has this user seen this ad?" |
| Username availability | Any social platform | Quick "definitely available" check |

---

## 📊 Count-Min Sketch (Frequency Estimation)

### The Problem

**Question:** "How many times has each URL been clicked in the last hour?"

If you have 1 billion unique URLs, a HashMap would need 1 billion entries — too much memory.

**Count-Min Sketch:** Approximate frequency counts using O(width × depth) memory, regardless of number of unique items.

### How It Works

```
2D array of counters: width=1000 columns, depth=5 rows (5 hash functions)
All initialized to 0.

Add "url_A" (frequency 1):
  h1("url_A") = 42  → row[0][42]++
  h2("url_A") = 137 → row[1][137]++
  h3("url_A") = 891 → row[2][891]++
  h4("url_A") = 23  → row[3][23]++
  h5("url_A") = 567 → row[4][567]++

Add "url_B" (frequency 1):
  h1("url_B") = 42  → row[0][42]++ (COLLISION with url_A!)
  h2("url_B") = 200 → row[1][200]++
  ...

Query "url_A" count:
  row[0][42]  = 2  ← overcounted (shared with url_B)
  row[1][137] = 1  ← exact
  row[2][891] = 1  ← exact
  row[3][23]  = 1  ← exact
  row[4][567] = 1  ← exact
  
  Answer: MIN(2, 1, 1, 1, 1) = 1  ← correct!

The minimum across all rows cancels most collision overcounting.
Result is always >= true count (overestimate, never underestimate).
```

```python
import hashlib
import math

class CountMinSketch:
    def __init__(self, width: int = 1000, depth: int = 5):
        """
        width: number of columns (higher = fewer collisions = more accurate)
        depth: number of rows = number of hash functions (higher = lower error probability)
        
        Error guarantee: estimate ≤ true_count + (total_events / width)
        with probability 1 - (1/2)^depth
        """
        self.width = width
        self.depth = depth
        self.table = [[0] * width for _ in range(depth)]

    def _hashes(self, item: str) -> list[int]:
        result = []
        for i in range(self.depth):
            h = int(hashlib.md5(f"{item}:{i}".encode()).hexdigest(), 16)
            result.append(h % self.width)
        return result

    def add(self, item: str, count: int = 1):
        """Increment count for item"""
        for row, col in enumerate(self._hashes(item)):
            self.table[row][col] += count

    def estimate(self, item: str) -> int:
        """Return estimated frequency (always >= true frequency)"""
        return min(self.table[row][col] for row, col in enumerate(self._hashes(item)))

# Memory: 1000 × 5 × 4 bytes = 20KB regardless of unique URLs!
cms = CountMinSketch(width=10_000, depth=7)

# Process 1 billion click events
for click in click_stream:
    cms.add(click.url)

# Query
cms.estimate("/products/123")    # → 58291 (approximate but close)
cms.estimate("/homepage")        # → 1024932 (approximate)
# Actual HashMap would need billions of entries.
# CMS uses fixed 10000×7×4 = 280KB.
```

**Used by:** Twitter for trending topics, Kafka for consumer lag estimation, Redis with HyperLogLog, network routers for traffic analysis.

---

## 🔢 HyperLogLog (Unique Count Estimation)

### The Problem

**Question:** "How many unique users visited our site today?"

**Naive:** HashSet → O(n) memory. For 1 billion unique users = 8 GB.

**HyperLogLog:** Estimates cardinality (unique count) with ~1.5% error using **only 12 KB of memory**, regardless of the input size.

### The Math (Simplified)

**Intuition:** If you flip a coin many times, the maximum run of heads you observe is related to how many flips you made. A longer max run → more flips → higher cardinality.

```
For each item:
  1. Hash it to a binary string: hash("user:alice") = "00001011010..."
  2. Count leading zeros: "00001..." → 4 leading zeros
  3. Track the MAXIMUM leading zeros seen (per "bucket")

Observation: if max leading zeros = k, you've likely seen ~2^k unique items.
  max leading zeros = 0  → ~1 unique item
  max leading zeros = 10 → ~1024 unique items
  max leading zeros = 20 → ~1 million unique items
  max leading zeros = 30 → ~1 billion unique items

HyperLogLog uses m buckets (typically 2^14 = 16384) and averages across them:
  Cardinality ≈ m × harmonic_mean(2^max_zeros[i] for i in range(m))
  Error: ± 1.04 / sqrt(m) ≈ ±0.81% for m=16384
```

```python
import hashlib
import math

class HyperLogLog:
    def __init__(self, p: int = 14):
        """
        p: precision (number of bits for bucket index)
        m = 2^p buckets
        Error: ±1.04/sqrt(m)
        
        p=14: 16384 buckets, 12KB memory, ±0.81% error
        p=10: 1024 buckets, 768B memory, ±3.25% error
        """
        self.p = p
        self.m = 2 ** p         # Number of buckets
        self.M = [0] * self.m   # Max leading zeros per bucket

    def _count_leading_zeros(self, bits: int, max_bits: int) -> int:
        count = 0
        for i in range(max_bits - 1, -1, -1):
            if bits & (1 << i):
                break
            count += 1
        return count + 1  # +1 because we count the first 1 bit

    def add(self, item: str):
        h = int(hashlib.sha1(item.encode()).hexdigest(), 16)
        # Use first p bits as bucket index
        bucket = h >> (160 - self.p)  # SHA1 = 160 bits
        # Count leading zeros in remaining bits
        remainder = h & ((1 << (160 - self.p)) - 1)
        leading_zeros = self._count_leading_zeros(remainder, 160 - self.p)
        # Update max for this bucket
        self.M[bucket] = max(self.M[bucket], leading_zeros)

    def count(self) -> int:
        """Estimate cardinality"""
        alpha = 0.7213 / (1 + 1.079 / self.m)  # Bias correction factor
        Z = sum(2 ** (-m) for m in self.M)
        estimate = alpha * self.m * self.m / Z
        # Small range correction
        if estimate <= 2.5 * self.m:
            zeros = self.M.count(0)
            if zeros > 0:
                estimate = self.m * math.log(self.m / zeros)
        return int(estimate)

# Usage
hll = HyperLogLog(p=14)  # 12KB of memory

# Process 1 billion events
for event in event_stream:
    hll.add(event.user_id)

print(f"Unique users: ~{hll.count():,}")  # e.g., "Unique users: ~987,234,122"
# True count: 1,000,000,000 → error ≈ 1.3% — impressive for 12KB!

# Redis has native HyperLogLog:
# PFADD my-hll user:alice user:bob user:carol
# PFCOUNT my-hll → returns estimated unique count
# PFMERGE combined my-hll-1 my-hll-2 → merge two HLLs
```

**Used by:** Google Analytics (unique visitors), Redis (`PFADD`/`PFCOUNT`), Apache Flink streaming analytics, Twitter for trending topics, Cloudflare edge analytics.

---

## 🔝 Top-K / Heavy Hitters Problem

### The Problem

**Question:** "What are the 10 most trending hashtags in the last 5 minutes?"

At Twitter scale: 6,000 tweets/second, each with potentially unique hashtags. How do you find the top 10 efficiently?

### Approach 1: HashMap + MinHeap (Small Scale)

```python
import heapq
from collections import defaultdict

def top_k(stream, k: int) -> list:
    counts = defaultdict(int)
    for item in stream:
        counts[item] += 1
    
    # Use MinHeap of size k to find top k
    heap = []
    for item, count in counts.items():
        heapq.heappush(heap, (count, item))
        if len(heap) > k:
            heapq.heappop(heap)  # Remove smallest
    
    return sorted(heap, reverse=True)

# Problem: counts dict grows without bound at Twitter scale
# 1M unique hashtags × 16 bytes = 16MB — manageable locally
# But across distributed servers: need to merge
```

### Approach 2: Count-Min Sketch + MinHeap (Production)

Combine Count-Min Sketch (frequency) with a MinHeap (top-K tracking):

```python
import heapq

class TopKHeavyHitters:
    def __init__(self, k: int, width: int = 10000, depth: int = 5):
        self.k = k
        self.cms = CountMinSketch(width=width, depth=depth)
        self.top_k_heap = []    # Min-heap of (count, item)
        self.top_k_set = set()  # Set of items currently in heap

    def add(self, item: str):
        self.cms.add(item)
        estimated_count = self.cms.estimate(item)

        if item in self.top_k_set:
            # Update existing entry (rebuild heap periodically)
            pass
        elif len(self.top_k_heap) < self.k:
            heapq.heappush(self.top_k_heap, (estimated_count, item))
            self.top_k_set.add(item)
        elif estimated_count > self.top_k_heap[0][0]:
            # Evict the smallest
            _, evicted = heapq.heapreplace(self.top_k_heap, (estimated_count, item))
            self.top_k_set.discard(evicted)
            self.top_k_set.add(item)

    def get_top_k(self) -> list:
        return sorted(self.top_k_heap, reverse=True)

# Usage
tracker = TopKHeavyHitters(k=10)
for tweet in tweet_stream:
    for hashtag in tweet.hashtags:
        tracker.add(hashtag)

print(tracker.get_top_k())
# → [('#WorldCup', 982341), ('#Python', 451233), ('#AI', 398721), ...]
```

### Distributed Top-K (Parallel Aggregation)

```
Twitter architecture:
  10 stream processors (each handles 600 tweets/sec)
  Each maintains its own local Top-K tracker
  
  Every 30 seconds:
    Each processor sends its local Top-K to an aggregator
    Aggregator merges (sum counts across processors, re-ranks)
    Global Top-K result published to Redis → served to clients

Aggregation is cheap: merge 10 lists of 100 items each → 1000 items total → re-rank
```

---

## 🌳 Trie (Prefix Search & Autocomplete)

### The Problem

**Question:** "A user types 'app' — show them autocomplete suggestions: 'apple', 'application', 'app store'..."

Tries are tree data structures optimized for prefix queries.

### How It Works

```
Insert: "app", "apple", "application", "apt", "bat", "ball"

Trie structure:
root
├── a
│   ├── p
│   │   ├── p [END: "app"]
│   │   │   ├── l
│   │   │   │   ├── e [END: "apple"]
│   │   │   │   └── i
│   │   │   │       └── c
│   │   │   │           └── a
│   │   │   │               └── t
│   │   │   │                   └── i
│   │   │   │                       └── o
│   │   │   │                           └── n [END: "application"]
│   └── t [END: "apt"]
└── b
    ├── a
    │   ├── t [END: "bat"]
    │   └── l
    │       └── l [END: "ball"]
```

```python
class TrieNode:
    def __init__(self):
        self.children: dict[str, 'TrieNode'] = {}
        self.is_end: bool = False
        self.count: int = 0        # How many times this word was searched
        self.top_words: list = []  # Cached top 3 suggestions at this prefix

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str, frequency: int = 1):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True
        node.count += frequency

    def search_prefix(self, prefix: str) -> TrieNode | None:
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node

    def autocomplete(self, prefix: str, top_k: int = 5) -> list[tuple[str, int]]:
        """Return top-k suggestions for the given prefix."""
        prefix_node = self.search_prefix(prefix)
        if not prefix_node:
            return []
        
        results = []
        self._dfs(prefix_node, prefix, results)
        # Sort by frequency, return top k
        results.sort(key=lambda x: -x[1])
        return results[:top_k]

    def _dfs(self, node: TrieNode, current: str, results: list):
        if node.is_end:
            results.append((current, node.count))
        for char, child in node.children.items():
            self._dfs(child, current + char, results)

# Usage
trie = Trie()
trie.insert("apple", frequency=9823)
trie.insert("application", frequency=4521)
trie.insert("app", frequency=15000)
trie.insert("apt", frequency=1200)

trie.autocomplete("app")
# → [("app", 15000), ("apple", 9823), ("application", 4521)]

trie.autocomplete("ap")
# → [("app", 15000), ("apple", 9823), ("application", 4521), ("apt", 1200)]
```

### Distributed Trie (Google Search Scale)

For billions of search queries:

```
1. Offline job (weekly): mine top 1M search queries from logs
2. Build Trie in memory with frequencies
3. At each Trie node: pre-cache top 5 suggestions (DFS too slow for real-time)
4. Serialize Trie → store in distributed cache (Redis or CDN)
5. Serve autocomplete from CDN edge (< 5ms latency)

Sharding: Split trie by first character
  Shard 'a-g' → Server 1
  Shard 'h-n' → Server 2
  Shard 'o-z' → Server 3
  Special characters → Server 4
```

**Used by:** Google search, Elasticsearch prefix queries, database LIKE 'prefix%' queries.

---

## 🗺️ Geohash & QuadTree (Proximity Search)

### The Problem

**Question:** "Find all restaurants within 5km of latitude 37.77, longitude -122.42"

A naive approach (`WHERE distance(lat, lng, user_lat, user_lng) < 5000`) scans all rows and computes distance for each. Unusable at scale.

### Approach 1: Geohash

Encode a lat/lng pair as a short alphanumeric string. Nearby locations share common prefixes.

```
Encoding:
  World split into 32 cells by first character (Z-curve interleaving of lat and lng bits)
  Each subsequent character subdivides further (32 sub-cells each)
  
  San Francisco (37.7749, -122.4194):
    Precision 1: "9"          (covers 5000km × 5000km)
    Precision 3: "9q8"        (covers 156km × 156km)
    Precision 5: "9q8yy"      (covers 4.9km × 4.9km)
    Precision 7: "9q8yy9m"    (covers 152m × 152m)
    Precision 9: "9q8yy9mgu"  (covers 4.8m × 4.8m)

1 block north (37.776, -122.4194):
    Precision 7: "9q8yy9t"   (same prefix "9q8yy" to 5 chars!)

Los Angeles (34.0522, -118.2437):
    Precision 5: "9muu0"      (different prefix — far away)
```

**Query: Find restaurants within ~5km of user**

```python
import geohash  # pip install python-geohash

def find_nearby_restaurants(user_lat: float, user_lng: float, radius_km: float) -> list:
    # Get geohash of user's location at appropriate precision
    precision = 5  # ~4.9km × 4.9km cells
    user_hash = geohash.encode(user_lat, user_lng, precision=precision)
    
    # Get 8 neighboring cells + the cell itself (to handle borders)
    neighbors = geohash.neighbors(user_hash) + [user_hash]
    
    # Query: find restaurants in any of the 9 cells
    # Fast B-tree index scan on 9 short string prefixes
    restaurants = db.query("""
        SELECT * FROM restaurants 
        WHERE geohash_5 = ANY(%s)
    """, [neighbors])
    
    # Filter to exact radius (geohash cells are squares, radius is a circle)
    return [r for r in restaurants 
            if haversine_distance(user_lat, user_lng, r.lat, r.lng) <= radius_km]

def haversine_distance(lat1, lng1, lat2, lng2) -> float:
    """Great-circle distance in km"""
    R = 6371
    dlat = math.radians(lat2 - lat1)
    dlng = math.radians(lng2 - lng1)
    a = math.sin(dlat/2)**2 + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(dlng/2)**2
    return R * 2 * math.asin(math.sqrt(a))
```

### Approach 2: QuadTree (Dynamic — for Moving Objects)

Geohash uses fixed cell sizes. QuadTree adaptively subdivides based on data density — ideal when objects are moving (drivers, users).

```
QuadTree:
  Split the world into 4 quadrants
  Each quadrant subdivides into 4 more when it has > MAX_CAPACITY items
  
  New York (dense): subdivides deeply
  Montana (sparse): stays as large cells
  
                 Root (World)
                /    |    |    \
               NW   NE   SW    SE
                     |
               [dense area]
               /  |  |  \
             NW  NE  SW  SE
             ...subdivides further...
```

**Used by:** Uber (H3 hexagonal grid, conceptually similar), Yelp (geohash), DoorDash, Lyft, Airbnb for map search.

---

## 🗑️ LRU & LFU Cache Eviction

### LRU (Least Recently Used)

Evict the item that was accessed least recently.

```
Capacity: 3 items

Access sequence: A, B, C, D, A, E

After A: [A]
After B: [A, B]
After C: [A, B, C]
Access D: Cache full! Evict LRU (A was first, but B and C are older... A was first)
  Current order (LRU first): A, B, C
  Evict A → cache: [B, C, D]
Access A: Cache miss! Evict LRU (B): cache: [C, D, A]
Access E: Cache miss! Evict LRU (C): cache: [D, A, E]
```

**O(1) implementation using HashMap + Doubly Linked List:**

```python
class Node:
    def __init__(self, key=None, val=None):
        self.key, self.val = key, val
        self.prev = self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.cache: dict[int, Node] = {}  # key → node
        # Sentinel nodes: head (MRU end) ←→ ... ←→ tail (LRU end)
        self.head, self.tail = Node(), Node()
        self.head.next, self.tail.prev = self.tail, self.head

    def _remove(self, node: Node):
        node.prev.next, node.next.prev = node.next, node.prev

    def _insert_at_head(self, node: Node):
        """Insert as most recently used (after head sentinel)"""
        node.next, node.prev = self.head.next, self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)           # Remove from current position
        self._insert_at_head(node)   # Move to MRU end
        return node.val

    def put(self, key: int, val: int):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, val)
        self.cache[key] = node
        self._insert_at_head(node)
        if len(self.cache) > self.cap:
            # Evict LRU (node before tail sentinel)
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

### LFU (Least Frequently Used)

Evict the item accessed the fewest times. On tie: evict least recently used among ties.

**Use case:** Long-term cache where popular items should never be evicted regardless of recency (e.g., trending songs on Spotify that are always played, not just recently played).

```python
from collections import defaultdict

class LFUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.key_count: dict[int, int] = {}          # key → frequency
        self.key_val: dict[int, int] = {}             # key → value
        self.count_keys: dict[int, dict] = defaultdict(dict)  # freq → {key: insertion_order}
        self.min_count = 0

    def _update_frequency(self, key: int):
        freq = self.key_count[key]
        del self.count_keys[freq][key]
        if self.min_count == freq and not self.count_keys[freq]:
            self.min_count += 1
        freq += 1
        self.key_count[key] = freq
        self.count_keys[freq][key] = None  # Use dict to maintain insertion order

    def get(self, key: int) -> int:
        if key not in self.key_val:
            return -1
        self._update_frequency(key)
        return self.key_val[key]

    def put(self, key: int, val: int):
        if self.cap <= 0:
            return
        if key in self.key_val:
            self.key_val[key] = val
            self._update_frequency(key)
            return
        if len(self.key_val) >= self.cap:
            # Evict LFU (and among LFU, the oldest = first key in count_keys[min_count])
            evict_key = next(iter(self.count_keys[self.min_count]))
            del self.count_keys[self.min_count][evict_key]
            del self.key_val[evict_key]
            del self.key_count[evict_key]
        self.key_val[key] = val
        self.key_count[key] = 1
        self.count_keys[1][key] = None
        self.min_count = 1
```

### LRU vs LFU vs FIFO vs TTL

| Policy | Evicts | Best For | Worst For |
|---|---|---|---|
| LRU | Least recently accessed | General purpose; works well for temporal locality | Items accessed once but recently (kept over frequently used old items) |
| LFU | Least frequently accessed | Stable working sets (Spotify songs, product catalog) | New items (start with freq=1, get evicted before becoming popular) |
| FIFO | Oldest inserted item | Simplest; queues | Frequently accessed old items get evicted |
| TTL | Expired by time | Session caches, rate limit counters | Highly popular items expire unnecessarily |
| ARC | Adaptive (blend LRU + LFU) | Best overall performance | Implementation complexity |

---

## 👑 Leader Election Algorithms

### Why Leader Election?

Distributed systems often need one authoritative node to:
- Coordinate actions (exactly one node schedules jobs)
- Hold a lock (only one node writes to a shared resource)
- Be the primary writer (database primary)

### Algorithm 1: Bully Algorithm

The node with the highest ID "bullies" others into accepting it as leader.

```
Nodes: [P1, P2, P3, P4, P5] (P5 is current leader, crashes)

P3 detects P5 is dead:
  P3 sends "Election" to P4, P5
  P4 responds "OK" to P3 (P4 has higher ID → P3 gives up)
  P4 sends "Election" to P5
  No response from P5
  P4 declares itself leader → sends "Coordinator" to all lower nodes

Result: P4 is new leader.
```

**Problem:** High message complexity O(N²). Used in small clusters.

### Algorithm 2: Raft Leader Election (Production Standard)

Covered in [10_Distributed_Systems_Fundamentals.md] in depth. Key points:

```
All nodes start as Followers.
Randomized election timeout (150-300ms): if no heartbeat → become Candidate.
Candidate requests votes from all nodes.
First candidate to get majority becomes Leader.
Leader sends periodic heartbeats.
On leader failure: another node times out → new election.

RTO: typically 300-600ms (one election timeout + network RTT)
```

### Algorithm 3: ZooKeeper / etcd (Practical Leader Election)

Use a distributed coordination service rather than implementing from scratch.

```python
import etcd3

def run_with_leader_election(node_id: str, task):
    client = etcd3.client()
    
    # Try to acquire a lease-based lock
    lease = client.lease(ttl=10)  # 10-second TTL
    
    # Attempt to set the leader key (only succeeds if key doesn't exist)
    success = client.put_if_not_exists(
        '/leader', 
        node_id.encode(), 
        lease=lease
    )
    
    if success:
        print(f"{node_id}: I am the leader!")
        
        # Keep lease alive while running
        def keep_alive():
            for _ in client.refresh_lease(lease):
                pass
        
        import threading
        threading.Thread(target=keep_alive, daemon=True).start()
        
        task()  # Run the leader's work
    else:
        # Watch for the leader key to be deleted (leader crashes)
        current_leader = client.get('/leader')[0].decode()
        print(f"{node_id}: Following {current_leader}")
        
        # Watch for changes → re-attempt election when leader dies
        events_iterator, cancel = client.watch('/leader')
        for event in events_iterator:
            if isinstance(event, etcd3.events.DeleteEvent):
                cancel()
                run_with_leader_election(node_id, task)  # Retry
                break
```

---

## 🌿 Checksums & Merkle Trees

### Checksums

A hash of data used to detect corruption or changes.

```python
import hashlib

def checksum(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()

# S3 uses MD5 checksums for integrity verification
# Git uses SHA-1 for commit hashes
# TCP/UDP use CRC-32 for packet integrity
```

### Merkle Trees

A binary tree where each leaf is a hash of a data block, and each internal node is a hash of its children.

```
Data blocks: [B0, B1, B2, B3]

Leaf hashes:
  H0 = hash(B0)    H1 = hash(B1)    H2 = hash(B2)    H3 = hash(B3)
  
Internal nodes:
  H01 = hash(H0 + H1)              H23 = hash(H2 + H3)
  
Root:
  Root = hash(H01 + H23)
```

**Use case: Efficiently find which data blocks differ between two replicas.**

```
Replica 1 root: "abc123"
Replica 2 root: "xyz789"

They differ! Which blocks?
  Compare H01: match? No, differ.
    Compare H0: match? Yes.
    Compare H1: match? No.
    → Block B1 differs! (Only 1 block out of 4 identified in 3 comparisons)
    
With 1 million blocks: O(log N) comparisons instead of O(N)
```

**Used by:** Git (commit DAG), Cassandra (anti-entropy repair), DynamoDB (replication sync), Bitcoin (block integrity).

---

## 📋 When to Use Which Algorithm

| Problem | Algorithm | Time | Space |
|---|---|---|---|
| Rate limiting per user | Token Bucket / Sliding Window Counter | O(1) | O(1) |
| Smooth outbound rate | Leaky Bucket | O(1) | O(capacity) |
| Distributed cache routing | Consistent Hashing | O(log N) | O(N × vnodes) |
| "Is this URL already crawled?" | Bloom Filter | O(k) | O(m bits) |
| "How many times clicked?" | Count-Min Sketch | O(k) | O(width × depth) |
| "How many unique visitors?" | HyperLogLog | O(1) | O(m) ≈ 12KB |
| "Top 10 trending hashtags" | Count-Min Sketch + MinHeap | O(log k) | O(cms + k) |
| Autocomplete / prefix search | Trie | O(L) search | O(N × L) |
| "Restaurants within 5km" | Geohash + B-tree index | O(log N) | O(N) |
| LRU cache | HashMap + DLL | O(1) get/put | O(capacity) |
| LFU cache | HashMap + HashMap + linked sets | O(1) get/put | O(capacity) |
| Leader election (small cluster) | Bully | O(N²) | O(N) |
| Leader election (production) | Raft / etcd | O(log N) | O(N) |
| Detect data corruption | Checksum (SHA-256) | O(data_size) | O(hash_size) |
| Replica sync (which blocks differ?) | Merkle Tree | O(log N) | O(N) |

---

## 🎯 Key Takeaways

| Algorithm | Core Insight |
|---|---|
| **Sliding Window Counter** | Production rate limiting: O(1) memory, ~99.997% accurate, no window-boundary attacks |
| **Token Bucket** | Allow controlled bursts (natural for API clients); smooth over time |
| **Consistent Hashing** | ~1/N keys move when adding/removing a node; use 150 vnodes for even distribution |
| **Bloom Filter** | "Definitely not in set" with ~1% FP; saves 85% memory vs HashSet |
| **Count-Min Sketch** | Approximate frequency in O(fixed) memory; 0 false negatives |
| **HyperLogLog** | Unique counts within 1% error using 12KB regardless of cardinality |
| **Top-K** | CMS + MinHeap; distributed: local tracking → periodic merge |
| **Trie** | O(prefix_length) prefix search; pre-cache suggestions at each node for autocomplete |
| **Geohash** | Encode lat/lng as string; proximity = string prefix match; 9 cells cover radius |
| **LRU** | HashMap + doubly linked list = O(1) get/put; evict by access recency |
| **Raft / etcd** | Consensus-based leader election; always use a battle-tested implementation |
| **Merkle Tree** | O(log N) to find differing data blocks between replicas |
