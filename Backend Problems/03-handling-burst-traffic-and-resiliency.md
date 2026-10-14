# Handling Burst Traffic & Distributed System Resiliency: High-Level & Low-Level System Design

## 1. Executive Summary & Real-World Production Disasters

In distributed systems, traffic rarely grows in a smooth, linear progression. While capacity planning often assumes predictable diurnal cycles, real-world platforms routinely experience **Burst Traffic**—instantaneous, step-function surges where inbound request rates spike by $10\times$, $100\times$, or even $1,000\times$ within milliseconds. 

```
                          BURST TRAFFIC VS. ORGANIC GROWTH
                          
   Requests / Sec (RPS)
        ▲
500k ───┼                                   ┌──────────────────────────┐
        │                                   │ FLASH SALE / PUSH NOTIF  │
        │                                   │ (Step-Function Surge)    │
        │                                   │ +500,000 RPS in 2 sec    │
        │                                   └─────────┬────────────────┘
        │                                             │
100k ───┼                                             █
        │                                             █
        │                                             █
 10k ───┼                 ┌───────────────────────────█─────────
        │                 │ Organic Diurnal Peak      █
  1k ───┼───/\───────────/ \──────────────────────────█─────────
        │  /  \         /   \                         █
   0 ───┴─┴────┴───────┴─────┴────────────────────────┴────────► Time
```

Standard horizontal auto-scaling (e.g., Kubernetes Horizontal Pod Autoscaler or AWS Auto Scaling Groups) requires **3 to 7 minutes** to observe metrics, trigger scaling policies, provision compute instances, pull container images, and warm up runtimes (such as the JVM JIT compiler or application connection pools). Under a sub-second burst, unmitigated traffic saturates operating system socket buffers, thread pools, and database connection pools before a single new container can spin up.

---

### Real-World Production Disasters & Post-Mortems

#### 1. Amazon Prime Day Outage (July 16, 2018)
- **The Event:** Amazon Prime Day kicked off at 3:00 PM EST. Millions of shoppers opened the Amazon homepage and mobile app simultaneously to claim lightning deals.
- **The Failure Mechanism:** The burst overwhelmed Amazon's internal distributed key-value store, **Sable** (an internal DynamoDB derivative used for session tracking, cart state, and homepage rendering). Sable experienced thread pool saturation and extreme read latency. Because the frontend services called Sable synchronously without aggressive client timeouts or adequate circuit breaking, worker threads in the frontend fleets were pinned waiting for responses. The entire frontend cluster entered a death spiral, presenting shoppers with images of dogs ("Dogs of Amazon" 500 error page).
- **The Remediation:** Amazon engineers temporarily disabled complex personalized recommendation widgets, shed non-critical homepage features, and forced traffic shedding at the CloudFront edge CDN. The outage lasted over an hour and cost an estimated $100 million in lost sales.

#### 2. AWS US-East-1 Kinesis & CloudWatch Cascading Collapse (November 25, 2020)
- **The Event:** Amazon Kinesis provisioned a modest capacity addition to its front-end fleet in `us-east-1`.
- **The Failure Mechanism:** When the new capacity was added, every existing node in the Kinesis front-end fleet established internal tracking sockets with every other node. This triggered the operating system's maximum file descriptor and thread limits (`threads-max` in Linux). The front-end nodes began crashing. As nodes crashed, the remaining healthy nodes absorbed the entire burst traffic, immediately running out of threads and crashing as well.
- **Cascading Blast Radius:** Because Amazon Cognito, CloudWatch, and EventBridge internally relied on Kinesis for event streaming, these services failed simultaneously, taking down large portions of the internet (including Roku, Adobe, and Flickr) for over 17 hours.

#### 3. Ticketmaster & The Taylor Swift "Eras Tour" Meltdown (November 2022)
- **The Event:** Presale ticket registration opened for millions of fans.
- **The Failure Mechanism:** Over 3.5 billion system requests hit the platform. Millions of automated scalper bots bypassed the front-end waiting room and hit backend checkout endpoints directly. The extreme burst caused lock-wait queue saturation in the relational database layer. Transaction abort rates spiked to over 99%, and database CPUs locked at 100% processing rollback logs. Synchronous microservices experienced cascading thread exhaustion, requiring the entire public ticket sale to be canceled.

---

## 2. Theoretical Foundations: Anatomy of a Crash Under Burst Load

To build resilient architectures, we must analyze the physical and mathematical mechanisms that govern system collapse under load.

```
                          THE CASCADE DEATH SPIRAL
                          
  [500,000 RPS Burst]
          │
          ▼
  ┌────────────────────────────────────────────────────────┐
  │ 1. Operating System Socket Buffer Saturation           │
  │    Linux SYN Queue and Listen Backlog fill up.          │
  │    Packets silently dropped; TCP RSTs generated.       │
  └───────────────────────┬────────────────────────────────┘
                          ▼
  ┌────────────────────────────────────────────────────────┐
  │ 2. Worker Thread Pool Starvation                       │
  │    Tomcat/Netty/Go worker goroutines block on I/O.     │
  │    Context switching overhead spikes; latency explodes.│
  └───────────────────────┬────────────────────────────────┘
                          ▼
  ┌────────────────────────────────────────────────────────┐
  │ 3. Database Connection Pool Exhaustion                 │
  │    HikariCP/pgBouncer pools reach capacity (100/100).  │
  │    Upstream requests block waiting for DB connections.  │
  └───────────────────────┬────────────────────────────────┘
                          ▼
  ┌────────────────────────────────────────────────────────┐
  │ 4. Garbage Collection (GC) Stop-The-World Freeze       │
  │    Massive in-flight allocations fill heap memory.     │
  │    Full GC triggers multi-second STW pauses.           │
  └───────────────────────┬────────────────────────────────┘
                          ▼
  ┌────────────────────────────────────────────────────────┐
  │ 5. Health Check Timeouts & Node Eviction               │
  │    Kubelet liveness probe fails due to STW/CPU freeze. │
  │    Kubernetes terminates node; traffic shifts to       │
  │    surviving nodes, accelerating the death spiral!     │
  └────────────────────────────────────────────────────────┘
```

### 2.1 The Mathematics of Queuing Collapse: Little's Law & Kingman's Formula

Every software service contains internal queues (TCP socket backlogs, thread pool task queues, database connection pools). The behavior of these queues under burst load is governed by two fundamental theorems of queuing theory.

#### Little's Law
$$L = \lambda W$$
Where:
- $L$ = Number of requests concurrently inside the system (concurrency / in-flight requests).
- $\lambda$ = Arrival rate of requests (requests per second / RPS).
- $W$ = Average time a request spends in the system (latency / duration).

If average processing latency $W = 100\text{ ms } (0.1\text{ s})$, and normal arrival rate $\lambda = 2,000\text{ RPS}$, the concurrent in-flight requests are:
$$L = 2,000 \times 0.1 = 200\text{ in-flight requests}$$

When a burst occurs and $\lambda$ jumps to $50,000\text{ RPS}$, if downstream dependencies (databases, third-party APIs) slow down such that latency $W$ increases to $2\text{ seconds}$, the in-flight concurrency explodes:
$$L = 50,000 \times 2 = 100,000\text{ concurrent in-flight requests}$$

If the application server's thread pool is provisioned for 500 threads, the remaining 99,500 requests must sit in memory buffers or be dropped.

#### Kingman's Formula for Queuing Delay
For a $G/G/1$ queue (general arrival and service time distributions), the expected wait time in the queue $W_q$ is:
$$W_q \approx \left(\frac{\rho}{1 - \rho}\right) \left(\frac{c_a^2 + c_s^2}{2}\right) \frac{1}{\mu}$$
Where:
- $\rho = \frac{\lambda}{\mu}$ is server utilization ($0 \le \rho < 1$).
- $\mu$ is service capacity.
- $c_a$ and $c_s$ are coefficients of variation for arrival and service times.

```
   Queue Wait Time (Wq)
        ▲
        │                                         /
        │                                        /  ASYMPTOTIC EXPLOSION
        │                                       /   AS UTILIZATION -> 1.0
        │                                      /
        │                                     /
        │                                    /
        │                                   │
        │                                  /
        │                             ____/
        │___________________________--
        └───────────────────────────────────┴───────► Server Utilization (ρ)
        0%                                 100%
```

> [!CRITICAL]
> Notice the factor $\frac{\rho}{1 - \rho}$. As system utilization $\rho$ approaches $1.0$ ($100\%$), queue wait time does not increase linearly; **it approaches infinity asymptotically**. A system running at $95\%$ utilization experiences exponentially worse queuing delay than one running at $70\%$. When burst traffic drives $\rho \ge 1.0$, the queue grows without bound until memory or socket buffers are completely exhausted.

---

### 2.2 Metastable Failures: The Self-Sustaining Crash

A **Metastable Failure** is an engineering state where a system remains broken or degraded **even after the initial trigger (the burst traffic) has completely disappeared**.

```
    Normal Stable State ────[ Burst Traffic Spike ]────► Metastable Failure State
           ▲                                                        │
           │                                                        │
           └────────────[ CANNOT RECOVER AUTOMATICALLY ]────────────┘
                        - Retry storms keep load high
                        - Cold cache misses bypass L1/L2
                        - Garbage Collection pauses persist
                        - Slow queries hold DB locks
```

Metastable failures are powered by vicious feedback loops:
1. **The Client Retry Storm:** When servers slow down, clients (browsers, mobile apps, upstream microservices) time out after 2 seconds and retry. If $50,000\text{ RPS}$ fail, and clients retry 3 times, the incoming load becomes $50,000 \times (1 + 3) = 200,000\text{ RPS}$. Even if new users stop arriving, the retries sustain the overload.
2. **Cache Eviction and Stampede:** Under memory pressure, LRU caches evict active keys. When the system attempts to service requests, every request triggers an expensive database query, preventing the cache from warming back up.
3. **Garbage Collection Death Loop:** High concurrency causes allocations in Java/Go heaps to surge into Old Generation. The JVM triggers a Full GC Stop-The-World (STW) pause. During the 4-second STW pause, no requests are served, but incoming requests accumulate in the TCP socket buffer. When the GC pause completes, the server is hit with thousands of queued requests simultaneously, immediately triggering another GC pause.

---

## 3. Comprehensive Deep Dive into Architectural Approaches

To prevent catastrophic collapse under burst traffic, distributed architectures deploy six core resiliency patterns across different layers of the infrastructure stack:

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               MULTI-LAYER DEFENSE ARCHITECTURE                                    │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

 [ Client Layer ]              Exponential Backoff with Full Jitter, Hedging & Retry Budgets
        │
        ▼
   [ DNS / CDN ]               Edge Caching, Stale-While-Revalidate, DDoS Protection
        │
        ▼
 [ Waiting Room ]              Virtual Waiting Room (Queue-IT / Redis Sorted Set Admission)
        │
        ▼
  [ API Gateway ]              Distributed Multi-Tier Rate Limiting (Token Bucket / Redis Lua)
        │
        ▼
 [ Ingress/Mesh ]              Adaptive Concurrency Limiting & Load Shedding (Little's Law)
        │
        ▼
 [ Service Tier ]              Circuit Breakers & Bulkhead Isolation (Thread / Semaphore Pools)
        │
        ▼
 [ Async Buffer ]              Message Queue Load Leveling (Kafka / RabbitMQ / SQS)
        │
        ▼
 [ Data Tier ]                 Throttled DB Workers, Read Replicas, In-Memory Sharded Caches
```

---

## Approach 1: Asynchronous Decoupling & Queue-Based Load Leveling (The Buffer Pattern)

### High-Level Design (HLD)

The most fundamental law of handling write-heavy burst traffic is **decoupling the fast ingestion interface from the slow persistence engine**. 

In a synchronous architecture, a client request directly translates to an HTTP thread holding open an expensive database connection until a write transaction commits. Under a 100,000 RPS burst, the database runs out of connections instantly.

Under Queue-Based Load Leveling, the ingress service performs minimal validation, appends the write payload to an ultra-fast streaming buffer (Kafka, Pulsar, SQS), and immediately returns `202 Accepted` to the client. Downstream workers pull from the queue at a controlled, constant rate that matches database write capacity.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                             QUEUE-BASED LOAD LEVELING TOPOLOGY                                   │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

                               INGESTION TIER                        PROCESSING TIER
  [ 100,000 RPS Burst ]
       │    │    │
       ▼    ▼    ▼
  ┌─────────────────────────┐
  │   API Gateway Fleet     │  - Stateless, non-blocking I/O
  │   (Go / Netty / Rust)   │  - Validates request payload & auth
  └────────────┬────────────┘
               │ 1. Produce message (async append)
               ▼
  ┌─────────────────────────┐
  │  Distributed Message    │  - Partitions: 64
  │  Broker (Kafka/Pulsar)  │  - Retention: Append-only disk log
  └────────────┬────────────┘  - Buffer absorbs 500,000 messages in seconds
               │
               │ 2. Controlled Rate Pull (Constant 2,000 msg/sec)
               ▼
  ┌─────────────────────────┐
  │ Worker Pool Cluster     │  - Number of workers calibrated to DB capacity
  │ (Throttled Consumers)   │  - Batch processing & idempotency checks
  └────────────┬────────────┘
               │
               │ 3. Batch Inserts (JDBC / COPY / Bulk Write)
               ▼
  ┌─────────────────────────┐
  │ Primary RDBMS / Store   │  - CPU utilization remains flat at 65%
  │ (PostgreSQL / Aurora)   │  - Zero connection pool starvation
  └─────────────────────────┘
```

#### Client Interaction Models for Asynchronous Processing
When an API returns `202 Accepted`, how does the client receive the final result?
1. **Polling:** The API returns a `Location: /api/v1/orders/ord_987123/status` header and a unique `job_id`. The client polls every 2 seconds.
2. **WebSocket / Server-Sent Events (SSE):** The client maintains a persistent lightweight SSE connection, and the worker publishes an event to Redis Pub/Sub upon completion, which pushes the update to the client.
3. **Webhooks:** For B2B integrations (Stripe, GitHub), the system fires an HTTP POST callback to the customer's webhook endpoint once processing finishes.

---

### Low-Level Design (LLD)

#### 1. Producer-Side Memory Buffering & Backpressure
If the message broker experiences transient network latency during a burst, API Gateway memory will exhaust if producers buffer messages without limits. Producers must implement bounded ring buffers with explicit backpressure policies.

```
   Incoming HTTP Request
            │
            ▼
    ┌───────────────┐
    │ Memory Queue  │ ──(Queue Full)──► Drop or Reject with 429/503
    │ (Bounded Size)│
    └───────┬───────┘
            │ Batch Drain (Every 10ms or 500 items)
            ▼
   Kafka Producer Thread
            │
            ▼
   Kafka Broker Socket
```

#### 2. Consumer-Side Rate-Governed Worker (Go Implementation)
Below is a production-grade worker demonstrating a rate-governed worker pool in Go using a Leaky Bucket rate limiter, batch processing, and graceful shutdown to prevent downstream database saturation.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"sync"
	"time"
)

type OrderEvent struct {
	OrderID   string  `json:"order_id"`
	UserID    string  `json:"user_id"`
	Amount    float64 `json:"amount"`
	Timestamp int64   `json:"timestamp"`
}

type ResilientConsumer struct {
	db          *sql.DB
	incomingMsg chan OrderEvent
	maxWorkers  int
	rateLimiter *time.Ticker // Rate regulator to cap DB operations
	batchSize   int
	batchFlush  time.Duration
}

func NewResilientConsumer(db *sql.DB, workers int, maxOpsPerSec int, batchSize int) *ResilientConsumer {
	interval := time.Second / time.Duration(maxOpsPerSec)
	return &ResilientConsumer{
		db:          db,
		incomingMsg: make(chan OrderEvent, 50000), // Bounded buffer
		maxWorkers:  workers,
		rateLimiter: time.NewTicker(interval),
		batchSize:   batchSize,
		batchFlush:  50 * time.Millisecond,
	}
}

// StartWorkerPool consumes from the message channel without exceeding downstream write capacity
func (rc *ResilientConsumer) Start(ctx context.Context, wg *sync.WaitGroup) {
	for i := 0; i < rc.maxWorkers; i++ {
		wg.Add(1)
		go rc.workerLoop(ctx, wg, i)
	}
}

func (rc *ResilientConsumer) workerLoop(ctx context.Context, wg *sync.WaitGroup, workerID int) {
	defer wg.Done()
	batch := make([]OrderEvent, 0, rc.batchSize)
	flushTimer := time.NewTicker(rc.batchFlush)
	defer flushTimer.Stop()

	for {
		select {
		case <-ctx.Done():
			if len(batch) > 0 {
				rc.persistBatch(batch)
			}
			log.Printf("[Worker-%d] Gracefully shutting down.", workerID)
			return

		case msg, ok := <-rc.incomingMsg:
			if !ok {
				return
			}
			batch = append(batch, msg)
			if len(batch) >= rc.batchSize {
				// Wait for rate-limiter token before executing DB write
				<-rc.rateLimiter.C
				rc.persistBatch(batch)
				batch = make([]OrderEvent, 0, rc.batchSize)
			}

		case <-flushTimer.C:
			if len(batch) > 0 {
				<-rc.rateLimiter.C
				rc.persistBatch(batch)
				batch = make([]OrderEvent, 0, rc.batchSize)
			}
		}
	}
}

// persistBatch executes a bulk SQL insert within a single round-trip
func (rc *ResilientConsumer) persistBatch(orders []OrderEvent) {
	if len(orders) == 0 {
		return
	}
	start := time.Now()
	
	// In production: Use parameterized batch query or PostgreSQL COPY protocol
	query := "INSERT INTO orders (order_id, user_id, amount, created_at) VALUES "
	vals := []interface{}{}
	for i, o := range orders {
		query += fmt.Sprintf("($%d, $%d, $%d, $%d),", i*4+1, i*4+2, i*4+3, i*4+4)
		vals = append(vals, o.OrderID, o.UserID, o.Amount, time.Unix(o.Timestamp, 0))
	}
	query = query[:len(query)-1] // Trim trailing comma
	query += " ON CONFLICT (order_id) DO NOTHING" // Idempotency guarantee

	_, err := rc.db.Exec(query, vals...)
	if err != nil {
		log.Printf("Batch write error: %v. Routing to Dead Letter Queue (DLQ)", err)
		rc.routeToDLQ(orders)
		return
	}
	log.Printf("Persisted %d orders in %v. DB connection safe.", len(orders), time.Since(start))
}

func (rc *ResilientConsumer) routeToDLQ(orders []OrderEvent) {
	// Write failed messages to a durable dead-letter topic for manual replay
}
```

#### 3. Tradeoffs & Production Failure Modes
* **Pros:** Guaranteed protection of downstream relational databases and legacy external payment gateways. Flattens extreme spikes into a smooth, manageable workload curve.
* **Cons & Risks:**
  * **Eventual Consistency Latency:** Users do not receive immediate confirmation. Requires asynchronous UI design and polling/socket infrastructure.
  * **Backlog Lag Risk:** If incoming traffic rate ($\lambda$) permanently exceeds total drain capacity ($\mu$), queue depth grows indefinitely until disk space is exhausted.
  * **Idempotency Mandate:** Network retries between brokers and workers cause duplicate message deliveries. Consumers **must** be strictly idempotent using database unique constraints (`ON CONFLICT DO NOTHING`) or distributed deduplication keys.

---

## Approach 2: Distributed Multi-Tier Rate Limiting & Throttling

### High-Level Design (HLD)

Rate limiting controls the rate of incoming requests by establishing maximum thresholds for clients, IP addresses, API keys, or specific endpoints. When applied correctly, rate limiting protects backend infrastructure from noisy neighbors, rogue automated crawlers, misconfigured clients, and DDoS attacks.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               MULTI-TIER RATE LIMITING TOPOLOGY                                   │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Incoming Request Stream ]
             │
             ▼
  ┌─────────────────────────────────────┐  TIER 1: EDGE CDN / WAF (Cloudflare / AWS WAF)
  │ Edge Rate Limiter (Anycast Network) │  - Scope: Global IP reputation, ASN, Geolocation
  │ - Threshold: 10,000 req / min / IP  │  - Action: TCP Drop or HTTP 429 at Edge
  └──────────────────┬──────────────────┘
                     │ (Allowed Traffic)
                     ▼
  ┌─────────────────────────────────────┐  TIER 2: API GATEWAY (Envoy / Kong / NGINX)
  │ Gateway Rate Limiter                │  - Scope: User ID, Organization ID, API Key Tier
  │ - Threshold: 100 req / sec / User   │  - Backed by: Centralized Redis Cluster
  └──────────────────┬──────────────────┘
                     │ (Allowed Traffic)
                     ▼
  ┌─────────────────────────────────────┐  TIER 3: SERVICE INGRESS (Service Mesh / Sidecar)
  │ Internal Service Rate Limiter       │  - Scope: Endpoint-specific limits (e.g. /checkout)
  │ - Threshold: 50 concurrent writes   │  - Action: Fail fast before business logic
  └──────────────────┬──────────────────┘
                     │
                     ▼
          [ Core Microservices ]
```

#### Multi-Tier Defense Strategy
1. **Tier 1 (Edge / CDN):** Filters high-volume Layer 7 volumetric attacks before traffic consumes origin transit bandwidth.
2. **Tier 2 (API Gateway):** Enforces commercial tiering and tenant quotas (e.g., Free Tier = 10 RPS, Enterprise Tier = 1,000 RPS).
3. **Tier 3 (Service Mesh / Internal):** Protects sensitive internal microservices from being overwhelmed by other internal microservices during a fan-out cascade.

---

### Low-Level Design (LLD)

#### Comparison of Core Rate Limiting Algorithms

| Algorithm | How it Works | Allows Bursts? | Memory Complexity | Concurrency / Race Hazard |
| :--- | :--- | :---: | :---: | :--- |
| **Fixed Window Counter** | Increments a counter for a fixed time block (e.g., 12:00:00 - 12:01:00). | Yes ($2\times$ limit at window boundary) | $O(1)$ (1 integer per key) | High boundary burst vulnerability. |
| **Sliding Window Log** | Stores exact timestamp of every request in a sorted set (`ZSET`). | No (Strict enforcement) | $O(N)$ ($N$ = request count) | Heavy memory footprint under high traffic. |
| **Sliding Window Counter** | Blends counter of previous window with current window based on time overlap. | Smooths traffic | $O(1)$ (2 integers per key) | Minor approximation error ($\approx 0.05\%$). |
| **Token Bucket** | Tokens refill at constant rate $r$ up to capacity $B$. Request consumes 1 token. | Yes (Up to capacity $B$) | $O(1)$ (Tokens + Last Refill Time) | Requires atomic Read-Modify-Write in Redis. |
| **Leaky Bucket (GCRA)**| Requests enter a queue and leak at a constant fixed rate. | No (Perfect smoothing) | $O(1)$ (Theoretical Arrival Time) | Inflexible for bursty human workflows. |

---

#### The Sliding Window Counter Mathematical Model
To combine the $O(1)$ memory efficiency of Fixed Windows with the strict accuracy of Sliding Window Logs, production gateways use the **Sliding Window Counter** algorithm.

```
       PREVIOUS WINDOW (60s)                 CURRENT WINDOW (60s)
  [ Count = 100 requests ]             [ Count = 30 requests ]
 ├────────────────────────────────────┼───────────────┬────────────────────┤
 0s                                  60s             84s                  120s
                                                      ▲
                                                      │ Current Time: 84s
                                                      │ (40% into Current Window)
```

Suppose the limit is **100 requests per minute**:
- Current time is **84 seconds** into the cycle ($40\%$ through the current window).
- Overlap with the previous window is $100\% - 40\% = 60\%$.
- The estimated request count over the rolling 60-second window is calculated as:
$$\text{Estimated Count} = (\text{Requests in Previous Window} \times (1 - \text{Percentage Elapsed})) + \text{Requests in Current Window}$$
$$\text{Estimated Count} = (100 \times (1 - 0.40)) + 30 = 60 + 30 = 90\text{ requests}$$

Since $90 < 100$, the request is permitted and the current window counter increments to 31.

---

#### Production-Grade Redis Lua Script (Atomic Sliding Window Counter)
In distributed environments, checking and updating counters across multiple Redis keys creates race conditions. Executing logic inside a **Redis Lua script** guarantees single-threaded atomic execution with zero network round-trip race hazards.

```lua
-- KEYS[1]: Rate limit key prefix (e.g. "ratelimit:user_123")
-- ARGV[1]: Current UNIX timestamp in seconds
-- ARGV[2]: Window size in seconds (e.g. 60)
-- ARGV[3]: Maximum permitted requests per window (e.g. 100)

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

-- Determine window keys
local current_window_bucket = math.floor(now / window)
local prev_window_bucket = current_window_bucket - 1

local cur_key = key .. ":" .. current_window_bucket
local prev_key = key .. ":" .. prev_window_bucket

-- Fetch current and previous window counts
local current_count = tonumber(redis.call("GET", cur_key) or "0")
local prev_count = tonumber(redis.call("GET", prev_key) or "0")

-- Calculate rolling window overlap
local time_into_current_window = now % window
local weight = (window - time_into_current_window) / window
local estimated_count = math.floor(prev_count * weight + current_count)

if estimated_count >= limit then
    -- Rate limit exceeded!
    local retry_after = window - time_into_current_window
    return {0, estimated_count, limit, retry_after}
end

-- Allow request: Increment current window and set TTL
local new_count = redis.call("INCR", cur_key)
if new_count == 1 then
    -- Expire key after 2 full windows to clean up memory
    redis.call("EXPIRE", cur_key, window * 2)
end

local remaining = limit - (estimated_count + 1)
return {1, estimated_count + 1, limit, 0}
```

#### Gateway Response Headers Specification
When rate limiting triggers, the gateway must return `HTTP 429 Too Many Requests` along with standard IETF rate-limiting headers:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 36
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1718000036

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please wait 36 seconds before retrying.",
  "quota": 100,
  "window_seconds": 60
}
```

#### Fail-Open vs. Fail-Closed Architectural Policy
What happens if the Redis rate-limiting cluster crashes or becomes unreachable during a network partition?
- **Fail-Open (Recommended for revenue paths like Checkout):** If Redis times out (e.g., after 15ms), the gateway logs an alert, increments a fallback metric, and **allows the request through**. Serving the user and risking backend load is preferred over a 100% outage.
- **Fail-Closed (Recommended for high-security endpoints like Login, OTP, Credit Card Processing):** If Redis fails, reject the request to prevent brute-force attacks and credential stuffing.

---

## Approach 3: Adaptive Concurrency Limiting & Load Shedding

### High-Level Design (HLD)

There is a critical architectural distinction between **Rate Limiting** and **Load Shedding**:
- **Rate Limiting** protects against specific abusive clients or ensures quota compliance based on identity.
- **Load Shedding** protects the internal system against total saturation based on internal health signals (concurrency, queue delay, CPU, memory), regardless of who is sending the requests.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                ADAPTIVE LOAD SHEDDING ARCHITECTURE                               │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

   All Inbound Requests (100,000 RPS)
                 │
                 ▼
      ┌─────────────────────┐
      │ Request Priority    │── Inspect HTTP Header / Path
      │ Classifier Filter   │
      └──────────┬──────────┘
                 │
      ┌──────────┴────────────────────────────────┐
      │                                           │
      ▼ (Tier 0: Critical)                        ▼ (Tier 2: Degradable)
  [ Checkout / Complete Payment ]             [ Product Reviews / Recommendations ]
      │                                           │
      ▼                                           ▼
  ┌─────────────────────────────────────────────────────────┐
  │         ADAPTIVE CONCURRENCY LIMITER (In-Flight)        │
  │ Current In-Flight: 480 | Dynamic Threshold: 500         │
  └──────────┬────────────────────────────────────┬─────────┘
             │                                    │
             │ Allowed                            │ Over Capacity (Shed!)
             ▼                                    ▼
    [ Service Execution ]                [ HTTP 503 Service Unavailable ]
    - Processes order                     - "System busy, please retry"
    - Latency: 45ms                       - Zero backend work performed
```

#### The Request Priority Matrix
When the system is forced to shed load, it drops low-priority requests at the earliest possible stage:
1. **Tier 0 (Mission Critical):** `/api/v1/checkout`, `/api/v1/payments/charge`. Must never be shed unless total server crash is imminent.
2. **Tier 1 (Core Business Operations):** `/api/v1/cart/add`, `/api/v1/auth/token`.
3. **Tier 2 (Non-Critical / Degradable):** `/api/v1/recommendations`, `/api/v1/reviews/list`, `/api/v1/search/autocomplete`. Shed immediately when queue delays exceed acceptable thresholds.

---

### Low-Level Design (LLD)

#### Why CPU Utilization is a Dangerous Metric for Load Shedding
Many teams write shedding policies like: `if (CPU > 85%) { dropRequests(); }`. This approach fails in production:
1. **CPU is a Lagging Metric:** Operating system kernel CPU counters are averaged over 1-second to 5-second intervals. By the time CPU reads $90\%$, thousands of requests have already queued up in socket buffers.
2. **Asymmetric Processing Costs:** A single heavy search query can consume $100\%$ CPU while safely running on 1 thread, whereas 10,000 requests waiting on database I/O consume $5\%$ CPU while completely exhausting system thread and socket pools.
3. **Misleading GC Activity:** During garbage collection, CPU spikes to $100\%$, causing the shedder to drop traffic even though the memory cleanup is nearly finished.

#### Little's Law & TCP Vegas-Style Adaptive Concurrency (Netflix Gradient Limiter)
Modern systems (e.g., Netflix Concurrency Limits library) monitor **Latency and Queuing Delay** rather than CPU. Based on TCP congestion control algorithms (such as TCP Vegas), the system continuously computes a dynamic concurrency limit $L_{\text{max}}$.

```
   Latency (RTT)
        ▲
        │                              Congestion Point (Queues filling up)
        │                                             /
        │                                            /
        │                                           /
        │                                          /
 Baseline RTT ────────────────────────────────────/
        │
        └─────────────────────────────────────────┴─────────► In-Flight Concurrency
                                              Optimal Limit
```

#### The Gradient2 Algorithm Formula
The system tracks two values:
- $R_{\text{min}}$: The minimum observed round-trip latency under zero load (baseline physical latency, e.g., 20ms).
- $R_{\text{avg}}$: The exponentially weighted moving average (EWMA) latency under current traffic.

The congestion gradient $G$ is calculated as:
$$G = \frac{R_{\text{min}}}{R_{\text{avg}}}$$

The new dynamic concurrency limit is updated continuously:
$$\text{Limit}_{t+1} = \text{Limit}_t \times G + \beta$$
Where $\beta$ is a headroom constant (e.g., $1 \le \beta \le 5$) that allows the system to probe for additional capacity when latency is healthy.
- When $R_{\text{avg}} \approx R_{\text{min}}$ ($G \approx 1$): The system is healthy; the limit expands by $\beta$.
- When $R_{\text{avg}} > R_{\text{min}}$ ($G < 1$): Queues are forming downstream; the limit immediately contracts multiplicatively, shedding excess in-flight requests.

---

#### Adaptive Concurrency Limiter Middleware (Go Implementation)
Here is a complete, thread-safe implementation of an Adaptive Concurrency Limiting middleware based on the Additive Increase / Multiplicative Decrease (AIMD) algorithm:

```go
package main

import (
	"log"
	"math"
	"net/http"
	"sync"
	"sync/atomic"
	"time"
)

type AdaptiveLimiter struct {
	inFlight       int64
	currentLimit   float64
	minRTT         time.Duration
	sampleCount    int
	mu             sync.Mutex
	alpha          float64 // Additive increase step
	beta           float64 // Multiplicative decrease factor
	rttTolerance   float64 // Maximum acceptable latency factor (e.g. 1.5x of minRTT)
}

func NewAdaptiveLimiter(initialLimit float64) *AdaptiveLimiter {
	return &AdaptiveLimiter{
		currentLimit: initialLimit,
		minRTT:       time.Hour, // Seed with large baseline
		alpha:        1.0,
		beta:         0.7, // Reduce limit by 30% on congestion
		rttTolerance: 1.5,
	}
}

func (al *AdaptiveLimiter) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		current := atomic.AddInt64(&al.inFlight, 1)
		defer atomic.AddInt64(&al.inFlight, -1)

		al.mu.Lock()
		maxAllowed := int64(math.Floor(al.currentLimit))
		al.mu.Unlock()

		// Shed load immediately if concurrent requests exceed calculated limit
		if current > maxAllowed {
			w.Header().Set("Retry-After", "1")
			http.Error(w, "503 Service Unavailable: Load Shedding Active", http.StatusServiceUnavailable)
			return
		}

		start := time.Now()
		next.ServeHTTP(w, r)
		duration := time.Since(start)

		al.updateLimit(duration)
	})
}

func (al *AdaptiveLimiter) updateLimit(rtt time.Duration) {
	al.mu.Lock()
	defer al.mu.Unlock()

	// Update baseline lowest latency
	if rtt < al.minRTT {
		al.minRTT = rtt
	}

	al.sampleCount++
	// Recalibrate limit every 20 completed requests
	if al.sampleCount >= 20 {
		al.sampleCount = 0
		threshold := time.Duration(float64(al.minRTT) * al.rttTolerance)

		if rtt > threshold {
			// Congestion detected: Multiplicative Decrease
			al.currentLimit = math.Max(10.0, al.currentLimit*al.beta)
			log.Printf("[LoadShedder] Congestion! High RTT: %v (Base: %v). Backing off limit to: %.1f", 
				rtt, al.minRTT, al.currentLimit)
		} else {
			// Healthy: Additive Increase
			al.currentLimit += al.alpha
		}
	}
}
```

---

## Approach 4: Circuit Breakers, Bulkheading & Fail-Fast Mechanics

### High-Level Design (HLD)

In a microservice architecture, failures propagate across service boundaries. If **Service A** calls **Service B**, and Service B experiences slow response times due to burst load, Service A's HTTP client threads block waiting for socket reads. In seconds, Service A runs out of worker threads, causing it to fail when called by upstream services. This is a **Cascading Failure**.

```
                           THE CASCADING TIMEOUT CASCADE
                           
  [Client Requests]
         │
         ▼
  ┌─────────────┐  Threads: 100/100 Blocked
  │  Service A  │  All worker threads waiting on B's 10-second timeout!
  └──────┬──────┘
         │ (HTTP GET /recommendations - HANGS)
         ▼
  ┌─────────────┐  CPU: 100% | Latency: 12 seconds
  │  Service B  │  Overwhelmed by burst traffic!
  └─────────────┘
```

The **Circuit Breaker** pattern terminates this domino effect. It wraps remote calls in a state machine that detects elevated error rates or timeouts. Once failures cross a predefined threshold, the breaker **trips open**, intercepting future calls and immediately returning a fallback response without touching the network.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 CIRCUIT BREAKER STATE MACHINE                                    │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

               ┌──────────────────────────────────────────────┐
               │                                              │
               ▼                                              │ Success Rate > 90%
       ┌───────────────┐        Failure Rate > 50%     ┌──────────────┐
       │               │ ────────────────────────────► │              │
       │    CLOSED     │                               │     OPEN     │
       │ (Normal Flow) │ ◄──────────────────────────── │ (Fast Fail)  │
       └───────┬───────┘        Manual Reset or Config └──────┬───────┘
               ▲                                              │
               │                                              │ Sleep Window Elapsed
               │                                              │ (e.g. 10 seconds)
               │                                              ▼
               │                                       ┌──────────────┐
               │              Success                  │  HALF-OPEN   │
               └────────────────────────────────────── │ (Trial Probe)│
                                                       └──────────────┘
                                                              │
                                                              │ Any Failure
                                                              └───────► (Trip back to OPEN)
```

---

### Low-Level Design (LLD)

#### 1. Bulkheading: Thread Pool vs. Semaphore Isolation
To prevent one failing downstream dependency from consuming all server resources, systems isolate resources using **Bulkheads** (named after the watertight compartments in ships).

```
   Incoming Requests
        │
   ┌────┴─────────────────────────────┐
   │                                  │
   ▼                                  ▼
┌───────────────────────────────┐  ┌───────────────────────────────┐
│ Bulkhead 1: Payment Service   │  │ Bulkhead 2: Recommendation    │
│ Dedicated Thread Pool: 20     │  │ Dedicated Thread Pool: 10     │
│ Bounded Queue: 50             │  │ Bounded Queue: 10             │
└───────────────────────────────┘  └───────────────────────────────┘
```

- **Thread Pool Isolation (Netflix Hystrix Style):** Each remote dependency runs on an independent, dedicated thread pool with its own task queue. If the recommendation service hangs, only its 10 threads block. Payment processing continues unhindered on its separate thread pool.
  - *Tradeoff:* Higher CPU context-switching overhead.
- **Semaphore Isolation (Resilience4j Style):** Requests execute on the parent container thread, but must acquire a permit from an atomic counting semaphore (`AtomicInteger`). If 20 calls to a dependency are currently in-flight, subsequent calls fail fast immediately.
  - *Tradeoff:* Extremely fast ($O(1)$ memory, zero context switching), but cannot enforce hard timeouts on hung threads.

---

#### 2. Sliding Window Metric Tracking: Circular Ring Buffer
Modern circuit breakers use a **Time-Based or Count-Based Sliding Window** backed by a circular array of buckets to evaluate failure rates without lock contention.

```
                      10-SECOND SLIDING WINDOW (10 BUCKETS)
                      
   Bucket 0     Bucket 1     Bucket 2     Bucket 3     Bucket 4               Bucket 9
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          ┌──────────┐
  │ S: 90    │ │ S: 95    │ │ S: 100   │ │ S: 20    │ │ S: 5     │  ......  │ S: 88    │
  │ F: 2     │ │ F: 1     │ │ F: 0     │ │ F: 45    │ │ F: 70    │          │ F: 4     │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘          └──────────┘
                                               ▲
                                               │ Current Second Bucket
                                               │ (Failure Rate: 70 / 75 = 93.3% -> TRIP!)
```

#### Production-Grade Circuit Breaker Implementation (Go)
Below is a thread-safe Circuit Breaker with ring-buffer statistics and fallback support:

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

type State int

const (
	StateClosed State = iota
	StateOpen
	StateHalfOpen
)

type CircuitBreaker struct {
	mu                sync.RWMutex
	state             State
	failureThreshold  float64       // e.g. 0.50 (50% failure rate)
	openTimeout       time.Duration // Time to remain open before half-open probe
	lastStateChange   time.Time
	successCount      int64
	failureCount      int64
	minRequests       int64 // Minimum sample size before calculating failure rate
}

func NewCircuitBreaker(threshold float64, openTimeout time.Duration, minRequests int64) *CircuitBreaker {
	return &CircuitBreaker{
		state:            StateClosed,
		failureThreshold: threshold,
		openTimeout:      openTimeout,
		minRequests:      minRequests,
		lastStateChange:  time.Now(),
	}
}

var ErrCircuitOpen = errors.New("circuit breaker is OPEN: fast-failing remote call")

// Execute wraps remote calls with circuit breaker state management
func (cb *CircuitBreaker) Execute(action func() (interface{}, error), fallback func(error) (interface{}, error)) (interface{}, error) {
	if !cb.allowRequest() {
		if fallback != nil {
			return fallback(ErrCircuitOpen)
		}
		return nil, ErrCircuitOpen
	}

	result, err := action()

	cb.recordResult(err == nil)
	if err != nil && fallback != nil {
		return fallback(err)
	}
	return result, err
}

func (cb *CircuitBreaker) allowRequest() bool {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	now := time.Now()
	switch cb.state {
	case StateClosed:
		return true

	case StateOpen:
		if now.Sub(cb.lastStateChange) > cb.openTimeout {
			logStateTransition("OPEN", "HALF-OPEN")
			cb.state = StateHalfOpen
			cb.lastStateChange = now
			cb.successCount = 0
			cb.failureCount = 0
			return true // Allow single trial probe request
		}
		return false

	case StateHalfOpen:
		// In Half-Open, allow limited canary requests
		return true

	default:
		return false
	}
}

func (cb *CircuitBreaker) recordResult(success bool) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	if success {
		if cb.state == StateHalfOpen {
			logStateTransition("HALF-OPEN", "CLOSED")
			cb.state = StateClosed
			cb.lastStateChange = time.Now()
			cb.resetCounts()
		} else {
			cb.successCount++
		}
	} else {
		cb.failureCount++
		if cb.state == StateHalfOpen {
			logStateTransition("HALF-OPEN", "OPEN")
			cb.state = StateOpen
			cb.lastStateChange = time.Now()
		} else if cb.state == StateClosed {
			total := cb.successCount + cb.failureCount
			if total >= cb.minRequests {
				rate := float64(cb.failureCount) / float64(total)
				if rate >= cb.failureThreshold {
					logStateTransition("CLOSED", "OPEN")
					cb.state = StateOpen
					cb.lastStateChange = time.Now()
				}
			}
		}
	}
}

func (cb *CircuitBreaker) resetCounts() {
	cb.successCount = 0
	cb.failureCount = 0
}

func logStateTransition(from, to string) {
	fmt.Printf("[CircuitBreaker] Transitioning from %s to %s\n", from, to)
}
```

---

## Approach 5: Client-Side Resiliency: Exponential Backoff, Jitter, Hedging & Retry Budgets

### High-Level Design (HLD)

When systems fail under burst load, the client's behavior dictates whether the system recovers or enters a permanent metastable death spiral. 

A naive client retries failed requests immediately in a tight loop. If 10,000 requests fail, 10,000 retries hit the recovering server 50 milliseconds later. This creates **Synchronized Pulsing (The Thundering Herd)**.

```
                          THE SYNCHRONIZED RETRY PULSE
                          
  Requests / Sec
        ▲
 20,000 ┼         █                   █                   █
        │         █ (Initial Fail)    █ (Retry 1 Pulse)   █ (Retry 2 Pulse)
        │         █                   █                   █
  1,000 ┼─────────█───────────────────█───────────────────█────────► Time
                  T0                  T0 + 1s             T0 + 2s
                  (Every client retries at the exact same 1.000s interval!)
```

To break synchronization, clients must introduce **Exponential Backoff with Full Jitter**. By randomizing retry intervals across a uniform distribution, the retry load is dispersed smoothly across the time axis, allowing backend servers to clear their queues.

---

### Low-Level Design (LLD)

#### 1. Mathematical Analysis of Jitter Algorithms
AWS Architecture research evaluated three retry algorithms under extreme contention:

```
  1. Exponential Backoff (No Jitter):
     Sleep = Min(MaxSleep, Base * 2^attempt)

  2. Equal Jitter:
     Half = Min(MaxSleep, Base * 2^attempt) / 2
     Sleep = Half + Random(0, Half)

  3. Full Jitter (Optimal for breaking synchronization):
     Sleep = Random(0, Min(MaxSleep, Base * 2^attempt))
```

```
                          LOAD DISTRIBUTION COMPARISON
                          
   No Jitter:       | | | | | | | | |        (Periodic severe spikes; server re-crashes)
   Equal Jitter:    - - - - - - - - -        (Moderate dispersion; maintains lower bound)
   Full Jitter:     . . : · . : · . ·        (Uniform distribution; maximum server recovery rate)
```

With **Full Jitter**, the expected wait time is half that of deterministic backoff, but peak concurrency on the server is reduced by over $90\%$.

---

#### 2. Retry Budgets (Preventing Amplification)
Exponential backoff alone does not prevent retry amplification if the overall failure rate is sustained. Production client libraries (such as Finagle and gRPC) enforce **Retry Budgets**.

- The client maintains a rolling 60-second window tracking successful and retried requests.
- **Rule:** Retries are only permitted if total retries account for less than **10%** of all historical requests:
$$\text{Retry Permitted} \iff \frac{\text{Retry Requests}}{\text{Total Requests}} \le 0.10$$
If the backend is down and the client has already used its 10% budget, future errors fail fast immediately without retrying, capping total load amplification at $1.1\times$.

---

#### 3. Request Hedging: Eliminating Tail Latency Under Burst
When downstream services experience burst load, average latency might remain acceptable (e.g., 50ms), but the 99th percentile ($p99$) latency spikes to 3,000ms due to thread queuing. 

Under **Request Hedging**, the client issues a request to Instance 1. If no response is received within the expected $p95$ latency (e.g., 80ms), the client **speculatively fires an identical twin request** to Instance 2 without canceling the first. Whichever instance responds first satisfies the request, and the slower call is aborted.

```
  Client                     Instance 1 (Stuck in GC)         Instance 2 (Healthy)
    │                                   │                               │
    ├───── 1. Primary Request ─────────►│                               │
    │      (Wait for p95 = 80ms...)     │                               │
    │                                   │                               │
    ├───── 2. Hedged Request (at 80ms) ─┼──────────────────────────────►│
    │                                   │                               │
    │◄──── 3. Response in 15ms ─────────┼───────────────────────────────┤
    │      (Cancels Instance 1)         │                               │
```

> [!TIP]
> To prevent doubling backend traffic, Request Hedging must only be triggered for the slowest $5\%$ of requests ($p95$), and hedged calls must be subject to the client's global 10% retry budget.

---

#### Production HTTP Client with Full Jitter & Retry Budget (Go)

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"net/http"
	"sync"
	"time"
)

type ResilientHttpClient struct {
	client       *http.Client
	baseInterval time.Duration
	maxInterval  time.Duration
	maxRetries   int
	mu           sync.Mutex
	totalCalls   int64
	retryCalls   int64
}

func NewResilientHttpClient() *ResilientHttpClient {
	return &ResilientHttpClient{
		client:       &http.Client{Timeout: 5 * time.Second},
		baseInterval: 50 * time.Millisecond,
		maxInterval:  3 * time.Second,
		maxRetries:   3,
	}
}

func (c *ResilientHttpClient) DoWithRetry(req *http.Request) (*http.Response, error) {
	var resp *http.Response
	var err error

	for attempt := 0; attempt <= c.maxRetries; attempt++ {
		c.trackCall(attempt > 0)

		// Check Retry Budget if this is a retry attempt
		if attempt > 0 && !c.hasRetryBudget() {
			return nil, fmt.Errorf("retry budget exhausted (retries > 10%% of total traffic)")
		}

		resp, err = c.client.Do(req)

		// Success condition (HTTP 2xx or 3xx)
		if err == nil && resp.StatusCode < 500 && resp.StatusCode != http.StatusTooManyRequests {
			return resp, nil
		}

		// Don't sleep after the final failed attempt
		if attempt == c.maxRetries {
			break
		}

		// Calculate Exponential Backoff with Full Jitter: Random(0, Min(Max, Base * 2^attempt))
		backoffLimit := float64(c.baseInterval) * float64(int(1)<<attempt)
		if backoffLimit > float64(c.maxInterval) {
			backoffLimit = float64(c.maxInterval)
		}
		sleepDuration := time.Duration(rand.Float64() * backoffLimit)

		time.Sleep(sleepDuration)
	}

	return resp, err
}

func (c *ResilientHttpClient) trackCall(isRetry bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.totalCalls++
	if isRetry {
		c.retryCalls++
	}
}

func (c *ResilientHttpClient) hasRetryBudget() bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.totalCalls < 20 {
		return true // Allow initial warmup sample
	}
	retryRatio := float64(c.retryCalls) / float64(c.totalCalls)
	return retryRatio <= 0.10 // Max 10% retries permitted
}
```

---

## Approach 6: Edge Defense, Pre-Warming & Virtual Waiting Rooms

### High-Level Design (HLD)

#### The Auto-Scaling Fallacy
A common architectural mistake is relying exclusively on Horizontal Pod Autoscaling (HPA) or AWS EC2 Auto Scaling Groups to handle flash sales and marketing pushes.

```
  TIME TO SCALE COMPARISON UNDER A 10-SECOND BURST
  
  0s: Traffic spikes from 2k -> 200k RPS
  ├─ 15s: Prometheus / CloudWatch collects metric sample
  ├─ 45s: HPA evaluates threshold condition
  ├─ 60s: Kubernetes API schedules new Pods
  ├─ 90s: Cloud provider provisions new EC2 Node / VM
  ├─ 150s: Container image pulled from registry
  ├─ 180s: Application boots, runs migrations, initializes DB connection pool
  └─ 240s: JVM JIT compiler warms up bytecode
  
  RESULT: Origin crashed 3 minutes ago!
```

To survive sudden bursts, platforms deploy two external architectural mechanisms: **Infrastructure Pre-Warming** and **Edge Virtual Waiting Rooms**.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                            VIRTUAL WAITING ROOM TOPOLOGY (QUEUE-IT STYLE)                        │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

 [ 1,000,000 Concurrent Users ]
               │
               ▼
   ┌───────────────────────┐
   │ Cloudflare / Edge CDN │ ──(Has valid HMAC Admission Cookie?)
   └───────────┬───────────┘
               │
       ┌───────┴────────────────────────┐
       │ NO                             │ YES (Admitted)
       ▼                                ▼
   ┌─────────────────────────┐    ┌─────────────────────────┐
   │ Virtual Waiting Room    │    │ Origin Cluster          │
   │ (Edge Static HTML/JS)   │    │ (Protected Backend)     │
   │ - Shows queue position  │    │ - Max 5,000 RPS         │
   │ - Polls Redis Token Svc │    │ - Flat response latency │
   └───────────┬─────────────┘    └─────────────────────────┘
               │
               │ Releases 100 users / sec
               ▼
   [ Issue HMAC Cookie & Redirect to Origin ]
```

---

### Low-Level Design (LLD)

#### 1. The Virtual Waiting Room Cryptographic Handshake
1. The user requests `https://store.example.com/checkout`.
2. The Edge Worker (Cloudflare Worker / AWS CloudFront Function) inspects the request for a signed cookie: `QueueAdmissionToken`.
3. If the cookie is absent or invalid, the Edge Worker returns an HTTP 302 redirecting the user to a static Waiting Room hosted on an external CDN (e.g., S3/CloudFront) with zero origin compute dependency.
4. The Waiting Room page establishes a WebSocket or polling connection to a lightweight Redis cluster.
5. The admission engine uses a **Redis Sorted Set (`ZSET`)**:
   ```redis
   ZADD waiting_room:flash_sale 1718000000 "usr_abc123"
   ZRANK waiting_room:flash_sale "usr_abc123"
   ```
   The user's position in line is their `ZRANK`.
6. When the admission engine decides to let in 500 users, it pops users from the top of the set and generates a cryptographically signed HMAC SHA-256 token:
   $$\text{Signature} = \text{HMAC-SHA256}(\text{SecretKey}, \text{UserID} + \text{ExpirationTimestamp})$$
7. The browser receives the token in a cookie and redirects back to `store.example.com/checkout`. The Edge Worker validates the HMAC signature using WebCrypto in sub-milliseconds and forwards the request to the origin.

#### 2. Infrastructure Pre-Warming Playbook
When a burst event is scheduled in advance (e.g., ticket drops, product launches, national TV broadcasts), automated systems execute a pre-warming sequence **15 minutes before the event**:

```
                 AUTOMATED PRE-WARMING SCHEDULE (-15 MINUTES)
                 
  T-15m ──► Set Kubernetes HPA MinReplicas = 100 (Bypasses reactive scale-up)
  T-12m ──► Pre-scale AWS Aurora Serverless ACUs / PostgreSQL Read Replicas
  T-10m ──► Warm application DB connection pools (HikariCP minimum-idle = max-pool)
  T-08m ──► Execute Cache Warming Script (Prefetch top 1,000 SKUs into Redis L1/L2)
  T-05m ──► Synthetic Traffic Probe (Fires 5,000 warmup queries to trigger JVM JIT compilation)
  T-00m ──► EVENT STARTS: All systems running at full capacity and optimal memory state!
```

---

## 4. Comprehensive Architectural Comparison Matrix

| Pattern / Approach | Primary Layer | Core Bottleneck Mitigated | Latency Impact | UX Impact | Major Failure Mode If Misconfigured | Industry Reference |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Queue Load Leveling** | Ingestion & Persistence | DB write saturation, connection pool drain | High (Async `202 Accepted` + Polling/SSE) | Requires async UI/UX (e.g., "Order queued") | Unbounded queue depth leading to broker disk exhaustion | Kafka, AWS SQS, Stripe Webhook Ingestion |
| **Multi-Tier Rate Limiting** | Edge, Gateway & Mesh | Rogue scrapers, DDoS, noisy neighbors | Negligible ($< 2\text{ms}$ Redis Lua) | HTTP 429 errors for abusive or over-quota users | Redis outage fails closed, blocking 100% of legitimate users | Cloudflare, Kong, Stripe API Gateway |
| **Adaptive Load Shedding** | Gateway & Ingress Mesh | Memory exhaustion, thread starvation, GC freezes | Low ($< 1\text{ms}$ in-memory check) | Drops degradable requests (HTTP 503) during saturation | Dropping mission-critical checkout paths instead of non-critical paths | Netflix Concurrency Limits, Twitter/X Mesh |
| **Circuit Breakers & Bulkheads** | Inter-Service Client RPC | Cascading thread exhaustion, slow dependencies | Zero (in-memory atomic state check) | Degrades cleanly to cached/fallback data | Setting failure threshold too sensitive, causing false-positive tripping | Resilience4j, Netflix Hystrix, Envoy Outlier Detection |
| **Backoff, Jitter & Retry Budgets** | Client Libraries & SDKs | Self-inflicted retry storms, thundering herds | Disperses retry traffic across time | Minor delay on transient network blips | Infinite retries without jitter, sustaining metastable failure loops | AWS SDKs, Finagle, gRPC |
| **Virtual Waiting Rooms & Pre-Warming** | Edge CDN & Anycast DNS | VM/Pod boot lag, cold starts, JIT compilation | Holds excess users outside origin infrastructure | Visual queue page ("You are #1,420 in line") | HMAC secret leak or desynchronized token expiration | Queue-IT, Ticketmaster, Cloudflare Waiting Room |

---

## 5. Production Observability, Alerting & Incident Runbook

### Key Golden Signals & Prometheus Metrics

To detect burst traffic and monitor resiliency subsystems in real time, production systems scrape the following Prometheus metrics:

```promql
# 1. Queue-to-Service Latency Ratio (Early indicator of queuing collapse)
# When Wait Time exceeds Execution Time, shedding must engage immediately.
rate(http_request_queue_duration_seconds_sum[1m]) / rate(http_request_execution_duration_seconds_sum[1m])

# 2. In-Flight Concurrency vs Adaptive Limit
# If in_flight / dynamic_limit >= 0.95, server is at the edge of shedding.
app_in_flight_requests / app_adaptive_concurrency_limit

# 3. Circuit Breaker State Gauge (0 = Closed, 1 = Half-Open, 2 = Open)
circuit_breaker_state{service="payment_service"}

# 4. Retry Amplification Factor
# Normal: 1.0 to 1.05. Danger: > 1.20 (Indicates active retry storm)
sum(rate(http_client_requests_total{is_retry="true"}[1m])) 
/ 
sum(rate(http_client_requests_total{is_retry="false"}[1m]))

# 5. Redis Rate Limiter P99 Latency
# If Redis takes > 10ms, gateway must trip to fail-open mode.
histogram_quantile(0.99, rate(redis_command_duration_seconds_bucket{cmd="evalsha"}[1m]))
```

---

### Production Incident Response: The Flash-Sale Disaster Runbook

When an unexpected $50\times$ traffic burst hits and services begin reporting elevated latencies and connection pool saturation, on-call site reliability engineers (SREs) follow this four-phase stabilization playbook:

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               5-MINUTE FLASH-SALE STABILIZATION RUNBOOK                          │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘

 [ Phase 1: Edge Defense ] (T + 60 Seconds)
 1. Enable Under-Attack Mode / Aggressive Edge Rate Limiting on Cloudflare / AWS WAF.
 2. Activate Virtual Waiting Room at the Edge: Cap origin throughput to 5,000 RPS.
 3. Extend Edge CDN Cache TTLs: Set `Cache-Control: public, max-age=300, stale-while-revalidate=600`
    on all product detail pages and catalog queries.

 [ Phase 2: Origin Load Shedding ] (T + 120 Seconds)
 1. Flip Global Feature Flags to Degraded Mode:
    - DISABLE: Recommendation algorithms (fallback to static top-10 list).
    - DISABLE: User reviews and ratings rendering.
    - DISABLE: Cart activity analytics and behavioral tracking webhooks.
 2. Restrict Adaptive Concurrency Limits on non-checkout ingress paths.

 [ Phase 3: Database & Worker Protection ] (T + 180 Seconds)
 1. Throttle background consumer worker pools: Scale down batch worker pods to relieve primary DB.
 2. Kill idle connections in connection pools (HikariCP / pgBouncer).
 3. Divert all non-transactional read traffic to read replicas.

 [ Phase 4: Metastable Recovery Verification ] (T + 300 Seconds)
 1. Inspect Retry Ratio Metric: Verify amplification factor is $< 1.10$.
 2. Verify JVM Garbage Collection times: Ensure STW pauses have normalized under 50ms.
 3. Gradually release Waiting Room admission tokens in increments of +500 RPS every 2 minutes.
```

---

## 6. Architectural Summary & Golden Rules

Building systems capable of withstanding extreme traffic bursts requires engineering for failure at every tier of the stack:

1. **Never block on writes during a burst:** Absorb mutations into durable streaming buffers (Kafka, SQS) and process them at a rate dictated by database capacity (**Load Leveling**).
2. **Never rely on auto-scaling for instantaneous bursts:** Auto-scalers take minutes to react; pre-warm capacity prior to scheduled events and hold unadmitted users at the Edge (**Waiting Rooms & Pre-Warming**).
3. **Measure Queuing Latency, not CPU:** CPU is a lagging indicator. Regulate system entry based on queue wait time and in-flight concurrency (**Adaptive Load Shedding**).
4. **Isolate failure domains:** Wrap external network calls in circuit breakers with dedicated bulkheads to prevent one slow dependency from draining the entire cluster's thread pool (**Circuit Breakers & Bulkheads**).
5. **Enforce client-side hygiene:** Mandate that all client SDKs implement Exponential Backoff with Full Jitter and enforce a strict 10% Retry Budget to eliminate catastrophic **Retry Storms**.
6. **Gracefully degrade over failing entirely:** It is always better to serve a simplified, static page without personalized recommendations than to return a `500 Internal Server Error`.
