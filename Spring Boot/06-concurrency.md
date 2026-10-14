# 06 — Concurrency & Race Conditions in Spring Boot

> This module is a production-grade, interview-focused deep dive into every layer of concurrency you will encounter in a Spring Boot application — from JVM threads and Java primitives all the way to distributed locks and reactive models.

---

## Table of Contents

1. [Java Concurrency Fundamentals (Spring Context)](#1-java-concurrency-fundamentals-spring-context)
2. [Race Conditions — Complete Explanation](#2-race-conditions--complete-explanation)
3. [Synchronization in Java](#3-synchronization-in-java)
4. [@Async Deep Dive](#4-async-deep-dive)
5. [Thread Pools in Spring Boot](#5-thread-pools-in-spring-boot)
6. [CompletableFuture](#6-completablefuture)
7. [Database-Level Concurrency](#7-database-level-concurrency)
8. [Distributed Concurrency](#8-distributed-concurrency)
9. [Common Concurrency Bugs in Spring](#9-common-concurrency-bugs-in-spring)
10. [WebFlux & Reactive — Brief Overview](#10-webflux--reactive--brief-overview)
11. [⚠️ Interview Traps & Gotchas](#️-interview-traps--gotchas)
12. [🔍 Deep Dive — JMM and Happens-Before](#-deep-dive--jmm-and-happens-before)
13. [📝 Summary & Key Takeaways](#-summary--key-takeaways)
14. [❓ Practice Questions](#-practice-questions)

---

## 1. Java Concurrency Fundamentals (Spring Context)

### 1.1 Threads and the JVM Thread Model

Every Java program runs in a JVM process. The JVM maps Java threads 1:1 to OS-level native threads (platform threads). Each thread has:

- **Program counter** — tracks which bytecode instruction to execute next
- **Stack** — stores stack frames (local variables, method arguments, return addresses) — **thread-private**
- **Heap** — shared among all threads; where objects live
- **Method Area / Metaspace** — class metadata, static fields — shared

```
JVM Process Memory Layout
┌──────────────────────────────────────────────────────────────┐
│                          HEAP (shared)                        │
│   ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │
│   │ Spring     │  │ Spring     │  │ Your singleton beans   │ │
│   │ AppContext │  │ BeanFactory│  │ (UserService, etc.)    │ │
│   └────────────┘  └────────────┘  └────────────────────────┘ │
├──────────────────────────────────────────────────────────────┤
│              PER-THREAD STACKS (thread-private)               │
│  Thread-1 Stack  │  Thread-2 Stack  │  Thread-N Stack        │
│  [frame: doWork] │  [frame: doWork] │  [frame: doWork]       │
│  [frame: run]    │  [frame: run]    │  [frame: run]          │
└──────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> Local variables (primitives and object references) live on the **stack** and are **thread-safe by default**. Objects on the **heap** are shared and require synchronization if mutated concurrently.

### 1.2 Spring Beans are Singletons — Why This Matters

Spring's default bean scope is **singleton**: one shared instance per `ApplicationContext`. Every HTTP request that reaches a `@RestController` is handled by a different thread from the servlet thread pool (e.g., Tomcat's), but ALL threads share the **same bean instance**.

```java
// This singleton is created ONCE, shared by ALL threads
@Service
public class CounterService {
    private int count = 0;          // ⚠️ SHARED MUTABLE STATE — DANGER!

    public void increment() {
        count++;                    // NOT atomic — race condition!
    }

    public int getCount() {
        return count;
    }
}
```

When 100 concurrent requests call `increment()`, you will lose updates because `count++` is three operations:
1. READ `count` from memory
2. ADD 1 to the value
3. WRITE the new value back

Two threads can both read `0`, both compute `1`, and both write `1` — losing one increment.

### 1.3 Stateless vs Stateful Beans

| Characteristic | Stateless Bean | Stateful Bean |
|---|---|---|
| Instance variables | Method params / injected dependencies only | Holds mutable data per-operation |
| Thread-safety | Naturally thread-safe | Requires synchronization |
| Spring scope | `singleton` (safe) | `prototype`, `request`, or `session` |
| Example | `UserService`, `OrderRepository` | Shopping cart, wizard state |

**Stateless service — perfectly safe as singleton:**

```java
@Service
public class UserService {
    private final UserRepository userRepository;   // immutable after injection
    private final PasswordEncoder passwordEncoder; // immutable, stateless

    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    // All state lives in method-local variables — thread-safe
    public User createUser(String username, String rawPassword) {
        String hashed = passwordEncoder.encode(rawPassword);
        return userRepository.save(new User(username, hashed));
    }
}
```

**Stateful bean — must use correct scope:**

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();

    public void addItem(CartItem item) { items.add(item); }
    public List<CartItem> getItems() { return Collections.unmodifiableList(items); }
}
```

### 1.4 ThreadLocal in Spring — RequestContextHolder & SecurityContextHolder

`ThreadLocal<T>` provides **per-thread storage** — each thread gets its own isolated copy of a variable. Spring uses `ThreadLocal` extensively to pass contextual data through the call stack without method parameter pollution.

```java
// Simplified view of how Spring stores request context
public class RequestContextHolder {
    // Each thread gets its own RequestAttributes
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
        new NamedThreadLocal<>("Request attributes");

    public static RequestAttributes getRequestAttributes() {
        return requestAttributesHolder.get();
    }

    public static void setRequestAttributes(RequestAttributes attributes) {
        requestAttributesHolder.set(attributes);
    }
}
```

**SecurityContextHolder — Spring Security:**

```java
// Spring Security stores authentication per-thread (default MODE_THREADLOCAL)
@Service
public class AuditService {

    public void logAction(String action) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth != null ? auth.getName() : "anonymous";
        // log username + action
        System.out.println("User: " + username + " performed: " + action);
    }
}
```

> [!WARNING]
> `ThreadLocal` values are **NOT propagated** to child threads spawned by `@Async` or `CompletableFuture.runAsync()`. This is a critical source of `NullPointerException` in async code. We cover this in the `@Async` section.

**Custom ThreadLocal — request correlation ID:**

```java
public final class CorrelationIdHolder {
    private static final ThreadLocal<String> HOLDER = new ThreadLocal<>();

    private CorrelationIdHolder() {}

    public static void set(String id)   { HOLDER.set(id); }
    public static String get()          { return HOLDER.get(); }
    public static void clear()          { HOLDER.remove(); } // ⚠️ ALWAYS call in finally!
}

@Component
public class CorrelationFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) req;
        String correlationId = Optional
            .ofNullable(httpReq.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());
        CorrelationIdHolder.set(correlationId);
        try {
            chain.doFilter(req, res);
        } finally {
            CorrelationIdHolder.clear(); // ⚠️ CRITICAL: prevents memory leaks in thread pools
        }
    }
}
```

> [!CAUTION]
> **Never forget to `ThreadLocal.remove()` in a `finally` block.** Thread pools reuse threads. Without cleanup, the next request on the same thread will see the previous request's ThreadLocal value — a severe data leak / security vulnerability.

---

## 2. Race Conditions — Complete Explanation

A **race condition** occurs when the program's correctness depends on the relative timing or interleaving of operations from multiple threads. The result is non-deterministic and depends on the thread scheduler.

### 2.1 Check-Then-Act Race Condition

```java
// ❌ BROKEN — classic check-then-act race condition
@Service
public class SeatBookingService {
    private final SeatRepository seatRepository;

    public void bookSeat(Long seatId, Long userId) {
        Seat seat = seatRepository.findById(seatId).orElseThrow();

        // Thread A checks here — seat is available
        // Thread B checks here — seat is ALSO available (A hasn't written yet)
        if (seat.isAvailable()) {
            // Thread A sets booked
            // Thread B ALSO sets booked — double booking!
            seat.setBookedBy(userId);
            seat.setAvailable(false);
            seatRepository.save(seat);
        }
    }
}
```

**Timeline of the bug:**

```
Thread A                    Thread B
READ seat (available=true)
                            READ seat (available=true)
CHECK: available? YES
                            CHECK: available? YES
seat.setBooked(userA)
seatRepository.save()
                            seat.setBooked(userB)
                            seatRepository.save()  ← OVERWRITES! Double booking!
```

### 2.2 Read-Modify-Write Race Condition

```java
// ❌ BROKEN — count++ is not atomic
@RestController
public class PageViewController {
    private long pageViews = 0L; // shared mutable state in a singleton!

    @GetMapping("/view")
    public long view() {
        pageViews++; // READ, INCREMENT, WRITE — not atomic
        return pageViews;
    }
}
```

Under the JVM, `pageViews++` compiles to roughly:
```
GETFIELD pageViews     // load from heap to thread's working memory
LCONST_1               // push 1
LADD                   // add
PUTFIELD pageViews     // write back to heap
```
Two threads executing this simultaneously can both read the same old value, both compute old+1, and write it twice — resulting in one lost increment.

### 2.3 Race Condition with Spring Singleton Beans

```java
// ❌ A realistic Spring bug: state held in a @Service singleton
@Service
public class ReportService {
    // ⚠️ Instance variable — shared across all threads!
    private List<String> currentReportLines = new ArrayList<>();

    public List<String> generateReport(List<Order> orders) {
        currentReportLines.clear(); // Thread A clears; Thread B's data gone!
        for (Order order : orders) {
            currentReportLines.add(formatOrder(order)); // concurrent add to non-thread-safe list
        }
        return new ArrayList<>(currentReportLines);
    }

    private String formatOrder(Order o) {
        return o.getId() + ": " + o.getTotal();
    }
}
```

**Fix — use local variables:**

```java
@Service
public class ReportService {
    // ✅ No instance state — all state is method-local (thread-safe)
    public List<String> generateReport(List<Order> orders) {
        List<String> lines = new ArrayList<>(); // local — lives on stack per thread
        for (Order order : orders) {
            lines.add(formatOrder(order));
        }
        return lines;
    }

    private String formatOrder(Order o) {
        return o.getId() + ": " + o.getTotal();
    }
}
```

### 2.4 Detecting Race Conditions

| Technique | How |
|---|---|
| **Code review** | Look for mutable instance fields in singleton beans |
| **Thread Sanitizer** | Available in native compilers; limited JVM support |
| **Java PathFinder** | NASA model checker for Java concurrency |
| **Load testing** | JMeter/Gatling with high concurrency uncovers symptoms |
| **Logging with thread name** | `[Thread-1] count=99, [Thread-1] count=99` — duplicate values |
| **AtomicInteger as oracle** | Compare AtomicInteger with plain int under same load |

---

## 3. Synchronization in Java

### 3.1 `synchronized` Keyword

`synchronized` ensures **mutual exclusion** (only one thread executes the block at a time) and **visibility** (changes made inside are visible to the next thread that acquires the lock).

**Method-level synchronization — locks on `this`:**

```java
@Service
public class SafeCounterService {
    private int count = 0;

    // Lock acquired on 'this' instance
    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

**Block-level synchronization — preferred for finer granularity:**

```java
@Service
public class InventoryService {
    private final Map<String, Integer> stock = new HashMap<>();
    private final Object stockLock = new Object(); // dedicated lock object

    public void updateStock(String productId, int delta) {
        synchronized (stockLock) {  // only locks what's needed
            stock.merge(productId, delta, Integer::sum);
        }
        // Other non-stock operations proceed concurrently
        notifyWarehouse(productId);
    }

    private void notifyWarehouse(String productId) {
        // ... HTTP call, doesn't need stockLock
    }
}
```

> [!WARNING]
> Synchronizing on `this` in a Spring bean can cause issues if the bean is accessed via a proxy (e.g., `@Transactional`). The proxy wraps the bean, and external callers lock on the **proxy object**, not `this` — the lock is ineffective. Always use a **private dedicated lock object** or use `java.util.concurrent` tools.

### 3.2 `volatile` Keyword — What It Does and Doesn't Do

`volatile` solves **visibility** but NOT **atomicity**.

```java
// volatile guarantees:
// 1. Writes are immediately flushed to main memory
// 2. Reads always come from main memory (not CPU cache)
// volatile does NOT guarantee compound operations are atomic

public class ConfigManager {
    private volatile boolean featureEnabled = false; // safe for single write, multiple reads

    public void enable()  { featureEnabled = true;  } // atomic single write — safe
    public void disable() { featureEnabled = false; } // atomic single write — safe
    public boolean isEnabled() { return featureEnabled; } // safe read

    // ❌ NOT safe — read-modify-write is still not atomic
    private volatile int counter = 0;
    public void increment() { counter++; } // STILL a race condition!
}
```

| | `volatile` | `synchronized` | `AtomicInteger` |
|---|---|---|---|
| Visibility guarantee | ✅ | ✅ | ✅ |
| Atomicity (compound ops) | ❌ | ✅ | ✅ (specific ops) |
| Mutual exclusion | ❌ | ✅ | ❌ (lock-free) |
| Performance | Fast | Moderate (blocking) | Fast (CAS) |

### 3.3 `java.util.concurrent` Overview

The `java.util.concurrent` (JUC) package provides high-level, production-grade concurrency utilities:

```
java.util.concurrent
├── Executors & Thread Pools  (ExecutorService, ThreadPoolExecutor, ScheduledExecutorService)
├── Synchronizers             (CountDownLatch, CyclicBarrier, Semaphore, Phaser, Exchanger)
├── Atomic Variables          (AtomicInteger, AtomicLong, AtomicReference, LongAdder)
├── Locks                     (ReentrantLock, ReadWriteLock, StampedLock)
├── Concurrent Collections    (ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue variants)
└── Futures                   (Future, CompletableFuture, ForkJoinPool)
```

### 3.4 ReentrantLock and ReadWriteLock

```java
@Service
public class CacheService {
    private final Map<String, String> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple readers concurrently — no blocking between them
    public String get(String key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock(); // ALWAYS unlock in finally!
        }
    }

    // Exclusive write — blocks all readers and other writers
    public void put(String key, String value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

**ReentrantLock advantages over `synchronized`:**

```java
@Service
public class AdvancedLockService {
    private final ReentrantLock lock = new ReentrantLock(true); // fair lock

    public boolean tryUpdate(String data, long timeoutMs) throws InterruptedException {
        // tryLock — non-blocking attempt with timeout
        if (lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS)) {
            try {
                performUpdate(data);
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false; // couldn't acquire lock in time
    }

    private void performUpdate(String data) { /* ... */ }
}
```

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Fairness | No | Optional (`new ReentrantLock(true)`) |
| Timed lock attempt | No | `tryLock(timeout, unit)` |
| Interruptible lock | No | `lockInterruptibly()` |
| Multiple conditions | One (wait/notify) | `newCondition()` — unlimited |
| Lock info / debugging | No | `isLocked()`, `getQueueLength()` |

### 3.5 Atomic Variables

Atomic variables use **Compare-And-Swap (CAS)** — a hardware instruction that atomically checks a memory location and updates it only if it hasn't changed:

```java
@Service
public class MetricsService {
    private final AtomicInteger requestCount  = new AtomicInteger(0);
    private final AtomicLong    totalLatency  = new AtomicLong(0L);
    private final LongAdder     errorCount    = new LongAdder(); // better under high contention

    public void recordRequest(long latencyMs) {
        requestCount.incrementAndGet();       // atomic increment, returns new value
        totalLatency.addAndGet(latencyMs);    // atomic add
    }

    public void recordError() {
        errorCount.increment(); // LongAdder: uses striped cells — less contention than AtomicLong
    }

    public double getAverageLatency() {
        int count = requestCount.get();
        return count == 0 ? 0 : (double) totalLatency.get() / count;
    }

    // CAS example — set only if current value matches expected
    private final AtomicReference<String> status = new AtomicReference<>("IDLE");

    public boolean transitionToRunning() {
        return status.compareAndSet("IDLE", "RUNNING"); // atomic check-and-set
    }
}
```

> [!TIP]
> Use `LongAdder` instead of `AtomicLong` for **high-contention counters** (e.g., request counters). `LongAdder` maintains an array of cells and reduces CAS contention. Call `.sum()` to get the total. Throughput can be **10x better** under high load.

### 3.6 ConcurrentHashMap vs HashMap vs Collections.synchronizedMap

```java
@Service
public class SessionStore {

    // ❌ HashMap — NOT thread-safe. Can cause infinite loops in Java 7,
    //    data corruption in Java 8+
    private final Map<String, Session> bad = new HashMap<>();

    // ⚠️ Collections.synchronizedMap — wraps entire map in synchronized.
    //    Every operation (get, put, size, iterator) locks the entire map.
    //    Iterating requires manual synchronization!
    private final Map<String, Session> coarse = Collections.synchronizedMap(new HashMap<>());

    // ✅ ConcurrentHashMap — segment-level locking (Java 7) / CAS + synchronized per bucket (Java 8+)
    //    Reads are lock-free. Writes lock only the affected bucket.
    //    Null keys/values NOT allowed.
    private final Map<String, Session> best = new ConcurrentHashMap<>();

    public void store(String token, Session session) {
        best.put(token, session);
    }

    // ✅ Atomic compound operation — putIfAbsent, computeIfAbsent, merge
    public Session getOrCreate(String token) {
        return best.computeIfAbsent(token, k -> new Session(k)); // atomic!
    }

    public void updateExpiry(String token, Instant newExpiry) {
        // compute is atomic — entire lambda runs under lock for the key
        best.compute(token, (k, existing) -> {
            if (existing == null) return null;
            existing.setExpiry(newExpiry);
            return existing;
        });
    }
}
```

| | `HashMap` | `synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|---|
| Thread-safe | ❌ | ✅ (coarse lock) | ✅ (fine-grained) |
| Read concurrency | ❌ | ❌ (blocks) | ✅ (lock-free reads) |
| Null keys/values | ✅ | ✅ | ❌ |
| Iteration | Fail-fast | Must sync externally | Weakly consistent |
| Performance (high concurrency) | N/A | Poor | Excellent |

---

## 4. @Async Deep Dive

### 4.1 @EnableAsync — What It Does

`@EnableAsync` triggers a **BeanPostProcessor** (`AsyncAnnotationBeanPostProcessor`) that wraps `@Async`-annotated beans in an AOP proxy. When an async method is called, the proxy submits the method invocation to a `TaskExecutor` rather than executing it inline.

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    // Without @EnableAsync, @Async annotations are silently ignored!
}
```

### 4.2 @Async Method Requirements

```java
@Service
public class EmailService {

    // ✅ Correct — public method, returns CompletableFuture
    @Async
    public CompletableFuture<Void> sendWelcomeEmail(String to) {
        // ... send email
        return CompletableFuture.completedFuture(null);
    }

    // ❌ TRAP 1: private methods — proxy cannot intercept
    @Async
    private void sendPrivateEmail(String to) { /* silently runs synchronously! */ }

    // ❌ TRAP 2: self-invocation — calling @Async from the same bean bypasses proxy
    public void process() {
        sendWelcomeEmail("user@example.com"); // ← calls THIS.sendWelcomeEmail
        // → calls the real method directly, NOT via proxy → runs SYNCHRONOUSLY!
    }

    // ❌ TRAP 3: @Async on a @Bean-less class (not managed by Spring)
    // Annotation is ignored if Spring doesn't manage the bean
}
```

> [!IMPORTANT]
> **Self-invocation is the #1 `@Async` gotcha.** If method `A()` and `@Async B()` are in the same class and `A()` calls `B()`, it calls the target object directly, bypassing the AOP proxy. The async behavior is silently lost. Fix: inject the bean into itself (Spring 4.3+) or move the async method to a separate bean.

**Fix for self-invocation:**

```java
@Service
public class NotificationService {
    @Autowired
    private NotificationService self; // inject own proxy!

    public void notifyAll(List<String> emails) {
        emails.forEach(email -> self.sendAsync(email)); // goes through proxy → truly async
    }

    @Async
    public CompletableFuture<Void> sendAsync(String email) {
        // async work
        return CompletableFuture.completedFuture(null);
    }
}
```

### 4.3 Default Thread Pool — The SimpleAsyncTaskExecutor Trap

> [!CAUTION]
> **Critical production trap:** If you do NOT configure a custom `TaskExecutor`, Spring uses `SimpleAsyncTaskExecutor` by default. This executor **creates a new thread for every single async call**. There is **no thread pool, no queue, no backpressure**. Under load, your application will create thousands of threads and OOM.

```java
// What SimpleAsyncTaskExecutor effectively does internally:
public class SimpleAsyncTaskExecutor implements AsyncTaskExecutor {
    @Override
    public void execute(Runnable task) {
        Thread thread = new Thread(task); // NEW THREAD EVERY TIME — no pooling!
        thread.start();
    }
}
```

**Proof it's a trap:**

```java
@RestController
public class DemoController {
    @Autowired EmailService emailService;

    @GetMapping("/send-bulk")
    public String sendBulk() throws Exception {
        for (int i = 0; i < 10_000; i++) {
            emailService.sendEmail("user" + i + "@example.com");
            // With SimpleAsyncTaskExecutor: 10,000 threads created!
            // Default OS thread stack: ~512KB - 1MB each
            // → 10GB of memory just for thread stacks → OutOfMemoryError
        }
        return "sent";
    }
}
```

### 4.4 Configuring a Proper ThreadPoolTaskExecutor

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // Core pool: always-alive threads (like a standing workforce)
        executor.setCorePoolSize(10);

        // Max pool: upper thread limit (only created when queue is full)
        executor.setMaxPoolSize(50);

        // Queue: holds tasks when all core threads are busy
        executor.setQueueCapacity(200);

        // Thread name prefix for debugging/monitoring
        executor.setThreadNamePrefix("async-worker-");

        // What to do when maxPoolSize AND queue are both full
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

        // Wait for queued tasks to finish before shutdown
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);

        executor.initialize();
        return executor;
    }

    // This makes it the DEFAULT executor for all @Async methods
    @Override
    public Executor getAsyncExecutor() {
        return taskExecutor();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}
```

**Multiple named executors for different workloads:**

```java
@Configuration
@EnableAsync
public class MultiExecutorConfig {

    @Bean("emailExecutor")
    public ThreadPoolTaskExecutor emailExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(5);
        exec.setMaxPoolSize(20);
        exec.setQueueCapacity(100);
        exec.setThreadNamePrefix("email-");
        exec.initialize();
        return exec;
    }

    @Bean("reportExecutor")
    public ThreadPoolTaskExecutor reportExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(2);  // reports are heavy, fewer threads
        exec.setMaxPoolSize(5);
        exec.setQueueCapacity(10);
        exec.setThreadNamePrefix("report-");
        exec.initialize();
        return exec;
    }
}

@Service
public class NotificationService {

    @Async("emailExecutor")  // uses specific named executor
    public CompletableFuture<Void> sendEmail(String to) { /* ... */ return CompletableFuture.completedFuture(null); }

    @Async("reportExecutor") // uses different pool
    public CompletableFuture<byte[]> generateReport(Long reportId) { /* ... */ return CompletableFuture.completedFuture(new byte[0]); }
}
```

### 4.5 Return Types for @Async

```java
@Service
public class AsyncReturnTypesDemo {

    // 1. void — fire-and-forget. Caller cannot observe result or exceptions.
    @Async
    public void fireAndForget(String data) {
        process(data);
        // Any exception here is caught by AsyncUncaughtExceptionHandler
    }

    // 2. Future<T> — old-style, blocking get()
    @Async
    public Future<String> legacyAsync(String input) {
        String result = process(input);
        return new AsyncResult<>(result); // Spring's Future wrapper
    }

    // 3. CompletableFuture<T> — modern, composable, non-blocking
    @Async
    public CompletableFuture<String> modernAsync(String input) {
        try {
            String result = process(input);
            return CompletableFuture.completedFuture(result);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }

    // 4. ListenableFuture — Spring's old callback-based future (deprecated in 6.x)
    // Avoid in new code. Use CompletableFuture instead.

    private String process(String input) { return input.toUpperCase(); }
}
```

### 4.6 Exception Handling in @Async

```java
// For void return type — implement AsyncUncaughtExceptionHandler
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomAsyncExceptionHandler.class);

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error("Async exception in method '{}' with params {}: {}",
            method.getName(), Arrays.toString(params), ex.getMessage(), ex);
        // optionally: send alert, write to dead-letter queue, etc.
    }
}

// For CompletableFuture — handle in the future chain
@Service
public class RobustAsyncService {

    @Async
    public CompletableFuture<String> riskyOperation(String input) {
        if (input == null) {
            return CompletableFuture.failedFuture(new IllegalArgumentException("input is null"));
        }
        return CompletableFuture.completedFuture(input.toUpperCase());
    }
}

// In the caller:
@RestController
public class DemoController {
    @Autowired RobustAsyncService asyncService;

    @GetMapping("/async")
    public CompletableFuture<ResponseEntity<String>> handleAsync(@RequestParam String input) {
        return asyncService.riskyOperation(input)
            .thenApply(result -> ResponseEntity.ok(result))
            .exceptionally(ex -> ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Error: " + ex.getMessage()));
    }
}
```

### 4.7 @Async with @Transactional

```java
@Service
public class OrderService {

    // ✅ Each @Async invocation runs in its OWN transaction
    // The transaction starts on the async thread's context — not the caller's
    @Async
    @Transactional
    public CompletableFuture<Void> processOrderAsync(Long orderId) {
        // This runs in a NEW transaction on a new thread
        // The caller's transaction (if any) is NOT joined
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(OrderStatus.PROCESSING);
        orderRepository.save(order);
        return CompletableFuture.completedFuture(null);
    }

    // ❌ TRAP: Do NOT assume @Async inherits the caller's transaction
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        // This async call runs on a DIFFERENT thread → DIFFERENT transaction
        // If the async work fails, the caller's transaction may already be committed
        processOrderAsync(order.getId()); // caller's tx and async tx are independent!
    }
}
```

### 4.8 Async Context Propagation (Security, Request Context)

```java
// ❌ Problem: SecurityContext is stored in ThreadLocal — not propagated to async threads
@Service
public class BrokenAsyncService {
    @Async
    public CompletableFuture<String> brokenAsync() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        // auth is NULL here! ThreadLocal was not propagated to this new thread
        return CompletableFuture.completedFuture(auth.getName()); // NullPointerException!
    }
}

// ✅ Fix 1: Configure SecurityContextHolder to use InheritableThreadLocal mode
// In your main class or config:
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
// ⚠️ This works for directly spawned threads but NOT for thread pool reuse!

// ✅ Fix 2: Use Spring Security's DelegatingSecurityContextExecutor
@Configuration
@EnableAsync
public class SecureAsyncConfig implements AsyncConfigurer {

    @Bean
    public ThreadPoolTaskExecutor baseExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(10);
        exec.setMaxPoolSize(30);
        exec.setQueueCapacity(100);
        exec.initialize();
        return exec;
    }

    @Override
    public Executor getAsyncExecutor() {
        // Wraps the executor to propagate SecurityContext to async threads
        return new DelegatingSecurityContextExecutor(baseExecutor(),
            SecurityContextHolder.getContext());
    }
}

// ✅ Fix 3: Manually capture and set context
@Service
public class FixedAsyncService {
    @Autowired
    @Qualifier("taskExecutor")
    private Executor executor;

    public CompletableFuture<String> asyncWithContext() {
        // Capture context on calling thread
        SecurityContext context = SecurityContextHolder.getContext();

        return CompletableFuture.supplyAsync(() -> {
            SecurityContextHolder.setContext(context); // set on async thread
            try {
                return SecurityContextHolder.getContext().getAuthentication().getName();
            } finally {
                SecurityContextHolder.clearContext(); // cleanup!
            }
        }, executor);
    }
}
```

---

## 5. Thread Pools in Spring Boot

### 5.1 ThreadPoolTaskExecutor — Configuration Deep Dive

The thread pool lifecycle follows this exact sequence under load:

```
Task submitted
    │
    ▼
Core threads busy?
  ├─ NO  → Assign to idle core thread
  └─ YES → Is queue full?
               ├─ NO  → Add to queue (wait for a core thread)
               └─ YES → Can create new thread (< maxPoolSize)?
                            ├─ YES → Spawn new (non-core) thread
                            └─ NO  → RejectedExecutionHandler fires!
```

> [!IMPORTANT]
> **The queue fills BEFORE max threads are created.** This surprises most developers. If you set `corePoolSize=10`, `maxPoolSize=50`, `queueCapacity=200`, then the 51st task to arrive while all 10 core threads are busy goes into the queue — it does NOT immediately spawn thread #11. Thread #11 is only created after the queue has 200 items AND a new task arrives.

```java
@Configuration
public class ThreadPoolConfig {

    @Bean("ioExecutor")
    public ThreadPoolTaskExecutor ioExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();

        // For I/O-bound tasks: more threads than CPU cores
        int cpuCores = Runtime.getRuntime().availableProcessors();
        exec.setCorePoolSize(cpuCores * 4);    // e.g., 32 for 8-core machine
        exec.setMaxPoolSize(cpuCores * 8);     // e.g., 64
        exec.setQueueCapacity(500);

        exec.setKeepAliveSeconds(60);          // non-core threads die after 60s idle
        exec.setAllowCoreThreadTimeOut(true);  // core threads also time out (good for elasticity)

        exec.setThreadNamePrefix("io-worker-");
        exec.initialize();
        return exec;
    }

    @Bean("cpuExecutor")
    public ThreadPoolTaskExecutor cpuExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();

        // For CPU-bound tasks: no more threads than cores (avoid context switching)
        int cpuCores = Runtime.getRuntime().availableProcessors();
        exec.setCorePoolSize(cpuCores);
        exec.setMaxPoolSize(cpuCores);
        exec.setQueueCapacity(1000);           // large queue — tasks wait, don't context-switch

        exec.setThreadNamePrefix("cpu-worker-");
        exec.initialize();
        return exec;
    }
}
```

### 5.2 RejectedExecutionHandler Strategies

```java
// Strategy 1: AbortPolicy (DEFAULT) — throws RejectedExecutionException
// Strategy 2: CallerRunsPolicy — caller thread runs the task (backpressure!)
// Strategy 3: DiscardPolicy — silently drops the task
// Strategy 4: DiscardOldestPolicy — drops the oldest queued task

@Bean
public ThreadPoolTaskExecutor resilientExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(10);
    exec.setMaxPoolSize(20);
    exec.setQueueCapacity(100);

    // CallerRunsPolicy is best for most Spring Boot apps:
    // - Provides natural backpressure (slows down the caller)
    // - No tasks are lost
    // - Works well with HTTP request handlers (slows inbound request)
    exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

    // Custom handler — for monitoring and alerting
    exec.setRejectedExecutionHandler((task, executor2) -> {
        log.error("Task rejected! Queue size: {}, Active threads: {}",
            executor2.getQueue().size(), executor2.getActiveCount());
        // Metrics counter, alerting, etc.
        throw new RejectedExecutionException("Thread pool saturated");
    });

    exec.initialize();
    return exec;
}
```

### 5.3 Graceful Shutdown

```java
@Configuration
@EnableAsync
public class GracefulShutdownConfig implements AsyncConfigurer {

    @Bean(name = "taskExecutor", destroyMethod = "shutdown")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(10);
        exec.setMaxPoolSize(50);
        exec.setQueueCapacity(200);

        // ✅ Graceful shutdown configuration
        exec.setWaitForTasksToCompleteOnShutdown(true);   // Don't kill running tasks
        exec.setAwaitTerminationSeconds(60);               // Wait up to 60s for completion

        exec.initialize();
        return exec;
    }
}
```

In `application.yml`:
```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 60s  # Spring Boot 2.3+ graceful shutdown
server:
  shutdown: graceful
```

### 5.4 Monitoring Thread Pools

```java
@Component
public class ThreadPoolMonitor {

    @Autowired
    @Qualifier("taskExecutor")
    private ThreadPoolTaskExecutor taskExecutor;

    @Scheduled(fixedDelay = 30_000)
    public void logThreadPoolStats() {
        ThreadPoolExecutor pool = taskExecutor.getThreadPoolExecutor();
        log.info("""
            Thread Pool Stats:
              Pool size (current/core/max): {}/{}/{}
              Active threads: {}
              Queue size: {}/{}
              Completed tasks: {}
              """,
            pool.getPoolSize(),
            pool.getCorePoolSize(),
            pool.getMaximumPoolSize(),
            pool.getActiveCount(),
            pool.getQueue().size(),
            taskExecutor.getQueueCapacity(),
            pool.getCompletedTaskCount()
        );
    }
}
```

**Expose via Actuator:**

```java
@Component
public class ThreadPoolMetrics implements MeterBinder {

    @Autowired
    @Qualifier("taskExecutor")
    private ThreadPoolTaskExecutor executor;

    @Override
    public void bindTo(MeterRegistry registry) {
        ThreadPoolExecutor pool = executor.getThreadPoolExecutor();

        Gauge.builder("thread.pool.active", pool, ThreadPoolExecutor::getActiveCount)
             .tag("pool", "taskExecutor")
             .register(registry);

        Gauge.builder("thread.pool.queue.size", pool, e -> e.getQueue().size())
             .tag("pool", "taskExecutor")
             .register(registry);

        Gauge.builder("thread.pool.size", pool, ThreadPoolExecutor::getPoolSize)
             .tag("pool", "taskExecutor")
             .register(registry);
    }
}
```

### 5.5 Virtual Threads in Java 21 / Spring Boot 3.2

Java 21 introduced **Virtual Threads** (Project Loom) — lightweight threads managed by the JVM, not the OS. Spring Boot 3.2+ can use them natively.

```yaml
# application.yml — enable virtual threads for Tomcat
spring:
  threads:
    virtual:
      enabled: true
```

```java
// Or configure programmatically:
@Configuration
public class VirtualThreadConfig {

    // Spring Boot 3.2+ auto-configures this when spring.threads.virtual.enabled=true
    @Bean
    public TomcatProtocolHandlerCustomizer<?> virtualThreadCustomizer() {
        return protocolHandler ->
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    }

    // Virtual thread executor for @Async
    @Bean
    public AsyncTaskExecutor applicationTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

| | Platform Threads | Virtual Threads |
|---|---|---|
| Managed by | OS | JVM |
| Stack size | 512KB–1MB | ~1KB (heap-backed) |
| Count limit | ~thousands | Millions |
| Blocking behavior | OS thread blocked | JVM parks virtual thread, platform thread free |
| Context switch cost | High (OS) | Low (JVM scheduler) |
| Best for | CPU-bound tasks | I/O-bound tasks |

> [!NOTE]
> Virtual threads shine for **I/O-bound** workloads (database calls, HTTP calls, file I/O). For CPU-bound work, they offer no advantage over platform threads. Do NOT pool virtual threads — create one per task and let the JVM handle it.

---

## 6. CompletableFuture

### 6.1 Core Transformation Methods

```java
@Service
public class OrderPipelineService {

    // thenApply — transform result (sync, on same thread)
    public CompletableFuture<OrderDto> fetchAndTransform(Long orderId) {
        return orderRepo.findByIdAsync(orderId)      // CompletableFuture<Order>
            .thenApply(order -> toDto(order));       // Order → OrderDto (sync transform)
    }

    // thenCompose — flat-map (async → async, avoids CompletableFuture<CompletableFuture<T>>)
    public CompletableFuture<Invoice> fetchOrderAndGenerateInvoice(Long orderId) {
        return orderRepo.findByIdAsync(orderId)
            .thenCompose(order -> invoiceService.generateAsync(order)); // returns CF<Invoice>
    }

    // thenCombine — combine two independent async results
    public CompletableFuture<OrderSummary> buildOrderSummary(Long orderId) {
        CompletableFuture<Order>    orderFuture    = orderRepo.findByIdAsync(orderId);
        CompletableFuture<Customer> customerFuture = customerRepo.findByOrderAsync(orderId);

        // Both run concurrently! Combine when both complete.
        return orderFuture.thenCombine(customerFuture,
            (order, customer) -> new OrderSummary(order, customer));
    }

    // thenAccept — consume result (returns CompletableFuture<Void>)
    public CompletableFuture<Void> processAndLog(Long orderId) {
        return orderRepo.findByIdAsync(orderId)
            .thenAccept(order -> log.info("Processed: {}", order.getId()));
    }
}
```

### 6.2 allOf and anyOf

```java
@Service
public class NotificationDispatcher {

    // allOf — wait for ALL futures to complete
    public CompletableFuture<Void> sendToAllChannels(Notification notification) {
        CompletableFuture<Void> email = emailService.sendAsync(notification);
        CompletableFuture<Void> sms   = smsService.sendAsync(notification);
        CompletableFuture<Void> push  = pushService.sendAsync(notification);

        return CompletableFuture.allOf(email, sms, push); // completes when all 3 complete
    }

    // Collecting results from allOf (it returns Void, need workaround):
    public CompletableFuture<List<String>> fetchAllNames(List<Long> userIds) {
        List<CompletableFuture<String>> futures = userIds.stream()
            .map(id -> userService.fetchNameAsync(id))
            .collect(Collectors.toList());

        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join) // safe here — all completed
                .collect(Collectors.toList()));
    }

    // anyOf — complete as soon as the FIRST future completes
    public CompletableFuture<Object> fetchFromFastestSource(String key) {
        CompletableFuture<Object> primary   = primaryCache.getAsync(key);
        CompletableFuture<Object> secondary = secondaryCache.getAsync(key);
        CompletableFuture<Object> database  = databaseService.fetchAsync(key);

        // Returns the result from whichever finishes first
        return CompletableFuture.anyOf(primary, secondary, database);
    }
}
```

### 6.3 Exception Handling

```java
@Service
public class ResilientService {

    // exceptionally — handle failure, provide fallback (like catch block)
    public CompletableFuture<String> withFallback(String key) {
        return riskyService.fetchAsync(key)
            .exceptionally(ex -> {
                log.warn("Fetch failed for {}: {}", key, ex.getMessage());
                return "DEFAULT_VALUE"; // fallback value
            });
    }

    // handle — always called (success OR failure), can transform both
    public CompletableFuture<ApiResponse<String>> withWrappedResponse(String key) {
        return riskyService.fetchAsync(key)
            .handle((result, ex) -> {
                if (ex != null) {
                    return ApiResponse.error(ex.getMessage());
                }
                return ApiResponse.success(result);
            });
    }

    // whenComplete — side effect only, does NOT transform result
    public CompletableFuture<String> withAudit(String key) {
        return riskyService.fetchAsync(key)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    auditLog.recordFailure(key, ex);
                } else {
                    auditLog.recordSuccess(key, result);
                }
                // result still propagates (success or failure) to next stage
            });
    }
}
```

### 6.4 Running with Custom Executor

```java
@Service
public class ParallelProductService {

    @Autowired
    @Qualifier("ioExecutor")
    private Executor ioExecutor;

    public CompletableFuture<ProductDetails> fetchProductDetails(Long productId) {
        // All operations run on our custom ioExecutor, not ForkJoinPool.commonPool()
        CompletableFuture<Product>     product  = CompletableFuture.supplyAsync(
            () -> productRepo.findById(productId).orElseThrow(), ioExecutor);

        CompletableFuture<List<Review>> reviews = CompletableFuture.supplyAsync(
            () -> reviewRepo.findByProductId(productId), ioExecutor);

        CompletableFuture<Integer>     stock    = CompletableFuture.supplyAsync(
            () -> inventoryService.getStock(productId), ioExecutor);

        return CompletableFuture.allOf(product, reviews, stock)
            .thenApplyAsync(v -> new ProductDetails(
                product.join(),
                reviews.join(),
                stock.join()
            ), ioExecutor);
    }
}
```

> [!WARNING]
> Avoid `ForkJoinPool.commonPool()` (used when no executor is specified) in Spring Boot web applications. It's a shared pool with limited size (`Runtime.availableProcessors() - 1` threads). I/O blocking on commonPool can starve other app features using it (e.g., parallel streams).

### 6.5 CompletableFuture in Spring MVC Controllers

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @Autowired ProductService productService;

    // Spring MVC handles CompletableFuture return types natively
    // The servlet thread is released while the future completes
    @GetMapping("/{id}/details")
    public CompletableFuture<ResponseEntity<ProductDetails>> getDetails(@PathVariable Long id) {
        return productService.fetchProductDetails(id)
            .thenApply(details -> ResponseEntity.ok(details))
            .exceptionally(ex -> ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .<ProductDetails>build());
    }

    // DeferredResult — for more control over timeout and error handling
    @GetMapping("/{id}/slow")
    public DeferredResult<ResponseEntity<Product>> getSlowProduct(@PathVariable Long id) {
        DeferredResult<ResponseEntity<Product>> result = new DeferredResult<>(5000L); // 5s timeout
        result.onTimeout(() -> result.setErrorResult(
            ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT).build()));

        productService.fetchSlowly(id)
            .thenAccept(p -> result.setResult(ResponseEntity.ok(p)))
            .exceptionally(ex -> { result.setErrorResult(ex); return null; });

        return result;
    }
}
```

---

## 7. Database-Level Concurrency

### 7.1 Optimistic Locking with @Version

Optimistic locking assumes conflicts are rare. It does NOT hold a database lock — instead, it detects conflicts at commit time using a version column.

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stock;

    @Version                     // JPA manages this column automatically
    private Long version;        // incremented on every UPDATE

    // getters/setters
}
```

**Generated SQL:**
```sql
-- SELECT: fetches version (e.g., version=5)
SELECT * FROM products WHERE id = 1;

-- UPDATE: checks version matches what we read
UPDATE products SET stock = 99, version = 6 WHERE id = 1 AND version = 5;
-- If another thread updated between our read and write,
-- version in DB is now 6 (not 5), so 0 rows updated → JPA throws OptimisticLockException
```

```java
@Service
public class StockService {

    @Transactional
    public void decrementStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        if (product.getStock() < quantity) {
            throw new InsufficientStockException("Not enough stock");
        }
        product.setStock(product.getStock() - quantity);
        productRepository.save(product); // may throw OptimisticLockException
    }

    // With retry for optimistic lock conflicts:
    @Transactional
    @Retryable(value = ObjectOptimisticLockingFailureException.class,
               maxAttempts = 3,
               backoff = @Backoff(delay = 100, multiplier = 2))
    public void decrementStockWithRetry(Long productId, int quantity) {
        decrementStock(productId, quantity);
    }
}
```

### 7.2 Pessimistic Locking

Pessimistic locking holds a database row lock for the duration of the transaction — suitable when conflicts are frequent.

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // SELECT ... FOR UPDATE — exclusive lock
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithWriteLock(@Param("id") Long id);

    // SELECT ... FOR SHARE — shared lock (others can read, not write)
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithReadLock(@Param("id") Long id);
}

@Service
public class BookingService {

    @Transactional(timeout = 10) // auto-rollback after 10s to prevent lock holding
    public void bookSeat(Long seatId, Long userId) {
        // Locks the row — other transactions block on this findById until we commit
        Seat seat = seatRepository.findByIdWithWriteLock(seatId)
            .orElseThrow(() -> new SeatNotFoundException(seatId));

        if (!seat.isAvailable()) {
            throw new SeatAlreadyBookedException(seatId);
        }

        seat.setAvailable(false);
        seat.setBookedBy(userId);
        seatRepository.save(seat);
        // Lock released on transaction commit
    }
}
```

| | Optimistic | Pessimistic |
|---|---|---|
| Lock held in DB | No | Yes (`SELECT FOR UPDATE`) |
| Conflict detection | At commit | At read |
| Throughput | High (no lock wait) | Lower (waiters block) |
| Suitable when | Conflicts rare | Conflicts frequent |
| Risk | `OptimisticLockException` at commit | Deadlocks |
| Retry needed | Often | Rarely (but timeout needed) |

### 7.3 Deadlocks

A **deadlock** occurs when thread A holds lock L1 and waits for L2, while thread B holds L2 and waits for L1 — both block forever.

```
Thread A: LOCKS seat_1 → waiting for seat_2
Thread B: LOCKS seat_2 → waiting for seat_1
                ↑                    ↑
             DEADLOCK! Neither can proceed
```

**Prevention strategies:**

```java
// ✅ Strategy 1: Always acquire locks in consistent order
@Transactional
public void swapSeats(Long seatId1, Long seatId2, Long userId) {
    // Always lock lower ID first — prevents A(1,2) vs B(2,1) deadlock
    Long firstId  = Math.min(seatId1, seatId2);
    Long secondId = Math.max(seatId1, seatId2);

    Seat first  = seatRepo.findByIdWithWriteLock(firstId).orElseThrow();
    Seat second = seatRepo.findByIdWithWriteLock(secondId).orElseThrow();
    // swap logic
}

// ✅ Strategy 2: Use transaction timeout
@Transactional(timeout = 5) // auto-rollback prevents infinite deadlock
public void updateOrder(Long orderId) { /* ... */ }

// ✅ Strategy 3: Keep transactions short — release locks ASAP
@Transactional
public void shortTransaction(Long id) {
    Entity e = repo.findByIdWithWriteLock(id).orElseThrow();
    e.setStatus("DONE");
    repo.save(e);
    // commit IMMEDIATELY — don't do HTTP calls, heavy computation inside @Transactional!
}
```

---

## 8. Distributed Concurrency

### 8.1 Why `synchronized` Fails in Distributed Systems

```
Pod 1 (JVM 1)                    Pod 2 (JVM 2)
──────────────                   ──────────────
synchronized(this) { ← THIS       synchronized(this) { ← DIFFERENT OBJECT!
  check inventory                   check inventory
  decrement                         decrement ← BOTH can enter! synchronized is per-JVM
}                                }
```

`synchronized` only works within a single JVM. In a cluster of microservices, you need a **distributed lock**.

### 8.2 Distributed Locks with Redis (Redisson)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.25.0</version>
</dependency>
```

```java
@Service
public class DistributedInventoryService {

    @Autowired
    private RedissonClient redissonClient;

    @Autowired
    private InventoryRepository inventoryRepository;

    public boolean reserveItem(String itemSku, int quantity) {
        // Lock key is specific to the item — no cross-item contention
        String lockKey = "inventory:lock:" + itemSku;
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // Try to acquire lock: wait up to 5s, auto-expire in 10s
            // Auto-expire prevents lock starvation if holder crashes
            boolean acquired = lock.tryLock(5, 10, TimeUnit.SECONDS);
            if (!acquired) {
                throw new LockAcquisitionException("Could not acquire lock for: " + itemSku);
            }

            // Now we're safe — only one JVM across the cluster holds this lock
            Inventory inventory = inventoryRepository.findBySku(itemSku);
            if (inventory.getAvailable() < quantity) {
                return false;
            }
            inventory.setAvailable(inventory.getAvailable() - quantity);
            inventoryRepository.save(inventory);
            return true;

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Lock interrupted", e);
        } finally {
            // Only unlock if WE hold the lock (prevents unlocking another thread's lock)
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**Redisson also provides fair locks, read-write locks, and semaphores:**

```java
@Service
public class RateLimiterService {

    @Autowired
    private RedissonClient redissonClient;

    // Distributed semaphore — limit concurrent access across cluster
    public void callExternalApi(String apiKey) throws InterruptedException {
        RSemaphore semaphore = redissonClient.getSemaphore("api:permits:" + apiKey);
        semaphore.trySetPermits(10); // max 10 concurrent calls across ALL pods

        if (semaphore.tryAcquire(2, TimeUnit.SECONDS)) {
            try {
                externalApiClient.call();
            } finally {
                semaphore.release();
            }
        } else {
            throw new TooManyRequestsException("Rate limit exceeded");
        }
    }
}
```

### 8.3 Database-Based Distributed Locks

```java
@Service
public class DbDistributedLockService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    // Uses a dedicated lock table
    // CREATE TABLE distributed_locks (
    //   lock_name VARCHAR(100) PRIMARY KEY,
    //   acquired_by VARCHAR(100),
    //   acquired_at TIMESTAMP,
    //   expires_at TIMESTAMP
    // );

    @Transactional
    public boolean tryAcquireLock(String lockName, String instanceId, Duration ttl) {
        try {
            int rows = jdbcTemplate.update("""
                INSERT INTO distributed_locks (lock_name, acquired_by, acquired_at, expires_at)
                VALUES (?, ?, NOW(), ?)
                ON CONFLICT (lock_name) DO UPDATE
                  SET acquired_by = EXCLUDED.acquired_by,
                      acquired_at = EXCLUDED.acquired_at,
                      expires_at  = EXCLUDED.expires_at
                  WHERE distributed_locks.expires_at < NOW()
                """,
                lockName, instanceId, Timestamp.from(Instant.now().plus(ttl)));
            return rows > 0;
        } catch (DuplicateKeyException e) {
            return false; // another instance holds the lock
        }
    }

    @Transactional
    public void releaseLock(String lockName, String instanceId) {
        jdbcTemplate.update(
            "DELETE FROM distributed_locks WHERE lock_name = ? AND acquired_by = ?",
            lockName, instanceId);
    }
}
```

> [!TIP]
> Redis-based locks (Redisson) are preferred over DB locks for performance. Use DB locks when Redis is not available or for simplicity in low-throughput scenarios. Always set a **TTL** on distributed locks to prevent starvation if the lock holder crashes.

---

## 9. Common Concurrency Bugs in Spring

### 9.1 Shared Mutable State in Singleton Beans

```java
// ❌ BROKEN — mutable state in singleton
@Service
public class BrokenReportBuilder {
    private StringBuilder buffer = new StringBuilder(); // SHARED!

    public String buildReport(List<String> lines) {
        buffer.setLength(0); // Thread A clears; Thread B's content gone!
        lines.forEach(line -> buffer.append(line).append("\n")); // concurrent append → garbage
        return buffer.toString();
    }
}

// ✅ FIXED — local state
@Service
public class SafeReportBuilder {
    public String buildReport(List<String> lines) {
        StringBuilder buffer = new StringBuilder(); // per-thread, on stack
        lines.forEach(line -> buffer.append(line).append("\n"));
        return buffer.toString();
    }
}
```

### 9.2 Instance Variables in @Controller

```java
// ❌ CLASSIC BUG — @Controller is singleton!
@RestController
@RequestMapping("/api")
public class BrokenController {
    private String currentUser;          // SHARED — all requests overwrite each other!
    private List<String> requestLog = new ArrayList<>(); // NOT thread-safe, SHARED!

    @GetMapping("/process")
    public String process(Authentication auth) {
        currentUser = auth.getName();    // Thread A sets "alice"; Thread B sets "bob"
        requestLog.add(currentUser);     // concurrent modification!
        return "Hello " + currentUser;  // Thread A may return "Hello bob" due to race!
    }
}

// ✅ FIXED — use local variables and thread-safe collections
@RestController
@RequestMapping("/api")
public class SafeController {
    private final List<String> requestLog = new CopyOnWriteArrayList<>();

    @GetMapping("/process")
    public String process(Authentication auth) {
        String currentUser = auth.getName(); // local variable — thread-safe!
        requestLog.add(currentUser);
        return "Hello " + currentUser;
    }
}
```

### 9.3 SimpleDateFormat Is Not Thread-Safe

```java
// ❌ BROKEN — SimpleDateFormat is not thread-safe (internal Calendar state is mutable)
@Service
public class BrokenDateService {
    private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd");

    public String format(Date date) {
        return SDF.format(date); // concurrent calls corrupt internal Calendar state!
        // Results in NumberFormatException, wrong dates, or ParseException
    }
}

// ✅ Fix 1: Use ThreadLocal<SimpleDateFormat>
@Service
public class ThreadLocalDateService {
    private static final ThreadLocal<SimpleDateFormat> SDF =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public String format(Date date) {
        return SDF.get().format(date); // each thread has its own instance
    }
}

// ✅ Fix 2 (Best): Use java.time.DateTimeFormatter — immutable and thread-safe
@Service
public class ModernDateService {
    private static final DateTimeFormatter FORMATTER =
        DateTimeFormatter.ofPattern("yyyy-MM-dd"); // immutable, safe to share

    public String format(LocalDate date) {
        return date.format(FORMATTER);
    }
}
```

### 9.4 HttpServletRequest Sharing

```java
// ❌ BROKEN — storing HttpServletRequest in an instance field
@Service
public class BrokenRequestService {
    private HttpServletRequest currentRequest; // ← NEVER DO THIS!

    public void setRequest(HttpServletRequest req) {
        this.currentRequest = req; // Thread A's request overwritten by Thread B
    }

    public String getClientIp() {
        return currentRequest.getRemoteAddr(); // may return wrong IP or NPE
    }
}

// ✅ FIXED — inject RequestContextHolder or pass request as method parameter
@Service
public class SafeRequestService {

    public String getClientIp() {
        HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.getRequestAttributes()).getRequest();
        return request.getRemoteAddr(); // gets THIS thread's request
    }
}
```

### 9.5 Lazy Initialization Race Condition — Double-Checked Locking

```java
// ❌ BROKEN — without volatile, other threads may see partially constructed object
public class BrokenSingleton {
    private static BrokenSingleton instance;

    public static BrokenSingleton getInstance() {
        if (instance == null) {              // Check 1: Thread A and B both see null
            synchronized (BrokenSingleton.class) {
                if (instance == null) {      // Check 2
                    instance = new BrokenSingleton();
                    // JVM may reorder: allocate memory → assign to instance → call constructor
                    // Thread B may see non-null but uninitialized instance!
                }
            }
        }
        return instance;
    }
}

// ✅ FIXED — volatile prevents reordering
public class SafeSingleton {
    private static volatile SafeSingleton instance; // volatile ensures happens-before

    public static SafeSingleton getInstance() {
        if (instance == null) {
            synchronized (SafeSingleton.class) {
                if (instance == null) {
                    instance = new SafeSingleton();
                }
            }
        }
        return instance;
    }
}

// ✅ Even Better — Initialization-on-demand holder (no synchronization needed)
public class BestSingleton {
    private BestSingleton() {}

    private static class Holder {
        private static final BestSingleton INSTANCE = new BestSingleton();
        // JVM guarantees class initialization is thread-safe
    }

    public static BestSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## 10. WebFlux & Reactive — Brief Overview

### 10.1 Reactive vs Imperative Concurrency Model

| | Spring MVC (Imperative) | Spring WebFlux (Reactive) |
|---|---|---|
| Thread model | One thread per request | Small fixed event loop (Netty) |
| Blocking allowed | Yes | ⚠️ NO — blocks event loop |
| Scalability | Limited by thread count | Handles millions of connections |
| Programming model | Synchronous, simple | Reactive streams, complex |
| Database support | JPA, JDBC | R2DBC (reactive), MongoDB reactive |
| Best for | Standard CRUD apps | High-concurrency, streaming, I/O-heavy |

### 10.2 When to Use WebFlux

```java
// Spring WebFlux controller — non-blocking
@RestController
@RequestMapping("/reactive")
public class ReactiveProductController {

    @Autowired ReactiveProductService productService;

    // Mono<T> — 0 or 1 result (reactive equivalent of CompletableFuture)
    @GetMapping("/{id}")
    public Mono<Product> getProduct(@PathVariable Long id) {
        return productService.findById(id); // non-blocking all the way down
    }

    // Flux<T> — 0 or N results (reactive stream)
    @GetMapping("/stream")
    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Product> streamProducts() {
        return productService.findAll()
            .delayElements(Duration.ofMillis(100)); // push events every 100ms
    }
}
```

### 10.3 Blocking in Reactive Context — The Big Trap

```java
// ❌ CRITICAL BUG — blocking call on Netty event loop thread
@Service
public class BrokenReactiveService {

    public Mono<String> fetchData(Long id) {
        return Mono.fromSupplier(() -> {
            // This BLOCKS the event loop thread!
            return jdbcTemplate.queryForObject("SELECT data FROM t WHERE id=?",
                String.class, id); // JDBC is blocking!
            // Netty event loop is now blocked → ALL requests to this pod stall!
        });
    }
}

// ✅ FIXED — use subscribeOn to offload blocking work to a bounded elastic scheduler
@Service
public class FixedReactiveService {

    public Mono<String> fetchData(Long id) {
        return Mono.fromSupplier(() ->
            jdbcTemplate.queryForObject("SELECT data FROM t WHERE id=?", String.class, id))
            .subscribeOn(Schedulers.boundedElastic()); // runs on dedicated blocking thread pool
    }
}

// ✅ BEST — use R2DBC (truly non-blocking database driver)
@Repository
public interface ReactiveProductRepository extends ReactiveCrudRepository<Product, Long> {
    Flux<Product> findByCategoryId(Long categoryId);
}
```

> [!CAUTION]
> **Never call blocking code (JDBC, RestTemplate, Thread.sleep, synchronized) from within a reactive pipeline without `subscribeOn(Schedulers.boundedElastic())`**. Doing so blocks the tiny event-loop thread pool and causes the entire application to stall.

---

## ⚠️ Interview Traps & Gotchas

### Trap 1: "Is Spring Singleton Thread-Safe?"

**Trick answer:** It depends on whether the bean has **mutable state**.

- Spring guarantees each singleton is created once and shared — it does NOT guarantee thread-safety.
- A singleton with **no mutable instance fields** (stateless) is naturally thread-safe.
- A singleton with **mutable instance fields** requires explicit synchronization.

```java
// Thread-SAFE singleton (no mutable state)
@Service
public class UserService {
    private final UserRepository repo; // final, immutable
    public User findUser(Long id) { return repo.findById(id).orElseThrow(); }
}

// NOT thread-safe singleton (mutable state)
@Service
public class CounterService {
    private int count = 0;           // ← mutable instance field
    public void increment() { count++; } // race condition!
}
```

### Trap 2: "What Does @Async Use by Default?"

**Answer:** `SimpleAsyncTaskExecutor` — which **creates a new thread for every invocation** with no pooling. This is a production disaster under load.

**Always configure a `ThreadPoolTaskExecutor` before using `@Async` in production.**

### Trap 3: "Can You Make @Async Synchronized?"

**Answer:** Technically yes, but it defeats the purpose:

```java
@Async
public synchronized CompletableFuture<Void> serialAsync(String data) {
    // Only one thread at a time due to synchronized
    // @Async puts it on a pool thread, synchronized serializes pool threads
    // Net effect: serialized async — you're just offloading to a single pool thread
    process(data);
    return CompletableFuture.completedFuture(null);
}
```

Better alternatives: use a `SingleThreadExecutor`, or a `LinkedBlockingQueue` with a single consumer.

### Trap 4: "ThreadLocal in @Async Context"

```java
// ❌ ThreadLocal does NOT propagate to @Async threads
ThreadLocal<String> local = new ThreadLocal<>();
local.set("request-context");

asyncService.doWork(); // @Async runs on a DIFFERENT thread — local.get() returns NULL!

// Fix: capture and manually pass
String ctx = local.get();
CompletableFuture.runAsync(() -> {
    local.set(ctx); // set on async thread
    try { doWork(); } finally { local.remove(); }
}, executor);
```

### Trap 5: "Singleton Bean with Prototype Dependency"

```java
// ❌ Singleton holds a reference to ONE prototype bean — defeats prototype scope!
@Service
public class SingletonService {
    @Autowired
    private PrototypeBean prototypeBean; // Injected ONCE at startup!
    // Every call to prototypeBean uses the SAME instance — not a new one each time
}

// ✅ Fix: use ApplicationContext.getBean() or ObjectFactory/Provider
@Service
public class CorrectSingletonService {
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void doWork() {
        PrototypeBean bean = prototypeBeanProvider.getObject(); // new instance each call
        bean.execute();
    }
}
```

### Trap 6: "volatile vs AtomicInteger"

```java
private volatile int count = 0;
count++; // NOT atomic! Still a race condition!
// volatile only ensures visibility — not atomicity of compound operations

private final AtomicInteger atomicCount = new AtomicInteger(0);
atomicCount.incrementAndGet(); // atomic — uses CAS hardware instruction — NO race condition
```

### Trap 7: "synchronized vs ReentrantLock"

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Syntax | Keyword (simpler) | Object (more code) |
| Fairness | No (barging) | Optional |
| Interruptible | No | `lockInterruptibly()` |
| Try with timeout | No | `tryLock(n, unit)` |
| Multiple conditions | One (`wait`/`notify`) | Multiple (`newCondition()`) |
| Automatic release | Yes (JVM manages) | No (must `unlock()` in `finally`!) |

> [!WARNING]
> Forgetting to call `lock.unlock()` in a `finally` block with `ReentrantLock` causes permanent deadlock. This is the primary argument for `synchronized` in simple cases.

---

## 🔍 Deep Dive — JMM and Happens-Before

The **Java Memory Model (JMM)** defines rules for when one thread's writes are visible to another thread's reads. Without visibility guarantees, CPUs and compilers reorder operations for performance.

### Happens-Before Rules

A happens-before (HB) relationship means: *all writes done by A before the HB point are visible to B after the HB point.*

| Action | Happens-Before |
|---|---|
| `synchronized` exit | subsequent `synchronized` entry on same monitor |
| `volatile` write | subsequent `volatile` read of same variable |
| `Thread.start()` | all actions in the started thread |
| Thread completion | `Thread.join()` returns |
| Object construction | `final` fields seen by all threads |
| `CompletableFuture.complete()` | subsequent reads of the result |

```java
// Without HB, this can print 0 (CPU reorders writes/reads)
int x = 0;
boolean ready = false;

// Thread A:
x = 42;      // Write 1
ready = true; // Write 2 — CPU may reorder before Write 1!

// Thread B:
while (!ready); // Read 2
System.out.println(x); // Read 1 — may see 0 without HB guarantee!

// Fix: make 'ready' volatile — volatile write happens-before volatile read
volatile boolean ready = false;
// Now: x=42 happens-before ready=true (program order for Thread A)
//       ready=true (write) happens-before !ready (read in Thread B)
//       therefore: x=42 happens-before printing x → x is visible as 42
```

---

## 📝 Summary & Key Takeaways

| Topic | Key Principle |
|---|---|
| Spring Singleton | One instance, many threads — stateless beans are safe, stateful are not |
| ThreadLocal | Per-thread storage; always clear in `finally`; NOT propagated to `@Async` |
| Race Conditions | Check-then-act and read-modify-write are the two root patterns |
| `synchronized` | Mutual exclusion + visibility; avoid on `this` in proxied beans |
| `volatile` | Visibility only — not atomic for compound ops like `count++` |
| `AtomicInteger` | Lock-free CAS atomicity; use `LongAdder` for high-contention counters |
| `ConcurrentHashMap` | Lock-free reads, fine-grained writes; use `computeIfAbsent` for atomic put-if-absent |
| `@Async` | Must be public, non-self-call; configure `ThreadPoolTaskExecutor`; default is `SimpleAsyncTaskExecutor` (DO NOT USE in prod) |
| Thread Pool | Understand core/max/queue interaction; queue fills before max threads spawn |
| `CompletableFuture` | `thenApply` (sync), `thenCompose` (flatmap), `allOf` (wait all), `exceptionally` (catch) |
| Optimistic Locking | `@Version` — no DB lock; retries on `OptimisticLockException` |
| Pessimistic Locking | `SELECT FOR UPDATE`; use timeouts to prevent deadlocks |
| Distributed Locks | `synchronized` is JVM-local; use Redisson for Redis-based cluster locks |
| WebFlux | Never block the event loop — use `subscribeOn(Schedulers.boundedElastic())` |
| Virtual Threads | Java 21 — lightweight, great for I/O; don't pool them |
| JMM | `volatile` + `synchronized` provide happens-before guarantees |

---

## ❓ Practice Questions

1. You have a `@Service` singleton with a `private List<String> cache = new ArrayList<>()` that's populated on first use. What are the concurrency problems, and how would you fix them without removing the field?

2. Your Spring Boot app uses `@Async` to send emails. Under load testing, you observe `OutOfMemoryError: unable to create new native thread`. What is the likely cause and how do you fix it?

3. Explain exactly why `count++` is not thread-safe even when `count` is `volatile`. What should you use instead?

4. An `@Async` method needs to access `SecurityContextHolder.getContext()`. In your test, it returns `null`. Why, and what are three ways to fix it?

5. You have two services that each lock tables in opposite order:
   - ServiceA: locks `orders` then `payments`
   - ServiceB: locks `payments` then `orders`
   What problem does this cause and how do you fix it?

6. What is the difference between `thenApply` and `thenCompose` in `CompletableFuture`? When would you use each?

7. A colleague says "I used `@Transactional` on my `@Async` method, so the async operation participates in the caller's transaction." Is this correct? Explain what actually happens.

8. In a Kubernetes deployment with 5 pods, a `synchronized` block in a singleton Spring bean is meant to prevent duplicate order processing. Will this work? Why or why not, and what should you use instead?

9. Your `@Entity` has `@Version Long version`. Under high concurrency, you observe many `OptimisticLockingFailureException` errors. What does this indicate about your data access pattern, and would you switch to pessimistic locking? Justify your answer.

10. Explain the double-checked locking pattern. Why is `volatile` mandatory for it to work correctly? What is the JMM issue it addresses?
