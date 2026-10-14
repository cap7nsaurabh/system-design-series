# Object Pool Pattern: Production Architecture & Low-Level Design in Java

> **Category:** [Creational Design Patterns](./04-creational-design-patterns.md) | **Navigation:** [Back to Creational Index](./04-creational-design-patterns.md) | [Master LLD Index](./00-index.md)

---


### 6.1 Intent & Problem Context
The **Object Pool Pattern** manages a reusable cache of expensive, stateful objects (such as raw TCP connections, cryptography hardware accelerators, or compute threads). Instead of creating and destroying objects on demandâ€”which triggers heavy CPU overhead and high Garbage Collection memory churnâ€”clients borrow an object from the pool, use it, and return it.

---

### 6.2 Complete Production Implementation: Thread-Safe Generic Object Pool

```java
package com.systemdesign.creational.objectpool;

import java.io.Closeable;
import java.time.Duration;
import java.time.Instant;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

// 1. Expensive Resource to be Pooled
public final class PooledDatabaseConnection implements Closeable {

    private final String connectionId;
    private final Instant createdAt;
    private volatile boolean healthy = true;

    public PooledDatabaseConnection() {
        this.connectionId = "conn-" + UUID.randomUUID();
        this.createdAt = Instant.now();
        // Simulate heavy network TCP handshake and authentication handshake
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public void executeQuery(String sql) {
        if (!healthy) {
            throw new IllegalStateException("Connection is invalid: " + connectionId);
        }
        System.out.printf("[DB Socket %s] Executed query: %s%n", connectionId, sql);
    }

    public boolean isValid() {
        return healthy;
    }

    public void markCorrupted() {
        this.healthy = false;
    }

    @Override
    public void close() {
        System.out.printf("[DB Socket %s] Physical TCP socket closed%n", connectionId);
    }

    public String getConnectionId() { return connectionId; }
}

// 2. Factory Interface for Pool Resource Lifecycle
public interface PoolableObjectFactory<T> {
    T create();
    boolean validate(T resource);
    void destroy(T resource);
}

// 3. Thread-Safe Generic Object Pool
public final class GenericObjectPool<T> implements Closeable {

    private final PoolableObjectFactory<T> factory;
    private final int minIdle;
    private final int maxSize;
    private final Duration borrowTimeout;
    private final BlockingQueue<T> availablePool;
    private final ConcurrentMap<T, Instant> borrowedTracking = new ConcurrentHashMap<>();
    private final AtomicInteger totalAllocatedCount = new AtomicInteger(0);
    private final AtomicBoolean isShutdown = new AtomicBoolean(false);
    private final ScheduledExecutorService reaperExecutor = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread t = new Thread(r, "pool-leak-detector");
        t.setDaemon(true);
        return t;
    });

    public GenericObjectPool(PoolableObjectFactory<T> factory, int minIdle, int maxSize, 
                             Duration borrowTimeout, Duration maxLeaseDuration) {
        this.factory = Objects.requireNonNull(factory);
        this.minIdle = minIdle;
        this.maxSize = maxSize;
        this.borrowTimeout = borrowTimeout;
        this.availablePool = new LinkedBlockingQueue<>();

        // Pre-warm the pool up to minIdle
        for (int i = 0; i < minIdle; i++) {
            T obj = factory.create();
            totalAllocatedCount.incrementAndGet();
            availablePool.offer(obj);
        }

        // Schedule background leak detector & idle eviction
        reaperExecutor.scheduleAtFixedRate(() -> detectLeaks(maxLeaseDuration), 
                maxLeaseDuration.toMillis(), maxLeaseDuration.toMillis(), TimeUnit.MILLISECONDS);
    }

    /**
     * Borrows an object from the pool, waiting up to the configured timeout.
     */
    public T acquire() throws InterruptedException, TimeoutException {
        if (isShutdown.get()) {
            throw new IllegalStateException("Object pool is shut down!");
        }

        long deadline = System.currentTimeMillis() + borrowTimeout.toMillis();

        while (System.currentTimeMillis() < deadline) {
            // Check if available in queue
            T resource = availablePool.poll();
            if (resource != null) {
                if (factory.validate(resource)) {
                    borrowedTracking.put(resource, Instant.now());
                    return resource;
                } else {
                    // Invalid object: destroy and decrement count
                    factory.destroy(resource);
                    totalAllocatedCount.decrementAndGet();
                }
            }

            // Can we create a new one under pool capacity?
            int currentAllocated = totalAllocatedCount.get();
            if (currentAllocated < maxSize && totalAllocatedCount.compareAndSet(currentAllocated, currentAllocated + 1)) {
                try {
                    T created = factory.create();
                    borrowedTracking.put(created, Instant.now());
                    return created;
                } catch (Exception ex) {
                    totalAllocatedCount.decrementAndGet();
                    throw new RuntimeException("Failed to allocate new pooled resource", ex);
                }
            }

            // Wait for a returning object
            long waitTimeMs = Math.max(1, deadline - System.currentTimeMillis());
            resource = availablePool.poll(waitTimeMs, TimeUnit.MILLISECONDS);
            if (resource != null) {
                if (factory.validate(resource)) {
                    borrowedTracking.put(resource, Instant.now());
                    return resource;
                } else {
                    factory.destroy(resource);
                    totalAllocatedCount.decrementAndGet();
                }
            }
        }

        throw new TimeoutException("Exhausted pool timeout: Could not acquire resource within " + borrowTimeout);
    }

    /**
     * Returns a borrowed resource to the pool.
     */
    public void release(T resource) {
        if (resource == null) return;

        Instant leaseStart = borrowedTracking.remove(resource);
        if (leaseStart == null) {
            // Double release or alien object returned
            System.err.println("[WARN] Attempted to release unborrowed resource: " + resource);
            return;
        }

        if (isShutdown.get() || !factory.validate(resource)) {
            factory.destroy(resource);
            totalAllocatedCount.decrementAndGet();
            return;
        }

        availablePool.offer(resource);
    }

    /**
     * Periodically sweeps borrowed resources that exceeded maxLeaseDuration (Leak Detection).
     */
    private void detectLeaks(Duration maxLeaseDuration) {
        Instant now = Instant.now();
        borrowedTracking.forEach((resource, startTime) -> {
            if (Duration.between(startTime, now).compareTo(maxLeaseDuration) > 0) {
                System.err.printf("[LEAK DETECTED] Resource %s was borrowed at %s and never returned!%n",
                        resource, startTime);
            }
        });
    }

    @Override
    public void close() {
        if (!isShutdown.compareAndSet(false, true)) return;

        reaperExecutor.shutdownNow();

        // Drain and destroy idle resources
        T resource;
        while ((resource = availablePool.poll()) != null) {
            factory.destroy(resource);
            totalAllocatedCount.decrementAndGet();
        }

        // Destroy borrowed resources upon forced close
        borrowedTracking.keySet().forEach(factory::destroy);
        borrowedTracking.clear();
        totalAllocatedCount.set(0);
    }

    public int getAvailableCount() { return availablePool.size(); }
    public int getActiveCount() { return borrowedTracking.size(); }
    public int getTotalCount() { return totalAllocatedCount.get(); }
}
```

---

### 6.3 Real-World Comparisons & Production Failure Modes
* **HikariCP**: The fastest JDBC connection pool in Java. Replaces `BlockingQueue` with a custom lock-free `FastList` and `ConcurrentBag` to eliminate thread synchronization overhead and CPU cache invalidations.
* **Apache Commons Pool2**: Powers DBCP2 and Jedis pooling using generic `PooledObjectFactory<T>`.
* **Production Failure Modes**:
  1. **Connection Leaks**: Clients borrow connections without a `try-with-resources` or `finally` block, leaving connections stranded in `borrowedTracking` until the pool exhausts.
  2. **Pool Sizing Miscalculation**: Too many connections exhaust database server file descriptors and memory; too few connections cause application request queuing and latency spikes.

---

---

## Navigation & Related Patterns
* [Back to Creational Patterns Index](./04-creational-design-patterns.md)
* [Master LLD Index](./00-index.md)
* [Previous: 04-05-prototype-pattern.md](./04-05-prototype-pattern.md)
