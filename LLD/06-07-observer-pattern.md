# Observer Pattern: Production Architecture & Low-Level Design in Java

> **Category:** [Behavioral Design Patterns](./06-behavioral-design-patterns.md) | **Navigation:** [Back to Behavioral Index](./06-behavioral-design-patterns.md) | [Master LLD Index](./00-index.md)

---


### 7.1 Intent, Architecture & Mechanics

The **Observer Pattern** defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

Key enterprise requirements:
1. **Thread-Safe Registration**: Use of `CopyOnWriteArrayList` to prevent race conditions during concurrent subscribe/unsubscribe calls.
2. **Slow-Subscriber Isolation**: Non-blocking asynchronous dispatch via thread pools to prevent a slow or unresponsive observer from blocking fast message producers.
3. **Memory Leak Prevention**: Use of weak references or explicit subscription lifecycle tokens.

```mermaid
flowchart TD
    subgraph Publisher["High Frequency Stock Exchange Engine"]
        Ticker["StockExchangeTicker<br/>(Subject)"]
        COW["CopyOnWriteArrayList&lt;StockSubscriber&gt;"]
        Executor["ThreadPoolExecutor<br/>(Async Notification Dispatcher)"]
    end

    subgraph Subscribers["Observer Ecosystem"]
        AlgoBot["AlgoTradingBot<br/>(Sub-millisecond Trading)"]
        AuditLog["AuditLogSubscriber<br/>(Batch DB Persister)"]
        MobilePush["MobilePushAlertSubscriber<br/>(High-Latency Network I/O)"]
    end

    Ticker -->|1. Register/Unregister| COW
    Ticker -->|2. Price Tick Triggered| Executor
    Executor -->|Async Worker 1| AlgoBot
    Executor -->|Async Worker 2| AuditLog
    Executor -->|Async Worker 3 (Isolated Latency)| MobilePush
```

### 7.2 Production-Grade Java Implementation: High-Frequency Stock Market Ticker Engine

```java
package com.systemdesign.lld.behavioral.observer;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Objects;

/**
 * Immutable Event representing a market price update.
 */
public final class MarketEvent {
    private final String symbol;
    private final BigDecimal price;
    private final long volume;
    private final Instant timestamp;

    public MarketEvent(String symbol, BigDecimal price, long volume) {
        this.symbol = Objects.requireNonNull(symbol);
        this.price = Objects.requireNonNull(price);
        this.volume = volume;
        this.timestamp = Instant.now();
    }

    public String getSymbol() { return symbol; }
    public BigDecimal getPrice() { return price; }
    public long getVolume() { return volume; }
    public Instant getTimestamp() { return timestamp; }

    @Override
    public String toString() {
        return String.format("[%s] %s = $%s (Vol: %d)", timestamp, symbol, price, volume);
    }
}
```

```java
package com.systemdesign.lld.behavioral.observer;

/**
 * Observer Interface.
 */
public interface StockSubscriber {
    String getSubscriberId();
    void onPriceUpdate(MarketEvent event);
}
```

```java
package com.systemdesign.lld.behavioral.observer;

import java.math.BigDecimal;
import java.util.concurrent.*;

/**
 * Concrete High-Speed Observer.
 */
public final class AlgoTradingBot implements StockSubscriber {
    private final String id;
    private final BigDecimal threshold;

    public AlgoTradingBot(String id, BigDecimal threshold) {
        this.id = id;
        this.threshold = threshold;
    }

    @Override
    public String getSubscriberId() { return id; }

    @Override
    public void onPriceUpdate(MarketEvent event) {
        if (event.getPrice().compareTo(threshold) < 0) {
            System.out.printf("[%s] ALERT: %s fell below $%s! Executing BUY ORDER.%n", 
                id, event.getSymbol(), threshold);
        }
    }
}
```

```java
package com.systemdesign.lld.behavioral.observer;

/**
 * Concrete Slow Observer (Simulating Network Latency).
 */
public final class MobilePushAlertSubscriber implements StockSubscriber {
    private final String id;

    public MobilePushAlertSubscriber(String id) {
        this.id = id;
    }

    @Override
    public String getSubscriberId() { return id; }

    @Override
    public void onPriceUpdate(MarketEvent event) {
        try {
            // Simulate 200ms remote push notification round trip
            Thread.sleep(200);
            System.out.printf("[%s] PUSH SENT to mobile devices for %s%n", id, event.getSymbol());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

```java
package com.systemdesign.lld.behavioral.observer;

import java.math.BigDecimal;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.*;

/**
 * The Subject: Thread-safe, non-blocking asynchronous publisher.
 */
public final class StockExchangeTicker implements AutoCloseable {
    private final String symbol;
    private volatile BigDecimal currentPrice;
    private final List<StockSubscriber> subscribers = new CopyOnWriteArrayList<>();
    
    // Dedicated bounded thread pool isolating subscriber execution from the ticker
    private final ExecutorService dispatchPool = new ThreadPoolExecutor(
        4, 8, 60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );

    public StockExchangeTicker(String symbol, BigDecimal initialPrice) {
        this.symbol = Objects.requireNonNull(symbol);
        this.currentPrice = Objects.requireNonNull(initialPrice);
    }

    public void registerSubscriber(StockSubscriber subscriber) {
        subscribers.add(Objects.requireNonNull(subscriber));
    }

    public void unregisterSubscriber(StockSubscriber subscriber) {
        subscribers.remove(subscriber);
    }

    public void updatePrice(BigDecimal newPrice, long volume) {
        this.currentPrice = Objects.requireNonNull(newPrice);
        MarketEvent event = new MarketEvent(symbol, newPrice, volume);

        // Asynchronous, non-blocking notification dispatch
        for (StockSubscriber subscriber : subscribers) {
            dispatchPool.submit(() -> {
                try {
                    subscriber.onPriceUpdate(event);
                } catch (Throwable t) {
                    System.err.printf("Error notifying subscriber %s: %s%n", 
                        subscriber.getSubscriberId(), t.getMessage());
                }
            });
        }
    }

    @Override
    public void close() {
        dispatchPool.shutdown();
    }
}
```

```java
package com.systemdesign.lld.behavioral.observer;

import java.math.BigDecimal;

public class ObserverPatternDemo {
    public static void main(String[] args) throws Exception {
        try (StockExchangeTicker ticker = new StockExchangeTicker("NVDA", new BigDecimal("120.00"))) {
            AlgoTradingBot bot = new AlgoTradingBot("HighFreqBot-1", new BigDecimal("115.00"));
            MobilePushAlertSubscriber push = new MobilePushAlertSubscriber("APNS-PushGateway");

            ticker.registerSubscriber(bot);
            ticker.registerSubscriber(push);

            System.out.println("=== 1. Ticker Update 1: Normal Price ($118.00) ===");
            ticker.updatePrice(new BigDecimal("118.00"), 50_000);

            System.out.println("\n=== 2. Ticker Update 2: Dip below threshold ($112.00) ===");
            ticker.updatePrice(new BigDecimal("112.00"), 120_000);

            // Await asynchronous dispatch workers
            Thread.sleep(500);
        }
    }
}
```

### 7.3 Step-by-Step Code Walkthrough & Concurrency Analysis
1. **CopyOnWriteArrayList for Traversal**: `CopyOnWriteArrayList` guarantees iterator thread safety without locking during iteration. Even if external threads call `registerSubscriber()` or `unregisterSubscriber()` during a market surge, no `ConcurrentModificationException` is thrown.
2. **Slow-Subscriber Isolation**: If the `MobilePushAlertSubscriber` runs synchronously on the main ticker thread, its 200ms latency blocks every other subscriber and stalls high-frequency price updates. Offloading dispatch to a bounded `ThreadPoolExecutor` ensures that slow consumers cannot degrade publisher throughput.
3. **Reactive Streams & Java Flow API Comparison**: In Java 9+, the Flow API (`Flow.Publisher`, `Flow.Subscriber`, `Flow.Subscription`) extends the Observer pattern by introducing **asynchronous backpressure**. Rather than push-flooding observers, the subscriber requests items explicitly via `subscription.request(n)`.

---

---

## Navigation & Related Patterns
* [Back to Behavioral Patterns Index](./06-behavioral-design-patterns.md)
* [Master LLD Index](./00-index.md)
* [Previous: 06-06-memento-pattern.md](./06-06-memento-pattern.md)
* [Next: 06-08-state-pattern.md](./06-08-state-pattern.md)
