# 06 — Concurrency & Race Conditions in Spring Boot

> A production-grade, deep-dive reference into JVM memory models, race conditions, thread synchronization, asynchronous architectures, ThreadLocal context propagation, database locking, and distributed synchronization in Spring Boot 3.x and Java 17/21+.

---

## Table of Contents

1. [Java Concurrency Fundamentals in Spring Context](#1-java-concurrency-fundamentals-in-spring-context)
   - [1.1 JVM Memory Model & OS Thread Mapping](#11-jvm-memory-model--os-thread-mapping)
   - [1.2 Spring Beans are Singletons by Default](#12-spring-beans-are-singletons-by-default)
   - [1.3 Stateless vs. Stateful Bean Architectures](#13-stateless-vs-stateful-bean-architectures)
   - [1.4 The ThreadLocal Architecture in Spring](#14-the-threadlocal-architecture-in-spring)
2. [Race Conditions & Concurrency Hazards](#2-race-conditions--concurrency-hazards)
   - [2.1 Check-Then-Act Race Condition](#21-check-then-act-race-condition)
   - [2.2 Read-Modify-Write Race Condition](#22-read-modify-write-race-condition)
   - [2.3 Spring Singleton State Pollution](#23-spring-singleton-state-pollution)
   - [2.4 Race Condition Detection & Reproducibility](#24-race-condition-detection--reproducibility)
3. [Multi-Approach: In-Process Thread Synchronization Strategies](#3-multi-approach-in-process-thread-synchronization-strategies)
   - [Approach 1: Stateless Architecture (Stack Confinement & Immutability)](#approach-1-stateless-architecture-stack-confinement--immutability)
   - [Approach 2: Hardware-Level Compare-And-Swap (CAS) & Scalable Adders](#approach-2-hardware-level-compare-and-swap-cas--scalable-adders)
   - [Approach 3: Explicit Lock Primitives (ReentrantLock & ReadWriteLock & StampedLock)](#approach-3-explicit-lock-primitives-reentrantlock--readwritelock--stampedlock)
   - [Approach 4: Thread-Safe Concurrent Collections & Atomic Lambdas](#approach-4-thread-safe-concurrent-collections--atomic-lambdas)
   - [Approach 5: Intrinsic Monitor Synchronization with Dedicated Private Lock Monitors](#approach-5-intrinsic-monitor-synchronization-with-dedicated-private-lock-monitors)
   - [In-Process Synchronization Comparison Matrix](#in-process-synchronization-comparison-matrix)
4. [Multi-Approach: Asynchronous Task Execution Architectures](#4-multi-approach-asynchronous-task-execution-architectures)
   - [Approach 1: Declarative `@Async` with Custom `ThreadPoolTaskExecutor`](#approach-1-declarative-async-with-custom-threadpooltaskexecutor)
   - [Approach 2: Programmatic Pipeline Composition via `CompletableFuture`](#approach-2-programmatic-pipeline-composition-via-completablefuture)
   - [Approach 3: Java 21 / Spring Boot 3.2+ Virtual Threads (Project Loom)](#approach-3-java-21--spring-boot-32-virtual-threads-project-loom)
   - [Approach 4: Non-Blocking Reactive Streams (Spring WebFlux & Project Reactor)](#approach-4-non-blocking-reactive-streams-spring-webflux--project-reactor)
   - [Asynchronous Architecture Comparison Matrix](#asynchronous-architecture-comparison-matrix)
5. [Multi-Approach: ThreadContext Propagation across Asynchronous Boundaries](#5-multi-approach-threadcontext-propagation-across-asynchronous-boundaries)
   - [Approach 1: Spring Framework `TaskDecorator` Configuration (Standard)](#approach-1-spring-framework-taskdecorator-configuration-standard)
   - [Approach 2: Micrometer Context Propagation (Spring Boot 3.x Unified Standard)](#approach-2-micrometer-context-propagation-spring-boot-3x-unified-standard)
   - [Approach 3: Spring Security `DelegatingSecurityContextExecutor`](#approach-3-spring-security-delegatingsecuritycontextexecutor)
   - [Approach 4: Manual Context Capture, Transfer, and Eviction via Wrapper Closures](#approach-4-manual-context-capture-transfer-and-eviction-via-wrapper-closures)
   - [Context Propagation Comparison Matrix](#context-propagation-comparison-matrix)
6. [Multi-Approach: Database-Level Concurrency Control](#6-multi-approach-database-level-concurrency-control)
   - [Approach 1: Optimistic Locking with `@Version` & Spring Retry](#approach-1-optimistic-locking-with-version--spring-retry)
   - [Approach 2: Pessimistic Write Locking (`PESSIMISTIC_WRITE` / `SELECT ... FOR UPDATE`)](#approach-2-pessimistic-write-locking-pessimistic_write--select--for-update)
   - [Approach 3: Pessimistic Read Locking (`PESSIMISTIC_READ` / `SELECT ... FOR SHARE`)](#approach-3-pessimistic-read-locking-pessimistic_read--select--for-share)
   - [Approach 4: Single-Statement Atomic Database Mutations (`UPDATE ... WHERE condition`)](#approach-4-single-statement-atomic-database-mutations-update--where-condition)
   - [Database Concurrency Comparison Matrix](#database-concurrency-comparison-matrix)
7. [Multi-Approach: Distributed Locking & Multi-Node Synchronization](#7-multi-approach-distributed-locking--multi-node-synchronization)
   - [Approach 1: Redis Distributed Lock with Redisson (Watchdog Auto-Renewal)](#approach-1-redis-distributed-lock-with-redisson-watchdog-auto-renewal)
   - [Approach 2: Database Advisory Locks & ShedLock Coordination](#approach-2-database-advisory-locks--shedlock-coordination)
   - [Approach 3: Distributed Idempotency Token Pattern with Redis & Unique DB Keys](#approach-3-distributed-idempotency-token-pattern-with-redis--unique-db-keys)
   - [Distributed Locking Comparison Matrix](#distributed-locking-comparison-matrix)
8. [Production Thread Pool Sizing & Schedulers](#8-production-thread-pool-sizing--schedulers)
   - [8.1 Little's Law, CPU-Bound vs. I/O-Bound Pool Sizing](#81-littles-law-cpu-bound-vs-io-bound-pool-sizing)
   - [8.2 Saturation Policies (CallerRunsPolicy vs AbortPolicy)](#82-saturation-policies-callerrunspolicy-vs-abortpolicy)
   - [8.3 Graceful Shutdown Protocol](#83-graceful-shutdown-protocol)
   - [8.4 Micrometer Thread Pool Monitoring & Alerts](#84-micrometer-thread-pool-monitoring--alerts)
9. [Common Spring Concurrency Traps & Anti-Patterns](#9-common-spring-concurrency-traps--anti-patterns)
10. [Deep Dive: JMM, Volatile, & Happens-Before Order](#10-deep-dive-jmm-volatile--happens-before-order)
11. [Summary & Key Takeaways](#11-summary--key-takeaways)
12. [Practice Questions & Interview Scenarios](#12-practice-questions--interview-scenarios)

---

## 1. Java Concurrency Fundamentals in Spring Context

### 1.1 JVM Memory Model & OS Thread Mapping

Every standard JVM process maps Java threads 1:1 to OS-level kernel platform threads. A typical Spring Boot application on Tomcat defaults to a pool of 200 worker threads.

```
JVM Process Memory Layout
┌────────────────────────────────────────────────────────────────────────┐
│                        SHARED HEAP & METASPACE                         │
│  ┌────────────────────────┐  ┌──────────────────────────────────────┐  │
│  │ ApplicationContext     │  │ Singleton Beans                      │  │
│  │ (Bean definitions)     │  │ (OrderService, ProductRepository)    │  │
│  └────────────────────────┘  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Shared Objects, Cached Entities, Static Variables, Class Data    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────────────────┤
│                     THREAD-LOCAL STACK MEMORY                          │
│ ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────┐ │
│ │ Thread: tomcat-1     │ │ Thread: tomcat-2     │ │ Thread: async-1  │ │
│ │  - Frame: doFilter   │ │  - Frame: doFilter   │ │  - Frame: run    │ │
│ │  - Frame: handleReq  │ │  - Frame: handleReq  │ │  - Frame: notify │ │
│ │  - Local Primitives  │ │  - Local Primitives  │ │  - Local Prims   │ │
│ │  - Local References  │ │  - Local References  │ │  - Local Refs    │ │
│ └──────────────────────┘ └──────────────────────┘ └──────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Stack vs Heap Rule**: Method local variables, method arguments, and primitive types live strictly inside per-thread call stacks and are thread-safe by isolation. Any object referenced by a bean field lives on the heap and is accessible by all concurrent threads traversing that bean.

### 1.2 Spring Beans are Singletons by Default

By default, every Spring bean defined via `@Service`, `@Component`, `@Repository`, or `@RestController` is a **singleton**. There is only **one instance** of the class in the entire `ApplicationContext`.

```
Incoming Request 1 ──> [Thread: http-nio-8080-exec-1] ──┐
                                                        ├──> [SINGLETON BEAN]
Incoming Request 2 ──> [Thread: http-nio-8080-exec-2] ──┤   OrderService (1 Instance)
                                                        │
Incoming Request 3 ──> [Thread: http-nio-8080-exec-3] ──┘
```

If `OrderService` defines mutable instance variables (e.g., `private int counter = 0;`), concurrent HTTP requests will mutate the identical memory location simultaneously, producing data corruption.

### 1.3 Stateless vs. Stateful Bean Architectures

| Metric | Stateless Bean (Production Standard) | Stateful Bean (Anti-Pattern / Scoped) |
|---|---|---|
| **Instance Fields** | Only `final` injected dependencies (`Repository`, `Client`) | Mutable buffers, counters, user credentials, caches |
| **Thread Safety** | Inherently thread-safe across unlimited requests | Requires locks, atomics, or thread-local confinement |
| **Spring Scope** | `singleton` (default) | `request`, `session`, or `prototype` with scoped proxy |
| **Performance Overhead** | Zero synchronization overhead; CPU cache friendly | High lock contention or allocation garbage collection |

### 1.4 The ThreadLocal Architecture in Spring

Spring Framework leverages `ThreadLocal<T>` internally to pass infrastructural contexts down the invocation stack without polluting method signatures:
1. `SecurityContextHolder`: Holds the `Authentication` and `SecurityContext` for the calling user.
2. `RequestContextHolder`: Exposes `HttpServletRequest` and `HttpSession` to arbitrary services.
3. `TransactionSynchronizationManager`: Tracks active JDBC transactions, database connections, and session resources.

```
Thread A Stack ──────> ThreadLocalMap [Key: SecurityContextHolder.holder -> SecurityContext (User: "Alice")]
Thread B Stack ──────> ThreadLocalMap [Key: SecurityContextHolder.holder -> SecurityContext (User: "Bob")]
```

> [!CAUTION]
> **Thread Pool Memory Leak Hazard**: Because servlet containers (Tomcat, Jetty) recycle threads indefinitely, failing to call `ThreadLocal.remove()` in a `finally` block leaves stale security contexts or tenant identifiers on the thread. The next user routed to that thread will inherit the previous user's credentials!

---

## 2. Race Conditions & Concurrency Hazards

### 2.1 Check-Then-Act Race Condition

A Check-Then-Act defect occurs when a thread inspects an invariant, observes it to be valid, and then performs a mutating action—while an interleaved concurrent thread mutates that exact invariant between the check and the act.

```java
// BROKEN Check-Then-Act Pattern
@Service
public class VulnerableVoucherService {
    private final VoucherRepository voucherRepository;

    public void redeemVoucher(String voucherCode, Long userId) {
        Voucher voucher = voucherRepository.findByCode(voucherCode)
            .orElseThrow(() -> new IllegalArgumentException("Invalid voucher"));

        // Thread 1 checks: voucher.isRedeemed() == false
        // Thread 2 checks: voucher.isRedeemed() == false (Thread 1 hasn't committed yet!)
        if (!voucher.isRedeemed()) {
            voucher.setRedeemed(true);
            voucher.setRedeemedByUserId(userId);
            voucherRepository.save(voucher); // Both threads record redemption! Double spend!
        }
    }
}
```

### 2.2 Read-Modify-Write Race Condition

A Read-Modify-Write hazard manifests whenever an operation appears atomic at the source code level (e.g., `count++` or `balance -= amount`) but decompiles into distinct bytecode instructions:
1. `GETFIELD` (Read value from heap into thread register/operand stack)
2. `IADD` / `ISUB` (Modify value in register)
3. `PUTFIELD` (Write updated value back to heap)

```
Thread 1 (Core 1):  [READ balance: 100]  ──────> [CALC 100 - 20 = 80] ──────> [WRITE 80]
Thread 2 (Core 2):         [READ balance: 100]  ─────────> [CALC 100 - 30 = 70] ──────> [WRITE 70 (Overwrites 80!)]
```

The final balance is `70` instead of `50`. $20 of deductions were completely lost due to an uncoordinated write race.

### 2.3 Spring Singleton State Pollution

Developers accustomed to client-side programming often accidentally introduce mutable fields into Spring `@Controller` or `@Service` classes:

```java
@RestController
@RequestMapping("/api/v1/invoices")
public class BuggyInvoiceController {

    // INSTANCE FIELD IN SINGLETON: Shared by all concurrent HTTP requests!
    private String clientTaxId;
    private final List<InvoiceItem> pendingItems = new ArrayList<>();

    @PostMapping("/calculate")
    public InvoiceSummary calculateTax(@RequestBody InvoiceRequest request) {
        this.clientTaxId = request.getTaxId(); // Thread 1 sets "US-99", Thread 2 overwrites with "DE-11"
        this.pendingItems.clear();             // Thread 2 clears Thread 1's items halfway through calculation!
        this.pendingItems.addAll(request.getItems());
        return computeSummary(this.clientTaxId, this.pendingItems); // Corrupted data returned!
    }
}
```

### 2.4 Race Condition Detection & Reproducibility

Race conditions are notoriously difficult to reproduce in unit tests because standard JUnit test execution runs sequentially on a single thread. To test concurrency under load:
1. **`java.util.concurrent.CountDownLatch` Barrier**: Align multiple test threads at a starting gate so they simultaneously bombard the target method.
2. **`CompletableFuture.allOf(...)` Load Injections**: Fire batches of 1,000 asynchronous invocations against a local embedded database.
3. **Chaos Monkey / Jepsen Testing**: Introduce simulated network jitter and lock wait latency into lower database drivers.

---

## 3. Multi-Approach: In-Process Thread Synchronization Strategies

When mutable in-memory state must be coordinated inside a single Spring Boot JVM instance, architects have five distinct approaches available.

```
                    IN-PROCESS SYNCHRONIZATION SPECTRUM
 Low Contention                                            High Contention
 Low Complexity                                            High Complexity
       │                                                         │
       ▼                                                         ▼
[Approach 1]        [Approach 5]        [Approach 4]       [Approach 3]       [Approach 2]
Stateless Design  Synchronized Monitor Concurrent Collections Explicit Locks  Lock-Free CAS
(Stack Confinement) (Private Lock)       (ConcurrentHashMap) (Reentrant/ReadWrite) (LongAdder/CAS)
```

---

### Approach 1: Stateless Architecture (Stack Confinement & Immutability)

#### Scenario
Standard production Spring Boot web microservices handling high-throughput REST APIs, business validation, and database orchestration where state transitions belong inside databases, distributed caches, or request payloads.

#### Use-Case
User registration, shopping cart calculations, payment checkout processing, and reporting where each request carries all necessary parameters, transforms them locally, and commits them to persistent storage.

#### Benefits Over Others / Trade-offs
- **Benefits**: 100% thread-safe without locks; zero CPU synchronization overhead; immune to deadlocks and race conditions; scales linearly across CPU cores.
- **Trade-offs**: Cannot maintain persistent in-memory state across multiple HTTP requests on the same node without querying external state stores (Redis/PostgreSQL).

#### Well-Documented Code Example

```java
package com.example.service.impl;

import com.example.dto.OrderRequest;
import com.example.dto.OrderResponse;
import com.example.entity.Order;
import com.example.repository.OrderRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.Objects;

/**
 * Approach 1: Fully Stateless Service
 * All dependencies are immutable (final) and all state lives inside method stack frames.
 */
@Service
public class StatelessOrderService {

    // Injected dependencies are immutable references
    private final OrderRepository orderRepository;
    private final TaxCalculationClient taxClient;

    public StatelessOrderService(OrderRepository orderRepository, TaxCalculationClient taxClient) {
        this.orderRepository = Objects.requireNonNull(orderRepository, "Repository must not be null");
        this.taxClient = Objects.requireNonNull(taxClient, "TaxClient must not be null");
    }

    @Transactional
    public OrderResponse processOrder(final OrderRequest request) {
        // Stack confinement: request, taxRate, total, and newOrder live strictly on THIS thread's stack
        BigDecimal taxRate = taxClient.fetchTaxRate(request.getZipCode());
        BigDecimal subtotal = request.getItems().stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal grandTotal = subtotal.add(subtotal.multiply(taxRate));

        // Create new instance; no shared mutable instance variables
        Order newOrder = new Order(request.getCustomerId(), grandTotal, List.copyOf(request.getItems()));
        Order savedOrder = orderRepository.save(newOrder);

        return new OrderResponse(savedOrder.getId(), savedOrder.getTotal(), savedOrder.getStatus());
    }
}
```

---

### Approach 2: Hardware-Level Compare-And-Swap (CAS) & Scalable Adders

#### Scenario
High-throughput telemetry, real-time counters, metrics collection, or rate counters where thousands of concurrent threads update numerical counters or state machines simultaneously.

#### Use-Case
Tracking active API connections, aggregating total latency across incoming servlet requests, implementing an atomic state machine transition (`INITIALIZING` -> `RUNNING` -> `SHUTTING_DOWN`).

#### Benefits Over Others / Trade-offs
- **Benefits**: Non-blocking lock-free execution backed directly by CPU hardware instructions (`CMPXCHG`); no thread context switching or kernel transitions. Under extreme write contention, `LongAdder` distributes increments across internal CPU cache-line aligned cell buffers.
- **Trade-offs**: Complex compound operations (updating two independent counters simultaneously) cannot be executed atomically with a single CAS variable; `LongAdder` consumes more memory than a primitive long.

#### Well-Documented Code Example

```java
package com.example.concurrency.metrics;

import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.LongAdder;

/**
 * Approach 2: Non-blocking CAS Atomics & LongAdders
 */
@Component
public class HighThroughputMetricsRegistry {

    // Scalable counter: eliminates memory bus contention under thousands of concurrent threads
    private final LongAdder totalProcessedRequests = new LongAdder();
    private final LongAdder totalExecutionTimeMs = new LongAdder();

    // AtomicInteger: perfect for tracking active live connections with exact increment/decrement
    private final AtomicInteger activeConnections = new AtomicInteger(0);

    // AtomicReference: atomic state machine transitions without locks
    public enum EngineState { STOPPED, STARTING, RUNNING, FAULTED }
    private final AtomicReference<EngineState> engineState = new AtomicReference<>(EngineState.STOPPED);

    public void recordInvocation(long durationMs) {
        totalProcessedRequests.increment();      // Stripped cell write; 10x faster than AtomicLong under contention
        totalExecutionTimeMs.add(durationMs);
    }

    public int onConnectionOpened() {
        return activeConnections.incrementAndGet(); // Returns exact integer value atomically
    }

    public int onConnectionClosed() {
        return activeConnections.decrementAndGet();
    }

    public boolean transitionToRunning() {
        // Compare-And-Set: Only updates to RUNNING if and only if current value is exactly STARTING
        return engineState.compareAndSet(EngineState.STARTING, EngineState.RUNNING);
    }

    public double getAverageLatencyMs() {
        long count = totalProcessedRequests.sum();
        return count == 0 ? 0.0 : (double) totalExecutionTimeMs.sum() / count;
    }
}
```

---

### Approach 3: Explicit Lock Primitives (ReentrantLock & ReadWriteLock & StampedLock)

#### Scenario
Complex critical sections requiring timed lock acquisition (`tryLock(timeout)`), fairness policies, multiple condition variables (`await()` / `signal()`), or highly read-skewed in-memory data structures (95% reads, 5% writes).

#### Use-Case
Local memory cache invalidation, synchronizing physical hardware connectors, coordinating thread pools with graceful draining, or large in-memory lookup maps where read operations must execute concurrently without blocking each other.

#### Benefits Over Others / Trade-offs
- **Benefits**: Fine-grained control; supports non-blocking polling, timed acquisition (preventing deadlocks), and separate read/write lock capabilities. `StampedLock` supports lock-free optimistic reading.
- **Trade-offs**: Manual lock management is error-prone; forgetting to release the lock in a `finally` block creates a permanent deadlock; higher memory overhead per lock instance.

#### Well-Documented Code Example

```java
package com.example.concurrency.locks;

import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * Approach 3: Explicit ReentrantLock and ReadWriteLock
 */
@Service
public class ControlledAccessBufferService {

    // Fair lock ensures First-In, First-Out (FIFO) queue for critical hardware access
    private final ReentrantLock hardwareLock = new ReentrantLock(true);

    // ReadWriteLock for read-heavy in-memory cache
    private final Map<String, String> internalCache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    /**
     * Non-blocking timed lock attempt to prevent application thread starvation.
     */
    public boolean executeHardwareCommand(String command, long timeoutSeconds) {
        try {
            // Attempt to acquire lock within window; returns false if unavailable
            if (!hardwareLock.tryLock(timeoutSeconds, TimeUnit.SECONDS)) {
                return false; // Safely back off without thread blocking
            }
            try {
                // Execute delicate critical section
                dispatchToPhysicalPort(command);
                return true;
            } finally {
                // MANDATORY: Always unlock inside finally block
                hardwareLock.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // Restore interrupted status
            throw new IllegalStateException("Thread interrupted while waiting for lock", e);
        }
    }

    /**
     * Concurrent readers execute simultaneously without blocking each other.
     */
    public String getCachedValue(String key) {
        readLock.lock();
        try {
            return internalCache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    /**
     * Writers acquire exclusive access, blocking all concurrent readers and writers.
     */
    public void putCachedValue(String key, String value) {
        writeLock.lock();
        try {
            internalCache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    private void dispatchToPhysicalPort(String cmd) {
        // Low-level port communication
    }
}
```

---

### Approach 4: Thread-Safe Concurrent Collections & Atomic Lambdas

#### Scenario
Shared in-memory state requiring high concurrency across thousands of requests without coarse lock boundaries, such as active session stores, WebSocket registry maps, or local debounce caches.

#### Use-Case
Tracking connected WebSocket users per tenant, debouncing rapid duplicate HTTP submissions, or computing expensive calculations on first cache miss.

#### Benefits Over Others / Trade-offs
- **Benefits**: Lock striping (bucket-level locking on writes) and lock-free volatile reads; atomic mutation methods (`computeIfAbsent`, `computeIfPresent`, `merge`) execute compound check-then-act operations atomically without external locks.
- **Trade-offs**: Does not allow null keys or values; aggregations like `size()` or `isEmpty()` represent weakly consistent snapshots under ongoing writes.

#### Well-Documented Code Example

```java
package com.example.concurrency.collections;

import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * Approach 4: High-Performance Concurrent Collections with Atomic Mutators
 */
@Service
public class TenantSessionRegistry {

    public record SessionInfo(String sessionId, String userId, Instant lastActive) {}

    // ConcurrentHashMap provides bucket-level synchronization on writes and lock-free reads
    private final ConcurrentMap<String, SessionInfo> sessionMap = new ConcurrentHashMap<>();

    /**
     * Atomic Check-Then-Act: Ensures only ONE session object is instantiated and stored,
     * even if 100 concurrent threads invoke this method simultaneously for the same token.
     */
    public SessionInfo getOrCreateSession(String token, String userId) {
        return sessionMap.computeIfAbsent(token, key -> {
            // Expensive initialization executes exactly once per distinct key
            return new SessionInfo(key, userId, Instant.now());
        });
    }

    /**
     * Atomic In-Place Mutation: Modifies existing session metadata atomically without external lock.
     */
    public SessionInfo refreshSessionActivity(String token) {
        return sessionMap.computeIfPresent(token, (key, existing) -> 
            new SessionInfo(existing.sessionId(), existing.userId(), Instant.now())
        );
    }

    /**
     * Atomic Remove: Guarantees eviction only occurs if value matches expected state.
     */
    public boolean evictSession(String token, SessionInfo expectedSession) {
        return sessionMap.remove(token, expectedSession);
    }
}
```

---

### Approach 5: Intrinsic Monitor Synchronization with Dedicated Private Lock Monitors

#### Scenario
Legacy maintenance, simple synchronization of compound operations within a single class, or where Spring AOP proxy interception makes `synchronized(this)` hazardous.

#### Use-Case
Serializing execution of an internal queue drainer or protecting a shared third-party legacy SDK object that is inherently not thread-safe.

#### Benefits Over Others / Trade-offs
- **Benefits**: Clean syntax supported natively by Java keywords; automatically unlocks when exceptions are thrown (no manual `finally` block needed).
- **Trade-offs**: Unbounded blocking (no timeout support); cannot interrupt waiting threads; prone to proxy bugs if locking on `this`.

#### Well-Documented Code Example

```java
package com.example.concurrency.intrinsic;

import org.springframework.stereotype.Component;

import java.util.ArrayDeque;
import java.util.Queue;

/**
 * Approach 5: Dedicated Private Lock Monitor Pattern
 * NEVER use 'synchronized(this)' in Spring beans because proxies wrap 'this'.
 */
@Component
public class SafeLegacyBridgeService {

    // Private final monitor object: Outside code cannot lock on it or cause accidental deadlocks
    private final Object monitor = new Object();
    private final Queue<String> internalQueue = new ArrayDeque<>();

    public void enqueuePayload(String payload) {
        synchronized (monitor) {
            internalQueue.offer(payload);
            // Intrinsic condition notification
            monitor.notifyAll();
        }
    }

    public String drainNextPayload() throws InterruptedException {
        synchronized (monitor) {
            while (internalQueue.isEmpty()) {
                // Releases monitor lock and suspends thread until notified
                monitor.wait();
            }
            return internalQueue.poll();
        }
    }
}
```

---

### In-Process Synchronization Comparison Matrix

| Synchronization Strategy | Concurrency Model | CPU Contention Overhead | Timeout Support | Safe with Spring Proxies? | Primary Use-Case |
|---|---|---|---|---|---|
| **Approach 1: Stateless** | None (Isolated Stacks) | **Zero** | N/A | **Yes** (100% Safe) | Web APIs, Business Logic, Repositories |
| **Approach 2: CAS / Scalable Atomics** | Lock-Free Hardware CAS | **Extremely Low** | No | **Yes** | High-throughput Counters, Metrics, Status Flags |
| **Approach 3: Explicit Locks** | Lock-Based (Kernel / ThreadPark) | Moderate to High | **Yes** (`tryLock`) | **Yes** | Complex critical blocks, Read-heavy caches |
| **Approach 4: Concurrent Collections** | Bucket-level CAS / Locks | **Low** | No | **Yes** | Shared in-memory session/cache tables |
| **Approach 5: Private Monitor** | Intrinsic JVM Monitors | Moderate | No | **Yes** (With private object) | Simple critical blocks, legacy queue bridges |

---

## 4. Multi-Approach: Asynchronous Task Execution Architectures

When long-running work (generating PDF invoices, sending bulk transactional emails, firing third-party webhooks) must be decoupled from the incoming HTTP servlet request thread, Spring Boot provides four distinct architectural approaches.

---

### Approach 1: Declarative `@Async` with Custom `ThreadPoolTaskExecutor`

#### Scenario
Standard Spring Boot enterprise applications requiring clean, declarative decoupling of background tasks while retaining full Spring dependency injection and AOP interception.

#### Use-Case
Sending email receipts upon checkout, auditing security events to append-only logs, and asynchronous notifications.

#### Benefits Over Others / Trade-offs
- **Benefits**: Clean annotation-driven design (`@Async`); seamless integration with Spring's exception handling and transaction managers.
- **Trade-offs**: Subject to Spring AOP self-invocation traps; requires explicit configuration of a custom `ThreadPoolTaskExecutor` (the default `SimpleAsyncTaskExecutor` creates unbounded threads and crashes JVMs under load).

#### Well-Documented Code Example

```java
package com.example.async.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.lang.reflect.Method;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * Production-ready ThreadPoolTaskExecutor configuration for @Async.
 */
@Configuration
@EnableAsync
public class AsyncThreadPoolConfig implements AsyncConfigurer {

    private static final Logger log = LoggerFactory.getLogger(AsyncThreadPoolConfig.class);

    @Override
    @Bean(name = "productionTaskExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // 1. Core Pool: standing pool of worker threads
        executor.setCorePoolSize(10);
        // 2. Queue Capacity: tasks wait here while core threads are busy
        executor.setQueueCapacity(250);
        // 3. Max Pool: only spawned when queue capacity is 100% saturated
        executor.setMaxPoolSize(50);
        // 4. Rejection Policy: CallerRuns provides backpressure by running task on HTTP thread
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 5. Thread Naming: critical for identifying workers in log traces
        executor.setThreadNamePrefix("async-worker-");
        // 6. Graceful Shutdown: finish in-flight tasks when SIGTERM received
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);

        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (Throwable ex, Method method, Object... params) -> {
            log.error("Unhandled async error in method [{}], Params: {}", method.getName(), params, ex);
        };
    }
}
```

```java
package com.example.async.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class OrderNotificationService {

    private static final Logger log = LoggerFactory.getLogger(OrderNotificationService.class);

    @Async("productionTaskExecutor")
    public CompletableFuture<Boolean> sendOrderConfirmationEmailAsync(String emailAddress, Long orderId) {
        log.info("Sending confirmation email for order [{}] to [{}] on thread [{}]", 
            orderId, emailAddress, Thread.currentThread().getName());
        try {
            // Simulate external email gateway network latency
            Thread.sleep(500);
            return CompletableFuture.completedFuture(true);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

---

### Approach 2: Programmatic Pipeline Composition via `CompletableFuture`

#### Scenario
Complex orchestration where a single service method must dispatch multiple parallel asynchronous outbound requests (e.g., querying 3 third-party payment providers simultaneously) and combine their results.

#### Use-Case
Aggregating account dashboards (fetching balances, recent transactions, reward points, and credit score simultaneously from distinct microservices).

#### Benefits Over Others / Trade-offs
- **Benefits**: Rich functional composition API (`thenApply`, `thenCompose`, `thenCombine`, `allOf`, `anyOf`); explicit error recovery via `exceptionally` and `handle`; immune to Spring proxy self-invocation bugs.
- **Trade-offs**: Complex exception wrapping (`CompletionException`); requires passing explicit custom executors to prevent falling back to the shared `ForkJoinPool.commonPool()`.

#### Well-Documented Code Example

```java
package com.example.async.pipeline;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;

@Service
public class AggregatedDashboardService {

    public record DashboardSummary(String userProfile, String balance, String creditScore) {}

    private final Executor productionTaskExecutor;

    public AggregatedDashboardService(@Qualifier("productionTaskExecutor") Executor executor) {
        this.productionTaskExecutor = executor;
    }

    public CompletableFuture<DashboardSummary> loadUserDashboardAsync(Long userId) {
        // Step 1: Fire three parallel requests using our custom thread pool
        CompletableFuture<String> profileFuture = CompletableFuture.supplyAsync(
            () -> fetchProfile(userId), productionTaskExecutor
        );

        CompletableFuture<String> balanceFuture = CompletableFuture.supplyAsync(
            () -> fetchBalance(userId), productionTaskExecutor
        );

        CompletableFuture<String> creditScoreFuture = CompletableFuture.supplyAsync(
            () -> fetchCreditScore(userId), productionTaskExecutor
        ).exceptionally(ex -> "Score Unavailable"); // Fallback on partial failure

        // Step 2: Combine all three futures when all finish
        return CompletableFuture.allOf(profileFuture, balanceFuture, creditScoreFuture)
            .thenApply(voidResult -> new DashboardSummary(
                profileFuture.join(),
                balanceFuture.join(),
                creditScoreFuture.join()
            ));
    }

    private String fetchProfile(Long id) { return "Profile-Data-" + id; }
    private String fetchBalance(Long id) { return "$1,500.00"; }
    private String fetchCreditScore(Long id) { return "780"; }
}
```

---

### Approach 3: Java 21 / Spring Boot 3.2+ Virtual Threads (Project Loom)

#### Scenario
High-concurrency Spring Boot applications on Java 21+ characterized by blocking I/O (JPA/Hibernate database queries, downstream REST calls via `RestClient`, Redis operations).

#### Use-Case
Microservices serving 50,000+ concurrent long-lived connections, slow external legacy APIs, or batch operations where platform thread limits bottleneck application throughput.

#### Benefits Over Others / Trade-offs
- **Benefits**: Near-zero RAM footprint (~1KB vs 1MB per thread); millions of virtual threads can exist concurrently; preserves simple imperative blocking coding styles; eliminates complex reactive callbacks.
- **Trade-offs**: Pinned carrier thread hazard when executing inside `synchronized` blocks or JNI native code; virtual threads must NEVER be pooled.

#### Well-Documented Code Example

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

```java
package com.example.async.virtual;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.AsyncTaskExecutor;
import org.springframework.core.task.support.TaskExecutorAdapter;

import java.util.concurrent.Executors;

/**
 * Approach 3: Programmatic Virtual Thread Configuration (Spring Boot 3.2+)
 */
@Configuration
public class VirtualThreadArchitectureConfig {

    @Bean(name = "applicationTaskExecutor")
    @ConditionalOnProperty(name = "spring.threads.virtual.enabled", havingValue = "true")
    public AsyncTaskExecutor virtualThreadAsyncTaskExecutor() {
        // Spawns an unbound, lightweight Virtual Thread per submitted task
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

---

### Approach 4: Non-Blocking Reactive Streams (Spring WebFlux & Project Reactor)

#### Scenario
Event-driven streaming applications, massive WebSocket hubs, Server-Sent Events (SSE), or systems where hardware resources are severely constrained and end-to-end non-blocking drivers (R2DBC, Reactive Mongo) are available.

#### Use-Case
Real-time financial ticker streaming, IoT sensor ingestion gateways, high-density chat servers.

#### Benefits Over Others / Trade-offs
- **Benefits**: Ultra-low resource consumption; runs on a fixed number of worker threads equal to CPU cores; built-in reactive backpressure.
- **Trade-offs**: Complete rewrite of programming model (`Mono`/`Flux`); blocking calls anywhere in the pipeline stall the entire Netty event loop; incompatible with classic JPA/JDBC.

#### Well-Documented Code Example

```java
package com.example.reactive.controller;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;

import java.time.Duration;

@RestController
@RequestMapping("/api/v1/stream")
public class ReactiveStreamController {

    @GetMapping(value = "/telemetry", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamTelemetry() {
        return Flux.interval(Duration.ofMillis(200))
            .map(sequence -> "Data Packet #" + sequence + " at " + System.currentTimeMillis())
            .publishOn(Schedulers.parallel()); // Non-blocking event loop execution
    }
}
```

---

### Asynchronous Architecture Comparison Matrix

| Feature | `@Async` + `ThreadPoolTaskExecutor` | Programmatic `CompletableFuture` | Java 21 Virtual Threads | Reactive (Spring WebFlux) |
|---|---|---|---|---|
| **Programming Paradigm** | Imperative / Annotation | Functional / Chaining | Imperative / Blocking | Reactive Streams (`Mono`/`Flux`) |
| **Thread Model** | OS Platform Threads | OS Platform Threads | Lightweight JVM Virtual Threads | Fixed Event Loop (Netty) |
| **Max Concurrency** | Hundreds (OS thread limited) | Hundreds | **Millions** | **Millions** |
| **Memory Footprint** | ~1MB per thread stack | ~1MB per thread stack | **~1-2KB per virtual thread** | Minimal (Event-driven) |
| **Database Compatibility** | JPA / Hibernate / JDBC | JPA / Hibernate / JDBC | JPA / Hibernate / JDBC | **R2DBC Only** (No blocking JDBC) |
| **Risk / Pitfall** | Self-invocation bypass; OOM | Shared commonPool starvation | Pinned carrier thread on `synchronized` | Accidental blocking stalls event loop |

---

## 5. Multi-Approach: ThreadContext Propagation across Asynchronous Boundaries

Because `ThreadLocal` variables are bound strictly to the calling OS thread, submitting a task to a background thread pool strips all context (`SecurityContext`, `MDC` trace IDs, `Locale`). The following four approaches solve this propagation dilemma.

---

### Approach 1: Spring Framework `TaskDecorator` Configuration (Standard)

#### Scenario
Production Spring Boot applications utilizing `@Async` or standard `ThreadPoolTaskExecutor` beans that must reliably transfer MDC logging trace IDs (`traceId`, `spanId`) and `SecurityContext` to every async execution.

#### Use-Case
Distributed tracing with Logback/SLF4J, propagating the currently authenticated user's credentials to async worker threads.

#### Benefits Over Others / Trade-offs
- **Benefits**: Centralized; completely non-invasive; works automatically for every `@Async` method routed through that executor; guarantees context cleanup in a `finally` block to prevent thread contamination.
- **Trade-offs**: Only applies to tasks submitted through the configured `ThreadPoolTaskExecutor`; does not apply to manual `new Thread()` or unconfigured pools.

#### Well-Documented Code Example

```java
package com.example.async.decorator;

import org.slf4j.MDC;
import org.springframework.core.task.TaskDecorator;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Map;

/**
 * Captures context from the calling thread and reinstates it onto the worker thread.
 */
public class SecurityAndMdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        // 1. Capture context on the CALLING thread (e.g., Tomcat HTTP thread)
        Map<String, String> mdcContext = MDC.getCopyOfContextMap();
        SecurityContext securityContext = SecurityContextHolder.getContext();

        return () -> {
            // 2. Set context on the WORKER thread
            Map<String, String> originalMdc = MDC.getCopyOfContextMap();
            if (mdcContext != null) {
                MDC.setContextMap(mdcContext);
            }
            SecurityContextHolder.setContext(securityContext);

            try {
                // 3. Execute the actual async task
                runnable.run();
            } finally {
                // 4. CRITICAL: Clean up worker thread context to prevent pollution
                MDC.clear();
                if (originalMdc != null) {
                    MDC.setContextMap(originalMdc);
                }
                SecurityContextHolder.clearContext();
            }
        };
    }
}
```

```java
// Register decorator inside configuration
@Bean("decoratedTaskExecutor")
public ThreadPoolTaskExecutor decoratedTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(100);
    executor.setTaskDecorator(new SecurityAndMdcTaskDecorator());
    executor.initialize();
    return executor;
}
```

---

### Approach 2: Micrometer Context Propagation (Spring Boot 3.x Unified Standard)

#### Scenario
Spring Boot 3.x applications integrating Micrometer Tracing, OpenTelemetry, Project Reactor, and Virtual Threads where a unified context abstraction must span reactive, async, and blocking boundaries.

#### Use-Case
End-to-end tracing across hybrid WebFlux, `CompletableFuture`, and Virtual Thread pipelines.

#### Benefits Over Others / Trade-offs
- **Benefits**: Native to Spring Boot 3; supports both reactive and imperative threads; unified API.
- **Trade-offs**: Requires the `io.micrometer:context-propagation` library dependency; slightly steeper initial configuration.

#### Well-Documented Code Example

```java
package com.example.async.micrometer;

import io.micrometer.context.ContextRegistry;
import io.micrometer.context.ContextSnapshot;
import org.slf4j.MDC;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.Executor;

@Configuration
public class MicrometerPropagationConfig {

    static {
        // Register MDC accessor with Micrometer Context Registry
        ContextRegistry.getInstance().registerThreadLocalAccessor(
            "mdc.traceId",
            () -> MDC.get("traceId"),
            traceId -> MDC.put("traceId", traceId),
            () -> MDC.remove("traceId")
        );
    }

    public static <T> Runnable wrapWithSnapshot(Runnable task) {
        // Captures all registered ThreadLocals and returns a decorated Runnable
        return ContextSnapshot.captureAll().wrap(task);
    }
}
```

---

### Approach 3: Spring Security `DelegatingSecurityContextExecutor`

#### Scenario
Applications exclusively needing to propagate `SecurityContext` without custom tracing mechanisms.

#### Use-Case
Passing authentication tokens down into async reporting tasks.

#### Benefits Over Others / Trade-offs
- **Benefits**: Out-of-the-box class provided directly by `spring-security-core`.
- **Trade-offs**: Only propagates security; ignores MDC, correlation IDs, or custom tenant contexts.

#### Well-Documented Code Example

```java
package com.example.async.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.security.concurrent.DelegatingSecurityContextExecutor;

import java.util.concurrent.Executor;

@Configuration
public class DelegatingSecurityConfig {

    @Bean("securityAwareExecutor")
    public Executor securityAwareExecutor() {
        ThreadPoolTaskExecutor baseExecutor = new ThreadPoolTaskExecutor();
        baseExecutor.setCorePoolSize(5);
        baseExecutor.setMaxPoolSize(10);
        baseExecutor.initialize();

        // Automatically propagates SecurityContextHolder to all async workers
        return new DelegatingSecurityContextExecutor(baseExecutor);
    }
}
```

---

### Approach 4: Manual Context Capture, Transfer, and Eviction via Wrapper Closures

#### Scenario
Ad-hoc asynchronous calls using `CompletableFuture.runAsync(...)` where custom thread pool infrastructure or third-party executors cannot be modified.

#### Use-Case
Passing a correlation token into a one-off `CompletableFuture` invocation.

#### Benefits Over Others / Trade-offs
- **Benefits**: Requires zero framework configuration; works in any vanilla Java environment.
- **Trade-offs**: Highly verbose; developer error can lead to missed cleanup in `finally` block causing memory leaks.

#### Well-Documented Code Example

```java
package com.example.async.manual;

import org.slf4j.MDC;

import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;
import java.util.function.Supplier;

public final class AsyncContextUtils {

    private AsyncContextUtils() {}

    public static <U> CompletableFuture<U> supplyAsyncWithMdc(Supplier<U> supplier, Executor executor) {
        // 1. Capture on current thread
        final Map<String, String> callingContext = MDC.getCopyOfContextMap();

        return CompletableFuture.supplyAsync(() -> {
            // 2. Install on pool thread
            if (callingContext != null) {
                MDC.setContextMap(callingContext);
            }
            try {
                return supplier.get();
            } finally {
                // 3. Clear pool thread
                MDC.clear();
            }
        }, executor);
    }
}
```

---

### Context Propagation Comparison Matrix

| Feature | Approach 1: `TaskDecorator` | Approach 2: Micrometer | Approach 3: DelegatingSecurity | Approach 4: Manual Wrapper |
|---|---|---|---|---|
| **Scope of Context** | Any custom ThreadLocal + MDC + Security | Any registered accessor | SecurityContext only | Ad-hoc per invocation |
| **Ease of Setup** | Clean, centralized Spring bean | Requires library setup | Built-in Spring Security | Boilerplate per call |
| **Reactive Support** | No (Imperative only) | **Yes** (Reactor + Imperative) | No | No |
| **Recommended For** | **Spring Boot 3 WebMVC apps** | **Spring Boot 3 + Observability** | Security-only apps | Ad-hoc vanilla tasks |

---

## 6. Multi-Approach: Database-Level Concurrency Control

When multiple independent application nodes or threads update identical relational records, synchronization must occur at the persistence tier.

---

### Approach 1: Optimistic Locking with `@Version` & Spring Retry

#### Scenario
Read-heavy database workloads with low-to-moderate write contention where transaction rollbacks and retries are acceptable.

#### Use-Case
E-commerce product descriptions, profile updates, metadata catalogs.

#### Benefits Over Others / Trade-offs
- **Benefits**: Never acquires database locks; maximum concurrency and throughput; completely eliminates database deadlocks.
- **Trade-offs**: Throws `OptimisticLockingFailureException` when collision occurs; requires automatic retry logic; wasted CPU cycles if collision rates are high.

#### Well-Documented Code Example

```java
package com.example.concurrency.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private Integer availableStock;

    @Version // JPA automatically checks and increments this column on UPDATE
    private Long version;

    public Product() {}
    // Getters and setters omitted for brevity
}
```

```java
package com.example.concurrency.service;

import com.example.concurrency.entity.Product;
import com.example.concurrency.repository.ProductRepository;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OptimisticInventoryService {

    private final ProductRepository productRepository;

    public OptimisticInventoryService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    /**
     * Retries up to 3 times with exponential backoff on version collision.
     */
    @Retryable(
        retryFor = { OptimisticLockingFailureException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 50, multiplier = 2.0)
    )
    @Transactional
    public void deductStockWithOptimisticRetry(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new IllegalArgumentException("Product not found"));

        if (product.getAvailableStock() < quantity) {
            throw new IllegalStateException("Insufficient inventory");
        }

        product.setAvailableStock(product.getAvailableStock() - quantity);
        productRepository.save(product); 
        // Generates: UPDATE products SET stock = ?, version = 2 WHERE id = ? AND version = 1
    }
}
```

---

### Approach 2: Pessimistic Write Locking (`PESSIMISTIC_WRITE` / `SELECT ... FOR UPDATE`)

#### Scenario
High-contention financial or reservation domains where conflicts are frequent and optimistic retries would cause severe thrashing.

#### Use-Case
Concert seat reservations, flash sales, high-value bank ledger balance transfers.

#### Benefits Over Others / Trade-offs
- **Benefits**: Guaranteed serial execution at the database row level; zero collision retries required; eliminates double spending.
- **Trade-offs**: Other transactions block until the holding transaction commits; potential for database deadlocks if multiple tables are locked out-of-order; reduces database throughput.

#### Well-Documented Code Example

```java
package com.example.concurrency.repository;

import com.example.concurrency.entity.Product;
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface ProductLockRepository extends JpaRepository<Product, Long> {

    // Emits SQL: SELECT ... FROM products WHERE id = ? FOR UPDATE
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithPessimisticWriteLock(@Param("id") Long id);
}
```

```java
package com.example.concurrency.service;

import com.example.concurrency.entity.Product;
import com.example.concurrency.repository.ProductLockRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PessimisticInventoryService {

    private final ProductLockRepository productRepository;

    public PessimisticInventoryService(ProductLockRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional(timeout = 5) // Critical: enforce timeout to prevent indefinite lock blocking
    public void deductStockPessimistic(Long productId, int quantity) {
        // Row is locked immediately upon SELECT; subsequent threads block until this transaction commits
        Product product = productRepository.findByIdWithPessimisticWriteLock(productId)
            .orElseThrow(() -> new IllegalArgumentException("Product not found"));

        if (product.getAvailableStock() < quantity) {
            throw new IllegalStateException("Insufficient stock");
        }

        product.setAvailableStock(product.getAvailableStock() - quantity);
        productRepository.save(product);
    }
}
```

---

### Approach 3: Pessimistic Read Locking (`PESSIMISTIC_READ` / `SELECT ... FOR SHARE`)

#### Scenario
Workloads where a record must be read and guaranteed not to be modified or deleted by concurrent writers while generating a consistent report or dependent calculation, but concurrent readers are permitted.

#### Use-Case
End-of-day reconciliation summaries, auditing compliance snapshots.

#### Benefits Over Others / Trade-offs
- **Benefits**: Multiple readers acquire shared locks concurrently without blocking each other; completely blocks concurrent writers (`FOR UPDATE`).
- **Trade-offs**: If two concurrent readers both decide to upgrade their read lock to a write lock, an immediate deadlock occurs.

#### Well-Documented Code Example

```java
package com.example.concurrency.repository;

import com.example.concurrency.entity.Product;
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface ProductReadLockRepository extends JpaRepository<Product, Long> {

    // Emits SQL: SELECT ... FROM products WHERE id = ? FOR SHARE
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithPessimisticReadLock(@Param("id") Long id);
}
```

---

### Approach 4: Single-Statement Atomic Database Mutations (`UPDATE ... WHERE condition`)

#### Scenario
Simple numerical increments or decrements where fetching the entity into Java memory is unnecessary.

#### Use-Case
Inventory quantity reduction, user wallet balance debits, coupon usage counts.

#### Benefits Over Others / Trade-offs
- **Benefits**: **Highest possible performance**; relies on underlying database engine row locks for the duration of a single SQL statement; eliminates ORM dirty-checking overhead; zero application-level locks or retries.
- **Trade-offs**: Bypasses Hibernate in-memory dirty checking and first-level cache; does not trigger `@EntityListeners`.

#### Well-Documented Code Example

```java
package com.example.concurrency.repository;

import com.example.concurrency.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface AtomicProductRepository extends JpaRepository<Product, Long> {

    /**
     * Executes atomic decrement directly in the database engine.
     * Returns the count of affected rows: 1 = success, 0 = insufficient stock.
     */
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
        UPDATE Product p 
        SET p.availableStock = p.availableStock - :quantity 
        WHERE p.id = :id AND p.availableStock >= :quantity
    """)
    int deductStockAtomically(@Param("id") Long id, @Param("quantity") int quantity);
}
```

```java
package com.example.concurrency.service;

import com.example.concurrency.repository.AtomicProductRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class HighPerformanceAtomicInventoryService {

    private final AtomicProductRepository productRepository;

    public HighPerformanceAtomicInventoryService(AtomicProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional
    public void deductStock(Long productId, int quantity) {
        int updatedRows = productRepository.deductStockAtomically(productId, quantity);
        if (updatedRows == 0) {
            throw new IllegalStateException("Insufficient stock or product does not exist");
        }
    }
}
```

---

### Database Concurrency Comparison Matrix

| Strategy | Lock Mechanism | Collision Handling | Database Lock Duration | Throughput | Best For |
|---|---|---|---|---|---|
| **Approach 1: Optimistic** | `@Version` column check | Exception + Retry | None | **Highest** | Low contention, Read-heavy |
| **Approach 2: Pessimistic Write** | `FOR UPDATE` row lock | Serialized wait | Full transaction duration | Moderate | High contention, Financial |
| **Approach 3: Pessimistic Read** | `FOR SHARE` row lock | Serialized wait | Full transaction duration | Moderate | Consistent Read Snapshots |
| **Approach 4: Atomic SQL** | Engine row lock | Return updated row count | Single SQL statement | **Maximum** | Simple counter/stock deductions |

---

## 7. Multi-Approach: Distributed Locking & Multi-Node Synchronization

In a clustered microservice deployment running across multiple Kubernetes pods, in-memory Java synchronization (`synchronized`, `ReentrantLock`) fails because each pod runs an isolated JVM. Distributed synchronization guarantees cross-cluster mutual exclusion.

---

### Approach 1: Redis Distributed Lock with Redisson (Watchdog Auto-Renewal)

#### Scenario
High-throughput distributed microservices requiring fast lock acquisition and automatic renewal if a node experiences prolonged garbage collection pauses.

#### Use-Case
Distributed inventory checkout, scheduled background tasks, cross-service payment capture.

#### Benefits Over Others / Trade-offs
- **Benefits**: Ultra-fast (~1ms); built-in Redisson Watchdog automatically extends lock TTL while business logic executes; fails safe via automatic lease expiration if pod crashes.
- **Trade-offs**: Introduces operational dependency on Redis cluster; split-brain risk in Redis Sentinel/Cluster during network partitions.

#### Well-Documented Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.27.0</version>
</dependency>
```

```java
package com.example.distributed.lock;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class RedissonDistributedLockService {

    private static final Logger log = LoggerFactory.getLogger(RedissonDistributedLockService.class);
    private final RedissonClient redissonClient;

    public RedissonDistributedLockService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    public boolean executeWithLock(String lockKey, long waitTimeSeconds, Runnable criticalTask) {
        RLock lock = redissonClient.getLock("lock:" + lockKey);

        try {
            // Attempt to acquire: wait up to waitTimeSeconds.
            // If acquired without explicit leaseTime, Redisson Watchdog auto-renews lock every 10s!
            boolean acquired = lock.tryLock(waitTimeSeconds, TimeUnit.SECONDS);
            if (!acquired) {
                log.warn("Failed to acquire distributed lock for key: [{}]", lockKey);
                return false;
            }

            try {
                criticalTask.run();
                return true;
            } finally {
                // MANDATORY: Only unlock if currently held by THIS specific thread
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Interrupted during lock acquisition", e);
        }
    }
}
```

---

### Approach 2: Database Advisory Locks & ShedLock Coordination

#### Scenario
Applications already connected to a relational database (PostgreSQL/MySQL) that want distributed synchronization without introducing Redis infrastructure.

#### Use-Case
Ensuring `@Scheduled` jobs execute on exactly one node in a multi-pod cluster.

#### Benefits Over Others / Trade-offs
- **Benefits**: Zero new infrastructure needed; utilizes existing ACID database; ShedLock integrates seamlessly with Spring Boot scheduling.
- **Trade-offs**: Adds lock contention to primary database; lower lock acquisition throughput than Redis.

#### Well-Documented Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.12.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.12.0</version>
</dependency>
```

```java
package com.example.distributed.shedlock;

import net.javacrumbs.shedlock.core.LockProvider;
import net.javacrumbs.shedlock.provider.jdbctemplate.JdbcTemplateLockProvider;
import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock;
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class ShedLockConfiguration {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new org.springframework.jdbc.core.JdbcTemplate(dataSource))
                .usingDbTime() // Uses DB clock to avoid clock skew across pods
                .build()
        );
    }
}
```

```java
@Component
public class DistributedScheduledJobs {

    @Scheduled(cron = "0 0 * * * *") // Runs every hour
    @SchedulerLock(
        name = "HourlyBillingSyncJob", 
        lockAtLeastFor = "30s", // Prevents execution on another node if job finishes in 2s
        lockAtMostFor = "15m"   // Safety net if pod dies while executing
    )
    public void executeHourlyBilling() {
        // Runs on exactly ONE pod across the entire Kubernetes cluster
    }
}
```

---

### Approach 3: Distributed Idempotency Token Pattern with Redis & Unique DB Keys

#### Scenario
Payment webhooks, order submissions, and messaging queues where the goal is not to serialize operations, but to ensure that duplicate requests are detected and processed exactly once.

#### Use-Case
Stripe/PayPal webhook processing, checkout form double-click deduplication.

#### Benefits Over Others / Trade-offs
- **Benefits**: Completely prevents duplicate side effects; highly performant; decouples state verification from locking.
- **Trade-offs**: Requires client to supply a unique idempotency key (UUID); requires caching previous responses.

#### Well-Documented Code Example

```java
package com.example.distributed.idempotency;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class IdempotencyTokenService {

    private final StringRedisTemplate redisTemplate;

    public IdempotencyTokenService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * Atomically sets key only if absent (SET NX) with TTL.
     * Returns true if this request is unique and should proceed.
     */
    public boolean acquireIdempotencyToken(String idempotencyKey, Duration ttl) {
        String key = "idempotency:" + idempotencyKey;
        Boolean success = redisTemplate.opsForValue().setIfAbsent(key, "IN_PROGRESS", ttl);
        return Boolean.TRUE.equals(success);
    }

    public void markCompleted(String idempotencyKey, String resultPayload, Duration ttl) {
        String key = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(key, resultPayload, ttl);
    }
}
```

---

### Distributed Locking Comparison Matrix

| Approach | Underlying Store | Failure Recovery | Throughput | Operational Burden | Primary Use Case |
|---|---|---|---|---|---|
| **Approach 1: Redisson** | Redis | Watchdog / TTL Expiry | **Very High** | Moderate (Redis cluster) | Flash Sales, High-Throughput APIs |
| **Approach 2: ShedLock** | PostgreSQL / MySQL | TTL Expiry (`lockAtMostFor`) | Moderate | **Lowest** (Existing DB) | Clustered Scheduled Background Jobs |
| **Approach 3: Idempotency Key** | Redis / Unique Index | Key Expiry / DB Constraint | **High** | Low | Webhooks, Checkout API Deduplication |

---

## 8. Production Thread Pool Sizing & Schedulers

### 8.1 Little's Law, CPU-Bound vs. I/O-Bound Pool Sizing

Selecting thread pool dimensions without mathematical justification is a leading cause of production crashes.

$$\text{Threads}_{\text{CPU-bound}} = N_{\text{cores}} + 1$$

$$\text{Threads}_{\text{I/O-bound}} = N_{\text{cores}} \times \left(1 + \frac{\text{Wait Time}}{\text{Service Time}}\right)$$

Where:
- $N_{\text{cores}}$ is available logical CPU processors (`Runtime.getRuntime().availableProcessors()`).
- $\text{Wait Time}$ is latency spent waiting for external responses (database, microservice REST calls, disk).
- $\text{Service Time}$ is active CPU processing time (JSON parsing, business validations).

If a REST call takes $100\text{ms}$ total, of which $95\text{ms}$ is waiting for PostgreSQL and $5\text{ms}$ is CPU parsing, $\frac{W}{S} = \frac{95}{5} = 19$. On an 8-core CPU server:

$$\text{Threads} = 8 \times (1 + 19) = 160\text{ threads}$$

### 8.2 Saturation Policies (CallerRunsPolicy vs AbortPolicy)

When both the thread pool (`maxPoolSize`) and the task queue (`queueCapacity`) are 100% saturated:

```
ThreadPool Saturated?
       │
       ├─ AbortPolicy (Default) ──> Throws RejectedExecutionException (Drops task immediately)
       ├─ CallerRunsPolicy ───────> Calling thread executes the task inline (Natural Backpressure!)
       ├─ DiscardPolicy ──────────> Silently drops task with no notification
       └─ DiscardOldestPolicy ────> Drops the oldest pending task in queue to make space
```

> [!TIP]
> In Spring Boot web applications, **`CallerRunsPolicy` is the production standard**. It forces the incoming Tomcat HTTP servlet thread to execute the rejected async task itself. This naturally throttles incoming traffic, preventing out-of-memory errors.

### 8.3 Graceful Shutdown Protocol

During a Kubernetes rolling deployment, pods receive a `SIGTERM` signal. Without graceful shutdown configuration, in-flight asynchronous tasks are terminated abruptly:

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### 8.4 Micrometer Thread Pool Monitoring & Alerts

```java
package com.example.async.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
public class ThreadPoolMetricsBinderConfig {

    public ThreadPoolMetricsBinderConfig(ThreadPoolTaskExecutor productionTaskExecutor, MeterRegistry registry) {
        // Automatically registers:
        // executor.active, executor.completed, executor.queued, executor.pool.size
        new ExecutorServiceMetrics(productionTaskExecutor.getThreadPoolExecutor(), "productionTaskExecutor", java.util.List.of())
            .bindTo(registry);
    }
}
```

---

## 9. Common Spring Concurrency Traps & Anti-Patterns

### Trap 1: The `@Async` Self-Invocation Proxy Bypass
Calling an `@Async` method from another method within the same bean bypasses the Spring AOP dynamic proxy. The method executes synchronously on the calling thread.

### Trap 2: The `SimpleAsyncTaskExecutor` Production Disaster
Failing to specify a custom `TaskExecutor` causes Spring to fall back to `SimpleAsyncTaskExecutor`. This creates a brand new OS platform thread for every task, leading to `java.lang.OutOfMemoryError: unable to create new native thread`.

### Trap 3: Unbounded In-Memory Queues
Configuring an `Executor` with `Integer.MAX_VALUE` queue capacity causes the thread pool never to scale past `corePoolSize`. Under burst traffic, millions of tasks accumulate in heap memory, triggering an `OutOfMemoryError`.

### Trap 4: ThreadLocal Leaks in Recycled Tomcat Threads
Setting values in `ThreadLocal` or `MDC` without executing `.remove()` inside a mandatory `finally` block leaks security contexts and user identifiers across requests.

### Trap 5: Carrier Thread Pinning on Virtual Threads (Java 21)
Executing blocking operations inside a `synchronized` block on a Virtual Thread "pins" the virtual thread to its underlying OS carrier thread, disabling Project Loom's scalability benefits. Use `ReentrantLock` instead.

---

## 10. Deep Dive: JMM, Volatile, & Happens-Before Order

The **Java Memory Model (JMM)** defines the formal specification governing when mutations executed by one thread become visible to another thread.

### The Problem: CPU Caching & Instruction Reordering

Modern CPUs do not write directly to main memory (DRAM). Writes hit L1/L2/L3 hardware caches:

```
[Core 1] ──> [L1 Cache] ──┐
                          ├──> [L3 Cache] ──> [Main Memory (DRAM)]
[Core 2] ──> [L1 Cache] ──┘
```

Compilers and CPU out-of-order execution units frequently reorder bytecode instructions as long as single-threaded program output remains unchanged. In multithreaded systems, this causes visibility hazards.

### The Happens-Before Formal Guarantees

An action $A$ **happens-before** action $B$ ($A \prec B$) guarantees that all memory writes performed by $A$ are completely visible to $B$:

1. **Monitor Lock Rule**: An unlock on an intrinsic monitor or explicit `Lock` happens-before every subsequent lock on the same monitor.
2. **Volatile Variable Rule**: A write to a `volatile` variable happens-before every subsequent read of that exact variable.
3. **Thread Start Rule**: A call to `Thread.start()` happens-before any action in the started thread.
4. **Thread Termination Rule**: Any action in a thread happens-before any other thread successfully detects that it has terminated (`thread.join()`).
5. **Transitivity**: If $A \prec B$ and $B \prec C$, then $A \prec C$.

```java
// Safe Double-Checked Locking via Volatile Happens-Before Guarantee
public class SafeDoubleCheckedSingleton {
    // Volatile is mandatory: prevents CPU instruction reordering of (memory alloc -> constructor -> field assign)
    private static volatile SafeDoubleCheckedSingleton instance;

    private SafeDoubleCheckedSingleton() {}

    public static SafeDoubleCheckedSingleton getInstance() {
        if (instance == null) { // Check 1 (Unsynchronized)
            synchronized (SafeDoubleCheckedSingleton.class) {
                if (instance == null) { // Check 2 (Synchronized)
                    instance = new SafeDoubleCheckedSingleton();
                }
            }
        }
        return instance;
    }
}
```

---

## 11. Summary & Key Takeaways

```
                      SPRING BOOT CONCURRENCY LANDSCAPE
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         ▼                            ▼                            ▼
[In-Process JVM Layer]      [Asynchronous Layer]        [Distributed Layer]
- Default: Stateless        - Use ThreadPoolTaskExecutor- Clustered Locks: Redisson
- Scalable: LongAdder/CAS   - Reject: CallerRunsPolicy  - ShedLock: Scheduled Jobs
- In-Memory: ConcurrentMap  - Java 21: Virtual Threads  - Idempotency: Redis SET NX
- Monitors: Private lock    - Decorate: TaskDecorator   - DB: Optimistic / Pessimistic
```

1. **Stateless Default**: Keep Spring singleton beans strictly stateless. Store transient state in method-local stack frames.
2. **Never Pool Virtual Threads**: Platform threads require pooling (`ThreadPoolTaskExecutor`). Virtual threads are cheap and short-lived; spawn one per task.
3. **Always Decorate Async Executors**: Use `TaskDecorator` to propagate logging MDC trace IDs and `SecurityContext` across thread pool boundaries.
4. **Choose the Right DB Lock**: Default to Optimistic Locking (`@Version`). Switch to Pessimistic Write (`FOR UPDATE`) or Atomic SQL (`UPDATE WHERE`) for high-contention inventory and ledger mutations.
5. **Cluster Synchronization**: In multi-pod deployments, JVM-level `synchronized` keywords are ineffective. Use Redisson for distributed locks and ShedLock for scheduled tasks.

---

## 12. Practice Questions & Interview Scenarios

1. **The Phantom Lock**: A developer wraps a method inside a `@Service` bean with `public synchronized void process()`. The bean is annotated with `@Transactional`. Why does a race condition still occur under concurrent load?
   - *Answer*: `@Transactional` creates an AOP proxy. The caller locks on the proxy object, while the transaction commits *after* the synchronized method exits. Another thread enters between method exit and transaction commit.

2. **The Async OOM Trap**: Under load testing, an application throws `OutOfMemoryError: unable to create new native thread`. What configuration in `@EnableAsync` was omitted?
   - *Answer*: No custom `ThreadPoolTaskExecutor` bean was declared, causing Spring to use `SimpleAsyncTaskExecutor`, creating unbounded platform threads.

3. **Virtual Thread Pinning**: Why is using `synchronized` blocks inside Virtual Threads discouraged in Java 21? What alternative should be used?
   - *Answer*: It pins the virtual thread to its OS carrier thread, preventing the JVM from unmounting it during blocking I/O. Use `ReentrantLock` instead.

4. **Context Loss**: A user logs in, but an `@Async` method throws `NullPointerException` when calling `SecurityContextHolder.getContext().getAuthentication()`. Why, and how is it resolved?
   - *Answer*: `SecurityContextHolder` uses per-thread `ThreadLocal`. Fix it by configuring a `TaskDecorator` or using `DelegatingSecurityContextExecutor`.

5. **Optimistic vs Pessimistic**: A ticket booking system experiences 2,000 requests/second for 50 concert seats. Which locking strategy should be chosen and why?
   - *Answer*: Pessimistic write locking or atomic database decrement (`UPDATE seats SET booked = true WHERE id = ? AND booked = false`). Optimistic locking will cause 1,950 failed version collisions and thrash CPU with retries.
