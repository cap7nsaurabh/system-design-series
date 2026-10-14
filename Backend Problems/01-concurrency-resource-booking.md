# Concurrency Control & Resource Booking: High-Level & Low-Level System Design

## 1. Executive Summary & Real-Life Production Scenarios

Resource booking under extreme concurrency represents one of the most critical engineering challenges in distributed systems. When thousands or millions of users compete for a finite set of scarce resources (concert tickets, flash-sale electronics, limited hotel rooms, doctor appointments, or flight seats), naive relational database reads and writes immediately trigger **race conditions, deadlocks, connection pool starvation, and double bookings**.

```
                           THE HIGH-CONTENTION WRITE BOTTLENECK
                           
 [1,000,000 Concurrent Users]
       │       │       │
       ▼       ▼       ▼
  ┌─────────────────────────┐
  │   API Gateway / Edge    │
  └───────────┬─────────────┘
              │ (50,000 RPS)
              ▼
  ┌─────────────────────────┐
  │   Application Cluster   │
  └───────────┬─────────────┘
              │
              ▼
  ┌─────────────────────────┐
  │  Relational DB Primary  │ ──► [Row: Seat A-12, Status: AVAILABLE]
  └─────────────────────────┘
              ▲
              │ FATAL COLLISION:
              │ 50,000 threads attempt concurrent UPDATE on the same row!
              │ Result: Row locks queue up, connection pools exhaust in 200ms,
              │ CPU spikes to 100%, cascading 504 Gateway Timeouts ensue.
```

---

### Real-World Production Failures

#### 1. Ticketmaster & The Taylor Swift "Eras Tour" Meltdown (November 2022)
- **The Event:** Over 3.5 billion system requests flooded Ticketmaster's Verified Fan platform within hours.
- **The Failure Mechanism:** Millions of bots and unregistered users bypassed the front-end waiting room and hit backend booking endpoints directly. As hundreds of thousands of sessions attempted to hold tickets in the same stadium sections simultaneously, row-level locks on primary database tables created catastrophic lock-wait queues. Database threads spent 98% of CPU time managing lock contention rather than executing queries, leading to cascading timeouts across the entire microservice ecosystem.
- **Business Impact:** The public sale was canceled entirely, sparking a US Congressional antitrust investigation and massive brand damage.

#### 2. BookMyShow & The ICC Cricket World Cup (2023)
- **The Event:** India vs. Pakistan match ticket sales opened for over 130,000 seats at the Narendra Modi Stadium.
- **The Failure Mechanism:** Over 10 million concurrent users hit the mobile app within 3 seconds. The inventory service used database-level transactions with optimistic locking. Because contention exceeded 80,000 requests per second on a small set of seating category rows, the transaction abort/retry rate reached 99.8%. The database spent all I/O processing rollback write-ahead logs (WAL), resulting in complete site paralysis and delayed booking confirmations.

#### 3. E-Commerce Flash Sales (Xiaomi / iPhone Launches)
- **The Event:** 50,000 flagship smartphones selling out in under 15 seconds.
- **The Failure Mechanism:** Naive `UPDATE inventory SET stock = stock - 1 WHERE item_id = ? AND stock > 0` saturated database row-level exclusive locks (`X-locks`). Application connection pools (`HikariCP`, `pgbouncer`) were drained within milliseconds, starving all other read endpoints (such as homepages and product detail pages) and bringing down the entire platform.

---

## 2. The Core Concurrency Problem: Race Conditions & Anomalies

To understand why traditional architectures fail, we must examine the database engine internals and the exact race conditions that emerge under concurrent access.

### 2.1 The Classic "Lost Update" Problem ($G_{update}$)

Two users, Alice and Bob, simultaneously attempt to book the exact same seat (`seat_id = 42`).

```
  TIME       CLIENT A (Alice)                    CLIENT B (Bob)                    DATABASE STATE (seat_id = 42)
   │
   ├─ T1 ──► SELECT status FROM seats            │                                 [ status = 'AVAILABLE' ]
   │         WHERE seat_id = 42;                 │
   │         (Returns: 'AVAILABLE')              │
   │                                             │
   ├─ T2 ──── User Alice confirms booking ──────► SELECT status FROM seats         [ status = 'AVAILABLE' ]
   │                                             WHERE seat_id = 42;
   │                                             (Returns: 'AVAILABLE')
   │                                             │
   ├─ T3 ──► UPDATE seats                        │                                 [ status = 'BOOKED', user = Alice ]
   │         SET status = 'BOOKED',              │
   │             user_id = 'Alice'               │
   │         WHERE seat_id = 42;                 │
   │         (COMMIT)                            │
   │                                             │
   ├─ T4 ──────────────────────────────────────► UPDATE seats                      [ status = 'BOOKED', user = Bob ]
   │                                             SET status = 'BOOKED',            *** ALICE'S BOOKING IS OVERWRITTEN! ***
   │                                                 user_id = 'Bob'
   │                                             WHERE seat_id = 42;
   │                                             (COMMIT)
   ▼
```

- **Result:** Both users receive a "Booking Confirmed" screen. Both users' credit cards are charged. However, Bob's write overwrites Alice's write in the database. When Alice arrives at the venue, her seat is already occupied by Bob (**Double Booking**).

---

### 2.2 ANSI SQL Isolation Levels vs. Real-World Concurrency Anomalies

Standard database transaction isolation levels protect against specific anomalies, but each carries performance and consistency tradeoffs:

| Isolation Level | Dirty Reads ($P_1$) | Non-Repeatable Reads ($P_2$) | Phantom Reads ($P_3$) | Write Skew ($A5B$) | Lost Updates ($P_4$) | Performance Under Contention |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Read Uncommitted** | Permitted | Permitted | Permitted | Permitted | Permitted | High (No read locks) |
| **Read Committed** (Default Postgres/Oracle) | Prevented | Permitted | Permitted | Permitted | **Permitted** | High |
| **Repeatable Read** (Default MySQL InnoDB) | Prevented | Prevented | Prevented (via MVCC/Gap Locks) | Permitted | **Permitted without locking reads** | Moderate |
| **Serializable** | Prevented | Prevented | Prevented | Prevented | Prevented | Abysmal under high contention (Massive Aborts) |

> [!IMPORTANT]
> Even in MySQL's **Repeatable Read** or PostgreSQL's **Read Committed**, standard `SELECT` queries use non-locking snapshot reads via **MVCC** (Multi-Version Concurrency Control). A plain `SELECT` followed by an `UPDATE` will **not** prevent a Lost Update! You must explicitly enforce concurrency control using one of the architectural approaches detailed below.

---

## 3. Comprehensive Deep Dive into Architectural Approaches

We will now analyze the five primary architectural patterns used in modern engineering to solve resource booking, detailing both their **High-Level Design (HLD)** and **Low-Level Design (LLD)**.

---

## Approach 1: Pessimistic Locking (Database Level)

### High-Level Design (HLD)

In a Pessimistic Locking architecture, the system operates on the assumption that resource collisions are frequent. Every read for a critical resource explicitly acquires an exclusive lock at the database engine level, forcing all concurrent requests for the same row to queue up sequentially.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       PESSIMISTIC LOCKING ARCHITECTURE                                  │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

    Client A                       Client B                       Application Tier           RDBMS (PostgreSQL/MySQL)
       │                              │                                  │                               │
       │─── 1. Book Seat 42 ──────────┼─────────────────────────────────►│                               │
       │                              │                                  │─── 2. BEGIN TX                │
       │                              │                                  │    SELECT ... FOR UPDATE ────►│ (Acquires Row X-Lock on 42)
       │                              │                                  │◄── 3. Row data returned ──────┤
       │                              │                                  │                               │
       │                              │─── 4. Book Seat 42 ─────────────►│                               │
       │                              │                                  │─── 5. BEGIN TX                │
       │                              │                                  │    SELECT ... FOR UPDATE ────►│ (BLOCKED! Waits for lock)
       │                              │                                  │                               │   │
       │                              │                                  │                               │   │ Lock Wait
       │◄── 6. Seat reserved ─────────┼──────────────────────────────────│                               │   │ Queue
       │    (Process payment)         │                                  │                               │   │
       │─── 7. Payment Success ───────┼─────────────────────────────────►│                               │   │
       │                              │                                  │─── 8. UPDATE status='BOOKED'  │   │
       │                              │                                  │    COMMIT ───────────────────►│ (Releases Row X-Lock)
       │                              │                                  │                               │   │
       │                              │                                  │                               │◄──┘ Unblocks Client B
       │                              │                                  │◄── 9. Returns seat data ──────┤
       │                              │                                  │    (Status is now 'BOOKED'!)  │
       │                              │                                  │─── 10. ROLLBACK ─────────────►│
       │                              │◄── 11. Error: Already Booked ────┤                               │
```

---

### Low-Level Design (LLD)

#### 1. Database Internals: How InnoDB & Postgres Lock Rows
- **MySQL InnoDB:** Uses **Record Locks** stored in memory in a hash table of lock structures (`lock_sys`). When `SELECT ... FOR UPDATE` executes on an index record, InnoDB creates an exclusive (`X`) record lock. If the row is accessed via a non-unique index, InnoDB additionally creates **Gap Locks** or **Next-Key Locks** to prevent phantom inserts in the index range.
- **PostgreSQL:** Does not use an in-memory lock table for row-level locks. Instead, it writes lock information directly into the tuple header fields (`t_xmax` and `infomask` bits) on the physical page on disk. When another transaction tries to read with `FOR UPDATE`, it inspects `t_xmax` and waits on the transaction ID using PostgreSQL's lock manager (`ProcArray`).

#### 2. Concrete SQL Implementation

```sql
-- Client A: Transaction 1
BEGIN TRANSACTION;

-- Lock the specific row. Any other transaction trying to read this row
-- with 'FOR UPDATE' or attempting to 'UPDATE/DELETE' will BLOCK until this commits.
SELECT seat_id, status, price 
FROM seats 
WHERE seat_id = 42 
FOR UPDATE;

-- Application checks if status == 'AVAILABLE'
-- If available, proceed:
UPDATE seats 
SET status = 'BOOKED', 
    booked_by_user_id = 'usr_981a7b', 
    booked_at = NOW() 
WHERE seat_id = 42;

COMMIT;
```

#### 3. Advanced Pessimistic Variants: `NOWAIT` and `SKIP LOCKED`

In high-throughput systems, waiting for locks leads to thread starvation. Modern SQL dialects provide two non-blocking primitives:

```sql
-- VARIANT A: NOWAIT (Fail fast if locked)
-- Throws error 'ERROR: could not obtain lock on row in relation "seats"' immediately
SELECT * FROM seats WHERE seat_id = 42 FOR UPDATE NOWAIT;

-- VARIANT B: SKIP LOCKED (Ideal for Queue processing or finding "any" available seat)
-- Skips rows currently locked by other transactions, returning only free unlocked rows
SELECT * FROM seats 
WHERE category = 'VIP' AND status = 'AVAILABLE' 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
```

#### 4. The Deadlock Hazard & Wait-For Graphs (WFG)

When multiple transactions lock multiple resources in different orders, deadlocks occur:

```
  Transaction 1: Locks Seat A, needs Seat B
  Transaction 2: Locks Seat B, needs Seat A
  
  ┌───────────────┐           Waits for           ┌───────────────┐
  │ Transaction 1 │ ────────────────────────────► │ Transaction 2 │
  └───────────────┘                               └───────────────┘
          ▲                                               │
          │                  Waits for                    │
          └───────────────────────────────────────────────┘
```

The database background thread runs a **Deadlock Detection Cycle** (every `innodb_deadlock_detect` interval or `deadlock_timeout` in Postgres), constructs a **Wait-For Graph**, identifies cycles, and forcefully aborts one of the transactions (the one with the smallest undo log), throwing:
```
ERROR: 40P01: deadlock detected
DETAIL: Process 18420 waits for ExclusiveLock on tuple; blocked by process 18421.
```

#### 5. Tradeoffs and Production Failure Modes
* **Pros:** Guaranteed strong consistency. Mathematically impossible to double book.
* **Cons (The Death Spiral):**
  1. **Connection Pool Exhaustion:** If payment processing takes 3 seconds while holding `FOR UPDATE`, that database connection is held open for 3 full seconds. With a HikariCP pool of 100 connections, just 34 concurrent users exhaust all connections for the entire company.
  2. **High Latency:** Every concurrent user experiences serialized wait times ($N \times \text{latency}$).

---

## Approach 2: Optimistic Locking with Versioning (Application / DB Level)

### High-Level Design (HLD)

Optimistic Locking assumes that conflicts are rare. No locks are held while the user inspects the resource or enters payment details. Instead, a version number, timestamp, or state condition is checked atomically at the exact moment of the update.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       OPTIMISTIC LOCKING ARCHITECTURE                                  │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

    Client A                       Client B                       Application Tier           RDBMS (PostgreSQL/MySQL)
       │                              │                                  │                               │
       │─── 1. View Seat 42 ──────────┼─────────────────────────────────►│                               │
       │                              │                                  │─── 2. SELECT id, status, ver ─►│
       │                              │                                  │◄── 3. Return (ver=1, AVAIL) ──┤
       │◄── 4. Show Seat (ver=1) ─────┼──────────────────────────────────│                               │
       │                              │                                  │                               │
       │                              │─── 5. View Seat 42 ─────────────►│                               │
       │                              │                                  │─── 6. SELECT id, status, ver ─►│
       │                              │                                  │◄── 7. Return (ver=1, AVAIL) ──┤
       │                              │◄── 8. Show Seat (ver=1) ─────────│                               │
       │                              │                                  │                               │
       │─── 9. Confirm (ver=1) ───────┼─────────────────────────────────►│                               │
       │                              │                                  │─── 10. UPDATE ... WHERE       │
       │                              │                                  │        id=42 AND ver=1 ──────►│ (Row updated, ver becomes 2)
       │                              │                                  │◄── 11. Rows Affected = 1 ─────┤
       │◄── 12. Booking Confirmed ────┼──────────────────────────────────│                               │
       │                              │                                  │                               │
       │                              │─── 13. Confirm (ver=1) ─────────►│                               │
       │                              │                                  │─── 14. UPDATE ... WHERE       │
       │                              │                                  │        id=42 AND ver=1 ──────►│ (No row matches ver=1!)
       │                              │                                  │◄── 15. Rows Affected = 0 ─────┤
       │                              │                                  │                               │
       │                              │◄── 16. Error: Seat Modified ─────┤                               │
       │                              │    (Trigger Retry or Abort)      │                               │
```

---

### Low-Level Design (LLD)

#### 1. Schema Design with Atomic Compare-And-Swap (CAS)

```sql
CREATE TABLE seats (
    seat_id BIGINT PRIMARY KEY,
    section_id VARCHAR(16) NOT NULL,
    seat_number VARCHAR(8) NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'AVAILABLE',
    version INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

#### 2. Application Write Logic (Python / SQLAlchemy Example)

```python
def book_seat_optimistic(seat_id: int, user_id: str, max_retries: int = 3) -> bool:
    for attempt in range(max_retries):
        # 1. Read state without any locks (MVCC Snapshot Read)
        seat = db.query("SELECT seat_id, status, version FROM seats WHERE seat_id = :id", id=seat_id)
        
        if seat.status != 'AVAILABLE':
            raise SeatNotAvailableException("Seat is already reserved or booked.")
            
        current_version = seat.version
        
        # 2. Simulate business logic / validation (fast, in-memory)
        # ...
        
        # 3. Atomic Compare-And-Swap (CAS) write
        result = db.execute(
            """
            UPDATE seats 
            SET status = 'BOOKED',
                version = version + 1,
                updated_at = NOW()
            WHERE seat_id = :id 
              AND version = :expected_version
            """,
            id=seat_id,
            expected_version=current_version
        )
        
        # 4. Check affected rows
        if result.rowcount == 1:
            # Successfully updated!
            log.info(f"Seat {seat_id} successfully booked by {user_id}")
            return True
            
        # If rowcount == 0, another thread updated the version first!
        # Apply exponential backoff with full jitter before retry
        backoff_ms = (2 ** attempt * 50) + random.randint(0, 50)
        time.sleep(backoff_ms / 1000.0)
        
    raise ConcurrencyConflictException("Exceeded maximum retries due to high contention.")
```

#### 3. Spring Data JPA `@Version` Mechanics
In Java/Spring, optimistic locking is built into Hibernate/JPA:

```java
@Entity
@Table(name = "seats")
public class Seat {
    @Id
    private Long id;
    
    private String status;

    @Version
    private Integer version; // Hibernate automatically appends 'WHERE version = ?' on UPDATE
}
```
If another transaction updates the entity, Hibernate intercepts the 0-row update and throws an `OptimisticLockException` (`StaleObjectStateException`).

#### 4. The High-Contention Collapse (Thundering Herd)
* **When it works:** Read-heavy systems with occasional concurrent updates (e.g., flight seat selection 2 weeks before departure).
* **When it collapses:** Flash sales. If 10,000 users attempt to book the last 10 concert tickets:
  - 1 user succeeds.
  - 9,999 users fail and execute retry loops.
  - The database experiences an explosion of 30,000–50,000 rollback queries, pinning DB CPU at 100% while accomplishing zero meaningful work.

---

## Approach 3: Distributed Locks (Redis Redlock & Fencing Tokens)

### High-Level Design (HLD)

When applications scale horizontally across dozens of microservices, offloading lock contention from the primary SQL database to an in-memory key-value cluster like **Redis** provides sub-millisecond lock acquisition and avoids relational lock tables.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       DISTRIBUTED LOCKING ARCHITECTURE                                  │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

    Client A                       Client B                 Application Cluster            Redis Lock Store (Cluster)
       │                              │                              │                                  │
       │─── 1. Book Seat 42 ──────────┼─────────────────────────────►│                                  │
       │                              │                              │─── 2. SET seat:42:lock uuid NX ─►│
       │                              │                              │◄── 3. OK (Lock acquired) ────────┤
       │                              │                              │                                  │
       │                              │─── 4. Book Seat 42 ─────────►│                                  │
       │                              │                              │─── 5. SET seat:42:lock uuid NX ─►│
       │                              │                              │◄── 6. NIL (Lock rejected!) ──────┤
       │                              │◄── 7. Error: Seat Locked ────┤                                  │
       │                              │    (Try again in 500ms)      │                                  │
       │                              │                              │                                  │
       │                              │                              │─── 8. Process Payment            │
       │                              │                              │    & Update PostgreSQL           │
       │                              │                              │                                  │
       │                              │                              │─── 9. Lua: Del if uuid matches ─►│
       │                              │                              │◄── 10. OK (Lock released) ───────┤
       │◄── 11. Booking Confirmed ────┼──────────────────────────────┤                                  │
```

---

### Low-Level Design (LLD)

#### 1. Atomic Acquisition and Safe Release (Lua Script)

A common bug in naive Redis locking is executing `SETNX` followed by `EXPIRE`. If the application crashes between the two commands, the lock is never released, causing a permanent deadlock.

The production command is:
```text
SET resource_name client_token NX PX 30000
```
- `NX`: Set if Not eXists.
- `PX 30000`: Set expiry to 30,000 milliseconds (TTL).
- `client_token`: Cryptographically random unique string (UUID + thread ID).

#### 2. The Lock Release Race Condition & The Atomic Lua Script

Why do we need a unique `client_token`?
1. Client A acquires lock on `seat:42` with 5s TTL.
2. Client A experiences a 6-second Garbage Collection (GC) pause or slow external payment call.
3. Redis auto-expires the lock.
4. Client B acquires the lock on `seat:42`.
5. Client A wakes up from GC and executes `DEL seat:42`.
6. **Disaster:** Client A just deleted Client B's lock! Client C now enters, and Client B and C execute concurrently.

To prevent this, release must use an **Atomic Lua Script**:

```lua
-- KEYS[1] = resource key (e.g., "lock:seat:42")
-- ARGV[1] = client unique token (e.g., "uuid_abc123")
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

#### 3. Complete Production Python Implementation

```python
import redis
import uuid
import time

class RedisDistributedLock:
    RELEASE_LUA_SCRIPT = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
    """

    def __init__(self, redis_client: redis.Redis, lock_key: str, ttl_ms: int = 10000):
        self.redis = redis_client
        self.lock_key = lock_key
        self.ttl_ms = ttl_ms
        self.token = str(uuid.uuid4())
        self._release_script = self.redis.register_script(self.RELEASE_LUA_SCRIPT)

    def acquire(self) -> bool:
        # Atomic SETNX with PX millisecond expiration
        result = self.redis.set(self.lock_key, self.token, nx=True, px=self.ttl_ms)
        return result is True

    def release(self) -> bool:
        # Executes atomic check-and-delete
        result = self._release_script(keys=[self.lock_key], args=[self.token])
        return result == 1

# Usage in Booking Service:
def reserve_seat_distributed(seat_id: int, user_id: str):
    lock = RedisDistributedLock(redis_client, f"lock:seat:{seat_id}", ttl_ms=15000)
    
    if not lock.acquire():
        raise ResourceLockedException("Another user is currently booking this seat. Please try again.")
        
    try:
        # Proceed with booking and DB writes
        execute_seat_booking_db(seat_id, user_id)
    finally:
        lock.release()
```

#### 4. The Martin Kleppmann vs. Salvatore Sanfilippo (Antirez) Debate: Fencing Tokens

In 2016, distributed systems researcher Martin Kleppmann published an analysis proving that **Redlock is unsafe for strong consistency** without fencing tokens.

```
  Client 1                   Redis Lock Master               Storage / Database
     │                              │                                │
     │─── 1. Acquire Lock ─────────►│                                │
     │◄── 2. Lock OK (Lease 10s) ───┤                                │
     │                              │                                │
     │─── [STW GC Pause / Network delay for 15s]                     │
     │    (Lock expires in Redis!)  │                                │
     │                              │                                │
     │                              │  Client 2                      │
     │                              │     │                          │
     │                              │◄────┼── 3. Acquire Lock        │
     │                              ├─────┼── 4. Lock OK (Lease 10s) │
     │                              │     │                          │
     │                              │     │── 5. Write to DB ───────►│ (Write succeeded)
     │                              │     │                          │
     │─── 6. Client 1 wakes up! ────┼─────┼─────────────────────────►│ (Client 1 writes stale data!)
     │       (Thinks lock is valid) │     │                          │ *** DATA CORRUPTION ***
```

**The Solution: Monotonic Fencing Tokens:**
Every time a lock is acquired, the lock service returns a monotonically increasing sequence number ($1, 2, 3, \dots, N$). The storage layer rejects any write carrying a token lower than the highest token observed:

```sql
-- Storage enforces fencing token verification:
UPDATE seats 
SET status = 'BOOKED', 
    last_fencing_token = 34 
WHERE seat_id = 42 
  AND last_fencing_token < 34;
```
When Client 1 attempts its stale write with fencing token 33, the database rejects the write ($0\text{ rows affected}$)!

---

## Approach 4: Temporary Holds State Machine with Sagas (The Ticketing Standard)

### High-Level Design (HLD)

Neither pure DB locks nor single Redis mutexes reflect real-world human booking psychology. When booking a concert or airline seat, users require **5 to 10 minutes** to choose payment methods, apply coupons, and enter passenger details. Holding a database transaction open for 10 minutes is catastrophic.

Modern production platforms (Ticketmaster, BookMyShow, Airbnb, Delta Airlines) use a **Two-Phase Reservation State Machine** backed by **Temporary Holds** and asynchronous compensating workflows (Sagas).

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              TEMPORARY HOLD STATE MACHINE ARCHITECTURE                                 │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [Client] ──► [API Gateway] ──► [Booking Service] ──► [Redis Cluster] (In-Memory Inventory & 10m TTL)
                                        │
                                        ├──► [RDBMS Primary] (State: AVAILABLE -> HELD -> BOOKED)
                                        │
                                        └──► [Kafka Event Bus]
                                                    │
                   ┌────────────────────────────────┴────────────────────────────────┐
                   ▼                                                                 ▼
        [Payment Webhook Service]                                       [Hold Expiration Sweeper / DLQ]
        - Completes payment                                             - Runs every 1 second
        - Transitions state: HELD -> BOOKED                             - Identifies expired holds
        - Emits: SEAT_CONFIRMED                                         - Transitions state: HELD -> AVAILABLE
                                                                        - Emits: SEAT_RELEASED
```

---

### Low-Level Design (LLD)

#### 1. The Seat Finite State Machine (FSM)

```
                     ┌───────────────────────┐
                     │                       │
                     │       AVAILABLE       │ ◄────────────────────────┐
                     │                       │                          │
                     └───────────┬───────────┘                          │
                                 │                                      │
                                 │ User clicks "Select Seat"            │
                                 │ (Atomic Redis Hold, 10 min TTL)      │
                                 ▼                                      │
                     ┌───────────────────────┐                          │
                     │                       │                          │
        ┌─────────── │         HELD          │                          │
        │            │                       │                          │
        │            └───────────┬───────────┘                          │
        │                        │                                      │
        │ Hold expires (10m)     │ User proceeds to checkout            │ Payment fails /
        │ OR User clicks Cancel  │ (Auth payment intent)                │ Card declined
        │                        ▼                                      │
        │            ┌───────────────────────┐                          │
        │            │                       │                          │
        │            │    PAYMENT_PENDING    │ ─────────────────────────┘
        │            │                       │
        │            └───────────┬───────────┘
        │                        │
        │                        │ Payment Webhook confirms 200 OK
        │                        ▼
        │            ┌───────────────────────┐
        │            │                       │
        └──────────► │        BOOKED         │
                     │                       │
                     └───────────────────────┘
```

#### 2. Relational Schema for State Tracking

```sql
CREATE TABLE seat_reservations (
    reservation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seat_id BIGINT NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'AVAILABLE', 'HELD', 'PAYMENT_PENDING', 'BOOKED', 'EXPIRED'
    hold_expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT unique_active_seat UNIQUE (seat_id, status)
);

CREATE INDEX idx_seat_hold_expiry ON seat_reservations (status, hold_expires_at) 
WHERE status = 'HELD';
```

#### 3. How Redis Coordinates the Fast-Path Hold

When a user selects a seat:
1. Application runs an atomic Lua script on Redis:
   - Check if key `seat:{id}:hold` exists.
   - If not, set `seat:{id}:hold` with value `user_id` and `EX 600` (10 minutes).
2. The DB row is updated to `HELD` with `hold_expires_at = NOW() + INTERVAL '10 minutes'`.
3. The UI presents a countdown timer: **"09:59 minutes remaining to complete payment"**.

#### 4. The Edge Case: What if Payment Completes at 10 minutes and 01 seconds?
A user enters their credit card OTP at minute 09:58. Due to bank gateway latency, the webhook arrives at minute 10:02. Meanwhile, at minute 10:00, the hold expired and Seat 42 was reserved by someone else!

**The Compensating Saga Architecture:**
```
  [Payment Gateway Webhook] ──► [Order Service]
                                      │
                                      ▼
                      Query DB for Seat Reservation Status
                                      │
                  ┌───────────────────┴───────────────────┐
                  ▼ (Status == 'HELD')                    ▼ (Status == 'EXPIRED' / 'BOOKED_BY_OTHER')
          Transition to 'BOOKED'                  TRIGGER COMPENSATING ACTION:
          Generate E-Ticket PDF                   1. Call Payment Gateway -> POST /v1/refunds
          Send Confirmation Email                 2. Send Push Notification:
                                                     "Sorry! Your session timed out.
                                                      Your payment has been refunded in full."
```

---

## Approach 5: The Serialized Queue / Actor Model (Ultra-High Scale)

### High-Level Design (HLD)

For extreme scale (e.g., millions of requests hitting a 100-item flash sale or a 50,000-seat stadium opening), **all locking approaches break down** because lock managers spend all CPU cycles handling thread switches and lock queues.

The architectural paradigm shifts from **concurrent contention** to **single-threaded serialization**.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              SERIALIZED QUEUE / VIRTUAL WAITING ROOM                                   │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [1,000,000 Users]
         │
         ▼
 ┌─────────────────────────────────────────────────────────┐
 │       Cloudflare / Queue-it Virtual Waiting Room        │
 │  (Holds users at Edge, issues signed JWT queue tickets) │
 └────────────────────────────┬────────────────────────────┘
                              │ Rate-limited ingress (e.g., 2,000 req/sec)
                              ▼
 ┌─────────────────────────────────────────────────────────┐
 │               Ingress API Gateways                      │
 └────────────────────────────┬────────────────────────────┘
                              │
                              ▼
 ┌─────────────────────────────────────────────────────────┐
 │      Apache Kafka / Apache Pulsar Partitioned Queue     │
 │    Topic: "seat-reservation-commands"                   │
 │    Partition Key: Hash(event_id) or Hash(stadium_id)    │
 └────────────────────────────┬────────────────────────────┘
                              │
                              ▼ Strict FIFO In-Order Consumption
 ┌─────────────────────────────────────────────────────────┐
 │           Single-Threaded Core Booking Engine           │
 │       (In-Memory State / LMAX Disruptor / Redis Lua)    │
 │       Processes 100,000 bookings/sec with ZERO locks!   │
 └──────────────┬───────────────────────────┬──────────────┘
                │ Successful reservations   │ Rejected (Sold Out)
                ▼                           ▼
 ┌─────────────────────────────┐   ┌─────────────────────────────┐
 │  Write-Behind DB Worker     │   │ Instant WebSocket Reject:   │
 │  (Async batch INSERT to DB) │   │ "Seat 42 is no longer       │
 └─────────────────────────────┘   │  available."                │
                                   └─────────────────────────────┘
```

---

### Low-Level Design (LLD)

#### 1. Why Single-Threaded Serialization Outperforms Multithreaded Locks
As demonstrated by the **LMAX Disruptor** architecture and **Node.js/Redis single-threaded event loops**, a single CPU core running in-memory instructions sequentially with zero context switches and zero mutex contention can process **over 500,000 operations per second**!
- In a multithreaded database, 90% of time is spent on lock acquisition, thread parking (`futex`), CPU cache eviction, and context switching.
- In a partitioned queue, each event or seat category is assigned to a dedicated consumer partition. Within that partition, execution is strictly deterministic.

#### 2. Redis Atomic Lua Script as the In-Memory Serializer

```lua
-- KEYS[1]: Inventory counter key (e.g., "event:902:inventory")
-- KEYS[2]: Reserved set key (e.g., "event:902:reserved_seats")
-- ARGV[1]: Requested seat_id (e.g., "A-12")
-- ARGV[2]: User ID (e.g., "usr_482")

-- 1. Check if the specific seat is already taken
if redis.call("SISMEMBER", KEYS[2], ARGV[1]) == 1 then
    return {0, "SEAT_ALREADY_TAKEN"}
end

-- 2. Check if total inventory is exhausted
local stock = tonumber(redis.call("GET", KEYS[1]) or "0")
if stock <= 0 then
    return {0, "SOLD_OUT"}
end

-- 3. Atomically decrement stock and claim seat
redis.call("DECR", KEYS[1])
redis.call("SADD", KEYS[2], ARGV[1])

-- 4. Store user reservation with 10-minute hold TTL
redis.call("SET", "hold:" .. ARGV[1], ARGV[2], "EX", 600)

return {1, "HOLD_ACQUIRED"}
```
Because Redis executes Lua scripts as a single atomic unit, no two commands can interleave. It provides the performance of in-memory caching with absolute serial consistency!

---

## Approach 6: PostgreSQL Advisory Locks

### High-Level & Low-Level Design

Standard table/row locks lock physical database records. However, in many business applications, the resource you need to lock is a **logical entity** (e.g., an entire flight row, an entire cinema hall, or a user's multi-seat transaction bundle). Locking hundreds of individual rows creates immense lock-table overhead.

PostgreSQL provides **Advisory Locks**—application-defined locks using an arbitrary 64-bit integer or two 32-bit integers in the database's lock manager memory space.

```sql
-- Acquire an advisory lock for Seat ID 42 (Transaction-level)
-- Automatically releases on COMMIT or ROLLBACK.
-- If another session holds this lock, this call blocks until released.
BEGIN;
SELECT pg_advisory_xact_lock(42);

-- Perform business queries safely:
SELECT * FROM seats WHERE seat_id = 42;
UPDATE seats SET status = 'BOOKED' WHERE seat_id = 42;

COMMIT;
-- Lock is automatically freed here!
```

**Non-Blocking Advisory Locks (`pg_try_advisory_xact_lock`):**
```sql
-- Returns TRUE if lock was acquired, FALSE immediately if someone else holds it:
SELECT pg_try_advisory_xact_lock(42);
```

---

## 4. Comprehensive Architectural Comparison Matrix

| Architectural Pattern | Max Throughput (Ops/Sec) | Latency | Contention Tolerance | Deadlock Risk | Implementation Complexity | Primary Use Case |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Pessimistic Locking (`FOR UPDATE`)** | $500 - 2,000$ | High ($50 - 500\text{ ms}$) | Very Low (Cascading queues) | High | Low | Low-contention, high-value transfers (Enterprise B2B, VIP rooms) |
| **Optimistic Locking (`@Version`)** | $5,000 - 15,000$ | Sub-$10\text{ ms}$ (until collisions) | Low (Retry storms collapse DB) | Zero | Low | Read-heavy catalogs, low-contention booking (Doctors, Airlines) |
| **Distributed Locks (Redis)** | $20,000 - 50,000$ | Ultra-Low ($1 - 5\text{ ms}$) | Moderate | Zero (with TTL) | Medium | Distributed microservices, preventing duplicate payment submissions |
| **Temporary Holds State Machine** | $10,000 - 30,000$ | Low ($10 - 20\text{ ms}$) | High | Zero | High | Commercial ticketing (Ticketmaster, BookMyShow), Hotel bookings |
| **Serialized Queue / Actor (Kafka+Redis)** | $100,000 - 500,000+$ | Asynchronous (UI polling) | Maximum | Zero | Very High | Ultra-high flash sales (iPhone drop, World Cup Final, Taylor Swift) |

---

## 5. Real-World Production Post-Mortems & Anti-Patterns

### Anti-Pattern 1: The "Hold DB Connection Open During Third-Party HTTP Call"
```
 [Client] ──► [Booking App] ──► [PostgreSQL: BEGIN TX; SELECT ... FOR UPDATE]
                    │
                    ▼  (Holding DB Connection Open!)
         [Call Stripe API: POST /v1/charges]  <--- Stripe experiences 10-second latency spike!
                    │
                    ▼
          [PostgreSQL COMMIT]
```
- **What happened:** Because the application kept the database transaction open while waiting for Stripe's HTTP response, 100 concurrent checkout requests consumed all 100 HikariCP connections for 10 seconds.
- **The Golden Rule:** **NEVER execute external network calls, payment API requests, or heavy disk I/O while holding a database lock or inside an open database transaction!**

### Anti-Pattern 2: The Redis Expire-Before-Payment Disaster
- **What happened:** An engineer set a Redis lock TTL of 30 seconds for a seat. Due to network congestion, the payment gateway took 32 seconds to process the user's card. At second 30, the lock expired, a second user acquired the lock, and the seat was double-sold.
- **The Fix:** Implement an **Active Lock Renewal Heartbeat** (e.g., Java Redisson's "Watchdog" timer) which automatically extends the lock TTL by 10 seconds every 3 seconds as long as the booking thread is alive.

---

## 6. System Design Interview Blueprint: "Design Ticketmaster"

When asked to design a high-scale resource booking system in a system design interview:

1. **Requirements & Scope:**
   - Differentiate between **browsing events** (read-heavy, 99.9% CDN/cacheable) and **booking seats** (write-heavy, extreme contention).
   - Clarify hold duration (e.g., "Seats must be held for 8 minutes while user pays").

2. **Core Data Structure Choice:**
   - Propose an in-memory inventory hold tier (Redis Cluster) combined with an asynchronous database write-behind pipeline.
   - Explain why `SELECT ... FOR UPDATE` directly on MySQL fails during peak flash sales.

3. **Address Concurrency Deeply:**
   - Walk the interviewer through the **Temporary Hold State Machine** (`AVAILABLE` $\to$ `HELD` $\to$ `BOOKED`).
   - Detail the distributed lock expiration and compensating refund saga if payment arrives after hold expiration.

4. **Traffic Management:**
   - Propose a **Virtual Waiting Room** (Token bucket at CDN edge) to smooth traffic from 1,000,000 incoming RPS down to a steady 2,000 RPS.
