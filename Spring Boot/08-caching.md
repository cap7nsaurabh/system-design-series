# 08 — Caching in Spring Boot

> A production-grade architectural guide to caching fundamentals, cache topologies, multi-approach caching patterns, stampede mitigation, distributed Redis caching, L1/L2 multi-tier setups, serialization strategies, and cache invalidation in Spring Boot 3.x and Java 17/21+.

---

## Table of Contents

1. [Caching Core Architecture in Spring Framework](#1-caching-core-architecture-in-spring-framework)
   - [1.1 `@EnableCaching` & The CacheInterceptor Proxy Pipeline](#11-enablecaching--the-cacheinterceptor-proxy-pipeline)
   - [1.2 The `CacheManager` & `Cache` Core Abstractions](#12-the-cachemanager--cache-core-abstractions)
2. [Multi-Approach: Cache Topologies & Storage Providers](#2-multi-approach-cache-topologies--storage-providers)
   - [Approach 1: High-Performance In-Memory Caffeine Cache (Single-Node Standard)](#approach-1-high-performance-in-memory-caffeine-cache-single-node-standard)
   - [Approach 2: Clustered Distributed Redis Cache (Multi-Pod Production Standard)](#approach-2-clustered-distributed-redis-cache-multi-pod-production-standard)
   - [Approach 3: Two-Tier Multi-Level Hybrid Cache (L1 Caffeine + L2 Redis + Pub/Sub Sync)](#approach-3-two-tier-multi-level-hybrid-cache-l1-caffeine--l2-redis--pubsub-sync)
   - [Approach 4: In-Memory `ConcurrentMapCacheManager` (Development & Testing)](#approach-4-in-memory-concurrentmapcachemanager-development--testing)
   - [Cache Topologies Comparison Matrix](#cache-topologies-comparison-matrix)
3. [Multi-Approach: Cache Access & Persistence Patterns](#3-multi-approach-cache-access--persistence-patterns)
   - [Approach 1: Cache-Aside (Lazy Loading / Read-Through Emulation)](#approach-1-cache-aside-lazy-loading--read-through-emulation)
   - [Approach 2: Write-Through Synchronous Persistence](#approach-2-write-through-synchronous-persistence)
   - [Approach 3: Write-Behind (Write-Back) Asynchronous Ingestion](#approach-3-write-behind-write-back-asynchronous-ingestion)
   - [Approach 4: Refresh-Ahead / Proactive Background Reloading](#approach-4-refresh-ahead--proactive-background-reloading)
   - [Cache Access Patterns Comparison Matrix](#cache-access-patterns-comparison-matrix)
4. [Multi-Approach: Declarative Annotations vs. Programmatic Operations](#4-multi-approach-declarative-annotations-vs-programmatic-operations)
   - [Approach 1: Declarative Annotations (`@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`)](#approach-1-declarative-annotations-cacheable-cacheput-cacheevict-caching)
   - [Approach 2: Programmatic Management via Spring `CacheManager`](#approach-2-programmatic-management-via-spring-cachemanager)
   - [Approach 3: Native Driver Operations via `RedisTemplate<String, Object>`](#approach-3-native-driver-operations-via-redistemplatestring-object)
   - [Declarative vs Programmatic Comparison Matrix](#declarative-vs-programmatic-comparison-matrix)
5. [Multi-Approach: Cache Stampede (Thundering Herd) Mitigation](#5-multi-approach-cache-stampede-thundering-herd-mitigation)
   - [Approach 1: In-Process Mutex Lock via `@Cacheable(sync = true)`](#approach-1-in-process-mutex-lock-via-cacheablesync--true)
   - [Approach 2: Distributed Lock Mutex via Redisson (`tryLock`)](#approach-2-distributed-lock-mutex-via-redisson-trylock)
   - [Approach 3: Probabilistic Early Expiration (XFetch Algorithm)](#approach-3-probabilistic-early-expiration-xfetch-algorithm)
   - [Approach 4: Pre-Warm & Background Refresh via Application Events](#approach-4-pre-warm--background-refresh-via-application-events)
   - [Stampede Mitigation Comparison Matrix](#stampede-mitigation-comparison-matrix)
6. [Multi-Approach: Redis Serialization & Schema Evolution](#6-multi-approach-redis-serialization--schema-evolution)
   - [Approach 1: Generic JSON Serialization with Class Metadata (`GenericJackson2JsonRedisSerializer`)](#approach-1-generic-json-serialization-with-class-metadata-genericjackson2jsonredisserializer)
   - [Approach 2: Strict Typed JSON DTO Serialization (`Jackson2JsonRedisSerializer<T>`)](#approach-2-strict-typed-json-dto-serialization-jackson2jsonredisserializert)
   - [Approach 3: Binary Protocol Buffers (Protobuf) Serialization](#approach-3-binary-protocol-buffers-protobuf-serialization)
   - [Approach 4: Legacy Java Serialization (`JdkSerializationRedisSerializer`) & Hazards](#approach-4-legacy-java-serialization-jdkserializationredisserializer--hazards)
   - [Serialization Strategies Comparison Matrix](#serialization-strategies-comparison-matrix)
7. [Multi-Approach: Cache Invalidation & Consistency Architectures](#7-multi-approach-cache-invalidation--consistency-architectures)
   - [Approach 1: Time-To-Live (TTL) & Time-To-Idle (TTI) Auto-Expiration](#approach-1-time-to-live-ttl--time-to-idle-tti-auto-expiration)
   - [Approach 2: Transactional Event-Driven Invalidation (`@TransactionalEventListener`)](#approach-2-transactional-event-driven-invalidation-transactionaleventlistener)
   - [Approach 3: Distributed Cross-Service Invalidation via Kafka / Redis Pub/Sub](#approach-3-distributed-cross-service-invalidation-via-kafka--redis-pubsub)
   - [Approach 4: Version-Tagged Key Namespaces (`v2:products:101`)](#approach-4-version-tagged-key-namespaces-v2products101)
   - [Cache Invalidation Comparison Matrix](#cache-invalidation-comparison-matrix)
8. [Production Observability, Micrometer Metrics, & Resilient Error Handlers](#8-production-observability-micrometer-metrics--resilient-error-handlers)
9. [Interview Traps, Anti-Patterns, & Gotchas](#9-interview-traps-anti-patterns--gotchas)
10. [Summary & Architectural Decision Trees](#10-summary--architectural-decision-trees)
11. [Practice Questions & Interview Scenarios](#11-practice-questions--interview-scenarios)

---

## 1. Caching Core Architecture in Spring Framework

### 1.1 `@EnableCaching` & The CacheInterceptor Proxy Pipeline

Spring's cache abstraction is an aspect-oriented programming (AOP) facade. When `@EnableCaching` is declared, Spring imports `CachingConfigurationSelector`, registering the `CacheInterceptor`.

```
Incoming Method Invocation ──> [CGLIB Proxy]
                                      │
                                      ▼
                           [CacheInterceptor (AOP Advice)]
                                      │
                 ┌────────────────────┴────────────────────┐
                 ▼                                         ▼
         [@Cacheable / Read]                     [@CachePut / @CacheEvict]
                 │                                         │
        Check CacheManager                        Execute Target Method
        - HIT: Return cached value                         │
        - MISS: Proceed to target ───────> Store Result / Evict from Cache
```

> [!IMPORTANT]
> Because caching relies on Spring AOP dynamic proxies, calling a `@Cacheable` method from another method **within the same class** completely bypasses the proxy. The call executes against `this` directly, and caching is silently ignored.

### 1.2 The `CacheManager` & `Cache` Core Abstractions

Spring decouples business code from vendor APIs through two primary contracts:
- `CacheManager`: Central registry of named cache regions (e.g., `"products"`, `"users"`).
- `Cache`: SPI representing an individual cache instance providing `get()`, `put()`, and `evict()`.

```java
// Spring Framework Core Cache Interface
public interface Cache {
    String getName();
    Object getNativeCache(); // Returns underlying Redis or Caffeine client
    ValueWrapper get(Object key);
    <T> T get(Object key, Class<T> type);
    void put(Object key, Object value);
    ValueWrapper putIfAbsent(Object key, Object value);
    void evict(Object key);
    void clear();
}
```

---

## 2. Multi-Approach: Cache Topologies & Storage Providers

Choosing the physical cache location determines whether data is private to a JVM or shared across a cluster.

```
                      CACHE TOPOLOGY SPECTRUM
 Low Latency (< 1µs)                               High Consistency
 Single-Node Heap                                  Cluster-Wide Shared
       │                                                   │
       ▼                                                   ▼
[Approach 1: Caffeine] ──> [Approach 3: Two-Tier L1/L2] ──> [Approach 2: Redis]
(Sub-microsecond local)     (Local speed + Cluster sync)     (Multi-pod shared state)
```

---

### Approach 1: High-Performance In-Memory Caffeine Cache (Single-Node Standard)

#### Scenario
Single-instance applications, read-heavy workloads where network latency to an external Redis cluster (0.5ms - 2ms) is unacceptable, or static lookup dictionaries (zip codes, tax tables).

#### Use-Case
Local product category hierarchy caching, currency conversion rate lookup tables, user session metadata on single-node deployments.

#### Benefits Over Others / Trade-offs
- **Benefits**: Sub-microsecond response times (nanoseconds); zero serialization overhead (stores Java heap pointers directly); state-of-the-art Window TinyLFU eviction.
- **Trade-offs**: Cache is node-local; multiple pods maintain duplicate, potentially diverging data; heap memory consumption increases JVM garbage collection pressure.

#### Well-Documented Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```java
package com.example.cache.caffeine;

import com.github.ben-manes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
@EnableCaching
public class CaffeineCacheConfiguration {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("taxRates", "categories");
        
        // Window TinyLFU algorithm with size bounds and time-to-live
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .initialCapacity(100)
            .maximumSize(5000)                        // Evict based on Window TinyLFU when exceeded
            .expireAfterWrite(30, TimeUnit.MINUTES)   // TTL: Evict 30 minutes after creation
            .expireAfterAccess(10, TimeUnit.MINUTES)  // TTI: Evict if idle for 10 minutes
            .recordStats());                          // Exposes hit/miss counters to Actuator

        return cacheManager;
    }
}
```

```java
package com.example.cache.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

@Service
public class TaxLookupService {

    private static final Logger log = LoggerFactory.getLogger(TaxLookupService.class);

    @Cacheable(value = "taxRates", key = "#zipCode")
    public BigDecimal calculateTaxRate(String zipCode) {
        log.info("Cache miss! Calculating tax rate from external source for zip: [{}]", zipCode);
        // Simulating heavy lookup
        return new BigDecimal("0.0825");
    }
}
```

---

### Approach 2: Clustered Distributed Redis Cache (Multi-Pod Production Standard)

#### Scenario
Cloud-native microservices running on Kubernetes across multiple replicas where all pods must see consistent, shared cache entries, and cache contents must survive pod restarts.

#### Use-Case
E-commerce shopping carts, shared user profile caches, centralized rate-limiting buckets, API response caches.

#### Benefits Over Others / Trade-offs
- **Benefits**: Shared state across unlimited pods; independent scaling of cache memory; survives application restarts; built-in TTL support.
- **Trade-offs**: Network round-trip overhead (~1-2ms per operation); requires serialization/deserialization to bytes; introduces Redis infrastructure dependency.

#### Well-Documented Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```java
package com.example.cache.redis;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.boot.autoconfigure.cache.RedisCacheManagerBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

@Configuration
public class ProductionRedisCacheConfiguration {

    @Bean
    public RedisCacheConfiguration defaultCacheConfiguration() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        // Preserves Java type metadata in JSON so generic collections deserialize properly
        objectMapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );

        GenericJackson2JsonRedisSerializer jsonSerializer = new GenericJackson2JsonRedisSerializer(objectMapper);

        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(15)) // Default TTL
            .disableCachingNullValues()       // Prevent caching null values
            .prefixCacheNameWith("app::v1::") // Namespace isolation
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jsonSerializer));
    }

    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        return builder -> builder
            .withCacheConfiguration("products",
                RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(1)))
            .withCacheConfiguration("userSessions",
                RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(30)));
    }
}
```

---

### Approach 3: Two-Tier Multi-Level Hybrid Cache (L1 Caffeine + L2 Redis + Pub/Sub Sync)

#### Scenario
Ultra-high-throughput systems requiring local sub-microsecond in-memory speeds while maintaining cache consistency across thousands of Kubernetes pods.

#### Use-Case
High-frequency financial trading prices, viral social media posts, breaking news articles where querying Redis 100,000 times/second would saturate network cards.

#### Benefits Over Others / Trade-offs
- **Benefits**: Combines nanosecond L1 local memory speed with centralized L2 cluster persistence; offloads 99% of read traffic from Redis.
- **Trade-offs**: High architectural complexity; requires Redis Pub/Sub topic to invalidate L1 local caches across peer pods on write operations.

#### Well-Documented Code Example

```java
package com.example.cache.twotier;

import com.github.ben-manes.caffeine.cache.Cache;
import com.github.ben-manes.caffeine.cache.Caffeine;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Service
public class TwoTierProductCacheService {

    private static final Logger log = LoggerFactory.getLogger(TwoTierProductCacheService.class);
    private static final String TOPIC_EVICT = "cache:evict:products";

    // Tier 1: Local In-Memory Cache (Sub-microsecond)
    private final Cache<Long, Object> l1Cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();

    // Tier 2: Distributed Shared Redis Cache
    private final RedisTemplate<String, Object> redisTemplate;

    public TwoTierProductCacheService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public Object getProduct(Long productId) {
        // Step 1: Check L1 Local Memory (0.001 ms)
        Object l1Value = l1Cache.getIfPresent(productId);
        if (l1Value != null) {
            log.trace("L1 Cache HIT for product: [{}]", productId);
            return l1Value;
        }

        // Step 2: Check L2 Centralized Redis (1 ms)
        String redisKey = "product:" + productId;
        Object l2Value = redisTemplate.opsForValue().get(redisKey);
        if (l2Value != null) {
            log.debug("L2 Cache HIT! Populating L1 for product: [{}]", productId);
            l1Cache.put(productId, l2Value);
            return l2Value;
        }

        // Step 3: Cache MISS on both tiers — Load from Database
        log.info("Full Cache MISS for product: [{}] — Loading from DB", productId);
        Object dbValue = loadFromDatabase(productId);

        // Populate both tiers
        redisTemplate.opsForValue().set(redisKey, dbValue, Duration.ofHours(1));
        l1Cache.put(productId, dbValue);

        return dbValue;
    }

    public void updateProduct(Long productId, Object updatedProduct) {
        persistToDatabase(productId, updatedProduct);

        // Update L2 Redis
        String redisKey = "product:" + productId;
        redisTemplate.opsForValue().set(redisKey, updatedProduct, Duration.ofHours(1));

        // Invalidate local L1
        l1Cache.invalidate(productId);

        // Publish invalidation broadcast to ALL other pods in the cluster via Redis Pub/Sub
        redisTemplate.convertAndSend(TOPIC_EVICT, productId.toString());
    }

    public void handleClusterEvictionMessage(String productIdStr) {
        Long productId = Long.valueOf(productIdStr);
        log.debug("Cluster eviction signal received for product: [{}]. Evicting L1.", productId);
        l1Cache.invalidate(productId);
    }

    private Object loadFromDatabase(Long id) { return "ProductData-" + id; }
    private void persistToDatabase(Long id, Object data) {}
}
```

---

### Approach 4: In-Memory `ConcurrentMapCacheManager` (Development & Testing)

#### Scenario
Local developer workstations, fast integration test suites, and minimal standalone CLI tools.

#### Use-Case
JUnit 5 testing where spinning up a Testcontainers Redis instance is unnecessary.

#### Benefits Over Others / Trade-offs
- **Benefits**: Zero dependencies; bundled inside `spring-context`.
- **Trade-offs**: **Unbounded memory growth**; no TTL or max size support; never use in production.

#### Well-Documented Code Example

```java
package com.example.cache.test;

import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
@Profile("test")
public class TestCacheConfiguration {

    @Bean
    public CacheManager testCacheManager() {
        // Simple ConcurrentHashMap wrapper for unit test execution
        ConcurrentMapCacheManager manager = new ConcurrentMapCacheManager("products", "users");
        manager.setAllowNullValues(false);
        return manager;
    }
}
```

---

### Cache Topologies Comparison Matrix

| Cache Topology | Latency | Data Consistency | Memory Isolation | Cluster Coordinated? | Best For |
|---|---|---|---|---|---|
| **Approach 1: Caffeine** | **< 1 µs** (Local RAM) | Per-JVM (Drift possible) | Direct Java Heap | No | Read-heavy dictionaries, Static data |
| **Approach 2: Redis** | **~1-2 ms** (Network) | **100% Consistent** | External Redis RAM | **Yes (Shared)** | Microservices, Cart/Session storage |
| **Approach 3: Two-Tier (L1/L2)** | **< 1 µs** (L1) / 1ms (L2) | Coordinated via Pub/Sub | Heap + External RAM | **Yes (Pub/Sub)** | Viral feeds, Massive read scale |
| **Approach 4: ConcurrentMap** | < 1 µs | Per-JVM | Unbounded Heap | No | Integration tests & Dev only |

---

## 3. Multi-Approach: Cache Access & Persistence Patterns

```
                                  CACHE PERSISTENCE PATTERNS
                                                │
         ┌──────────────────────────────┬───────┴──────────────────────┐
         ▼                              ▼                              ▼
[Approach 1: Cache-Aside]   [Approach 2: Write-Through]    [Approach 3: Write-Behind]
Application manages both    Cache synchronously writes DB  Cache queues async write to DB
```

---

### Approach 1: Cache-Aside (Lazy Loading / Read-Through Emulation)

#### Scenario
The industry-standard default for general-purpose web microservices. The application orchestrates queries between the cache and database.

#### Use-Case
Standard user profile lookups, order details retrieval.

#### Benefits Over Others / Trade-offs
- **Benefits**: Resilient (cache failure does not break the app; falls through to DB); only requested data occupies cache space.
- **Trade-offs**: Cache miss incurs penalty (query DB + write cache); possibility of stale data if mutations bypass cache invalidation.

#### Well-Documented Code Example

```java
package com.example.cache.patterns;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class CacheAsidePatternService {

    private static final Logger log = LoggerFactory.getLogger(CacheAsidePatternService.class);
    private final RedisTemplate<String, Object> redisTemplate;
    private final UserDatabaseRepository userRepository;

    public CacheAsidePatternService(RedisTemplate<String, Object> redisTemplate, UserDatabaseRepository userRepository) {
        this.redisTemplate = redisTemplate;
        this.userRepository = userRepository;
    }

    public UserProfile fetchUserProfile(Long userId) {
        String cacheKey = "user:" + userId;

        // 1. Check cache first
        try {
            UserProfile cachedUser = (UserProfile) redisTemplate.opsForValue().get(cacheKey);
            if (cachedUser != null) {
                return cachedUser; // Cache HIT
            }
        } catch (Exception e) {
            log.warn("Redis connectivity failure! Falling back to database.", e);
        }

        // 2. Cache MISS — Load from source of truth
        UserProfile dbUser = userRepository.findById(userId)
            .orElseThrow(() -> new IllegalArgumentException("User not found: " + userId));

        // 3. Write back to cache asynchronously or inline
        try {
            redisTemplate.opsForValue().set(cacheKey, dbUser, Duration.ofMinutes(15));
        } catch (Exception e) {
            log.warn("Failed to populate cache for user: [{}]", userId, e);
        }

        return dbUser;
    }
}
```

---

### Approach 2: Write-Through Synchronous Persistence

#### Scenario
Workloads where cache and database must be 100% strictly synchronized at all times, and subsequent reads must immediately reflect writes.

#### Use-Case
Financial account balances, user authentication credential updates.

#### Benefits Over Others / Trade-offs
- **Benefits**: Data is never stale in cache; reads immediately hit updated cache.
- **Trade-offs**: Higher write latency because every mutation must complete two network writes (Cache + DB).

#### Well-Documented Code Example

```java
package com.example.cache.patterns;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Duration;

@Service
public class WriteThroughPatternService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final AccountDatabaseRepository accountRepository;

    public WriteThroughPatternService(RedisTemplate<String, Object> redisTemplate, AccountDatabaseRepository accountRepository) {
        this.redisTemplate = redisTemplate;
        this.accountRepository = accountRepository;
    }

    @Transactional
    public AccountBalance updateBalance(Long accountId, Double newBalance) {
        // 1. Synchronously persist to database
        AccountBalance updated = accountRepository.updateAccountBalance(accountId, newBalance);

        // 2. Synchronously write to cache in same execution path
        String key = "account:" + accountId;
        redisTemplate.opsForValue().set(key, updated, Duration.ofHours(1));

        return updated;
    }
}
```

---

### Approach 3: Write-Behind (Write-Back) Asynchronous Ingestion

#### Scenario
Write-heavy systems where database write I/O is the primary bottleneck and small data loss windows are acceptable in exchange for massive write throughput.

#### Use-Case
Tracking page view counts, IoT sensor telemetry ingestion, gaming leaderboards.

#### Benefits Over Others / Trade-offs
- **Benefits**: Ultra-low write latency (< 1ms); database writes are coalesced and written in bulk batches.
- **Trade-offs**: Crash vulnerability (if the cache or queue fails before writing to DB, pending writes are lost).

#### Well-Documented Code Example

```java
package com.example.cache.patterns;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class WriteBehindPageViewService {

    private final StringRedisTemplate redisTemplate;
    private final PageViewBatchRepository batchRepository;
    private static final String HASH_KEY = "analytics:pageviews";

    public WriteBehindPageViewService(StringRedisTemplate redisTemplate, PageViewBatchRepository batchRepository) {
        this.redisTemplate = redisTemplate;
        this.batchRepository = batchRepository;
    }

    /**
     * Fast write path: In-memory increment (< 1ms).
     */
    public void recordPageView(String pageUrl) {
        redisTemplate.opsForHash().increment(HASH_KEY, pageUrl, 1L);
    }

    /**
     * Periodic batch write to relational DB every 30 seconds.
     */
    @Scheduled(fixedDelay = 30000)
    public void flushPageViewsToDatabase() {
        Map<Object, Object> pendingViews = redisTemplate.opsForHash().entries(HASH_KEY);
        if (!pendingViews.isEmpty()) {
            batchRepository.batchUpdateCounts(pendingViews);
            // Deduct or reset flushed counts
            redisTemplate.delete(HASH_KEY);
        }
    }
}
```

---

### Cache Access Patterns Comparison Matrix

| Pattern | Write Latency | Read Latency | Consistency | Risk |
|---|---|---|---|---|
| **Approach 1: Cache-Aside** | Low (DB only) | Miss: Slow / Hit: Fast | Eventual | Stale data if unevicted |
| **Approach 2: Write-Through** | Higher (DB + Cache) | Fast (Always Hit) | **High (Immediate)** | Higher write latency |
| **Approach 3: Write-Behind** | **Fastest (< 1ms)** | Fast (Always Hit) | Eventual | Data loss if node crashes before flush |

---

## 4. Multi-Approach: Declarative Annotations vs. Programmatic Operations

Spring provides declarative AOP annotations alongside direct programmatic APIs.

---

### Approach 1: Declarative Annotations (`@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`)

#### Scenario
Standard CRUD service methods with predictable method boundaries.

#### Use-Case
Annotating repository lookups, entity updates, and deletion endpoints.

#### Benefits Over Others / Trade-offs
- **Benefits**: Clean, readable code; zero boilerplate cache retrieval/storing logic.
- **Trade-offs**: Subject to proxy self-invocation bugs; limited ability to handle partial exceptions or conditional complex key logic.

#### Well-Documented Code Example

```java
package com.example.cache.declarative;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.Caching;
import org.springframework.stereotype.Service;

@Service
public class DeclarativeCatalogService {

    // 1. @Cacheable: Checks cache; only invokes method body on MISS
    @Cacheable(
        value = "catalogProducts", 
        key = "#sku", 
        condition = "#sku != null", 
        unless = "#result == null"
    )
    public CatalogItem getBySku(String sku) {
        return queryDatabase(sku);
    }

    // 2. @CachePut: Always executes method body; updates cache with result
    @CachePut(value = "catalogProducts", key = "#item.sku")
    public CatalogItem updateItem(CatalogItem item) {
        return persistItem(item);
    }

    // 3. @Caching: Complex composition of multiple evictions and puts
    @Caching(
        evict = {
            @CacheEvict(value = "catalogProducts", key = "#sku"),
            @CacheEvict(value = "searchResults", allEntries = true) // Clear all search caches
        }
    )
    public void deleteBySku(String sku) {
        deleteDatabaseRecord(sku);
    }

    private CatalogItem queryDatabase(String s) { return new CatalogItem(s); }
    private CatalogItem persistItem(CatalogItem item) { return item; }
    private void deleteDatabaseRecord(String s) {}
}
```

---

### Approach 2: Programmatic Management via Spring `CacheManager`

#### Scenario
Dynamic workflows where cache names are determined at runtime, or where partial cache updates occur conditionally within loops.

#### Use-Case
Iterating over imported files and conditionally evicting specific cache partitions based on row data.

#### Benefits Over Others / Trade-offs
- **Benefits**: Immune to AOP self-invocation traps; works on private methods; allows inspecting native cache objects.
- **Trade-offs**: More verbose imperative code.

#### Well-Documented Code Example

```java
package com.example.cache.programmatic;

import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ProgrammaticCacheOrchestrator {

    private final CacheManager cacheManager;

    public ProgrammaticCacheOrchestrator(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public void bulkEvictKeys(String cacheRegion, List<String> targetKeys) {
        Cache cache = cacheManager.getCache(cacheRegion);
        if (cache != null) {
            for (String key : targetKeys) {
                cache.evict(key);
            }
        }
    }

    public <T> T getOrLoad(String cacheRegion, String key, Class<T> type, java.util.function.Supplier<T> loader) {
        Cache cache = cacheManager.getCache(cacheRegion);
        if (cache == null) {
            return loader.get();
        }

        // Programmatic atomic get-or-put
        return cache.get(key, loader::get);
    }
}
```

---

### Approach 3: Native Driver Operations via `RedisTemplate<String, Object>`

#### Scenario
Advanced Redis data structures (Sorted Sets for leaderboards, Hashes for partial field updates, Bitmaps for active user tracking) that Spring's key-value cache abstraction cannot represent.

#### Use-Case
Storing a complex user object as a Redis Hash and modifying only `lastLogin` without re-serializing the entire entity.

#### Benefits Over Others / Trade-offs
- **Benefits**: Full access to 200+ native Redis commands; optimal memory efficiency with Redis data structures.
- **Trade-offs**: Couples code directly to Redis; cannot swap to Caffeine without rewriting code.

#### Well-Documented Code Example

```java
package com.example.cache.nativeapi;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class RedisNativeStructureService {

    private final RedisTemplate<String, Object> redisTemplate;

    public RedisNativeStructureService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void updateSessionField(String sessionId, String field, Object value) {
        String key = "session:" + sessionId;
        // Native Hash update: modifies only a single field instead of whole JSON
        redisTemplate.opsForHash().put(key, field, value);
        redisTemplate.expire(key, Duration.ofMinutes(30));
    }

    public void recordLeaderboardScore(String leaderboard, String username, double score) {
        // Native Sorted Set command (ZADD)
        redisTemplate.opsForZSet().add("leaderboard:" + leaderboard, username, score);
    }
}
```

---

### Declarative vs Programmatic Comparison Matrix

| Approach | Proxy-Dependent? | Supported Data Types | Complexity | Flexibility |
|---|---|---|---|---|
| **Approach 1: Declarative** | **Yes (AOP Proxy)** | Key-Value pairs only | **Lowest** | Method boundaries only |
| **Approach 2: Spring CacheManager** | No | Key-Value pairs | Moderate | Programmatic control |
| **Approach 3: RedisTemplate** | No | Strings, Hashes, Sets, Lists, ZSets | Moderate to High | **Full Redis Capabilities** |

---

## 5. Multi-Approach: Cache Stampede (Thundering Herd) Mitigation

A cache stampede occurs when a high-traffic key expires, causing hundreds of concurrent requests to simultaneously experience a cache miss and overwhelm the backend database.

---

### Approach 1: In-Process Mutex Lock via `@Cacheable(sync = true)`

#### Scenario
Single-instance applications or multi-instance systems using local Caffeine caches where multiple threads on the same JVM must be serialized during a cache miss.

#### Use-Case
High-frequency currency exchange rates requested 500 times/second on a single server.

#### Benefits Over Others / Trade-offs
- **Benefits**: One-line configuration; built into Spring Framework; completely eliminates local thread stampedes.
- **Trade-offs**: Only locks threads **within the same JVM**; does not prevent stampedes across separate Kubernetes pods querying a shared Redis instance.

#### Well-Documented Code Example

```java
package com.example.cache.stampede;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class LocalMutexPriceService {

    /**
     * sync = true: Guarantees that only ONE thread executes the method body on miss.
     * All other concurrent threads wait for the winning thread to compute and store the value.
     */
    @Cacheable(value = "exchangeRates", key = "#currencyPair", sync = true)
    public Double fetchExchangeRate(String currencyPair) {
        // Expensive remote financial API call
        return queryRemoteForexGateway(currencyPair);
    }

    private Double queryRemoteForexGateway(String pair) { return 1.085; }
}
```

---

### Approach 2: Distributed Lock Mutex via Redisson (`tryLock`)

#### Scenario
Clustered microservices sharing a centralized Redis cache where a key expiration would cause hundreds of pods to execute the same heavy database query simultaneously.

#### Use-Case
Black Friday flash sale catalog item, celebrity social media profiles.

#### Benefits Over Others / Trade-offs
- **Benefits**: Cross-cluster coordination; only ONE pod across the entire Kubernetes cluster queries the database; other pods wait and read the refreshed cache.
- **Trade-offs**: Introduces distributed lock acquisition overhead (~2-5ms).

#### Well-Documented Code Example

```java
package com.example.cache.stampede;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Service
public class ClusteredStampedeShieldService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final RedissonClient redissonClient;
    private final DatabaseCatalogRepository databaseRepository;

    public ClusteredStampedeShieldService(RedisTemplate<String, Object> redisTemplate, 
                                          RedissonClient redissonClient, 
                                          DatabaseCatalogRepository databaseRepository) {
        this.redisTemplate = redisTemplate;
        this.redissonClient = redissonClient;
        this.databaseRepository = databaseRepository;
    }

    public CatalogItem getCatalogItemSafe(Long itemId) {
        String cacheKey = "catalog:" + itemId;

        // Step 1: Check cache
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return (CatalogItem) cached;
        }

        // Step 2: Cache MISS — Acquire Distributed Lock
        String lockKey = "lock:cache:" + itemId;
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // Wait up to 3 seconds for lock; auto-release after 5 seconds to prevent deadlocks
            if (lock.tryLock(3, 5, TimeUnit.SECONDS)) {
                try {
                    // Double-Check inside lock: another pod may have populated it while we waited!
                    cached = redisTemplate.opsForValue().get(cacheKey);
                    if (cached != null) {
                        return (CatalogItem) cached;
                    }

                    // WINNER: Only this pod executes the heavy database query
                    CatalogItem loaded = databaseRepository.fetchHeavyItem(itemId);
                    redisTemplate.opsForValue().set(cacheKey, loaded, Duration.ofMinutes(10));
                    return loaded;
                } finally {
                    if (lock.isHeldByCurrentThread()) {
                        lock.unlock();
                    }
                }
            } else {
                // FALLBACK: Could not acquire lock in time; fetch directly to avoid timeout
                return databaseRepository.fetchHeavyItem(itemId);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return databaseRepository.fetchHeavyItem(itemId);
        }
    }
}
```

---

### Approach 3: Probabilistic Early Expiration (XFetch Algorithm)

#### Scenario
Massive read-throughput keys where even a 50ms lock wait causes unacceptable latency spikes.

#### Use-Case
Search engine home page top results, live election vote totals.

#### Benefits Over Others / Trade-offs
- **Benefits**: Zero lock contention; the cache is refreshed in the background **before** it actually expires, based on probabilistic access frequencies.
- **Trade-offs**: Requires storing additional metadata (creation time, computational duration) inside the cache payload.

#### Well-Documented Code Example

```java
package com.example.cache.stampede;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.io.Serializable;
import java.time.Duration;

@Service
public class ProbabilisticEarlyExpiryService {

    public record CacheWrapper<T>(T payload, long expiryEpochMs, long computeDurationMs) implements Serializable {}

    private final RedisTemplate<String, Object> redisTemplate;
    private static final double BETA = 1.0; // Eagerness coefficient

    public ProbabilisticEarlyExpiryService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @SuppressWarnings("unchecked")
    public String getWithXFetch(String key, java.util.function.Supplier<String> loader, Duration ttl) {
        CacheWrapper<String> wrapper = (CacheWrapper<String>) redisTemplate.opsForValue().get(key);

        if (wrapper == null || shouldRecomputeEarly(wrapper)) {
            long start = System.currentTimeMillis();
            String freshValue = loader.get();
            long duration = System.currentTimeMillis() - start;

            long expiryTime = System.currentTimeMillis() + ttl.toMillis();
            CacheWrapper<String> newWrapper = new CacheWrapper<>(freshValue, expiryTime, duration);
            redisTemplate.opsForValue().set(key, newWrapper, ttl);
            return freshValue;
        }

        return wrapper.payload();
    }

    /**
     * Optimal XFetch Probability Formula: -delta * beta * ln(random) > (expiry - now)
     */
    private boolean shouldRecomputeEarly(CacheWrapper<?> wrapper) {
        long remainingTtlMs = wrapper.expiryEpochMs() - System.currentTimeMillis();
        if (remainingTtlMs <= 0) return true;

        double random = Math.random();
        double threshold = -wrapper.computeDurationMs() * BETA * Math.log(random);
        return threshold > remainingTtlMs;
    }
}
```

---

### Stampede Mitigation Comparison Matrix

| Strategy | Scope | Latency Impact | Locking Mechanism | Best For |
|---|---|---|---|---|
| **Approach 1: `sync = true`** | JVM Local | Minimal | In-Process Mutex | Caffeine, Single-node deployments |
| **Approach 2: Redisson Lock** | Cluster-Wide | Low (~2ms lock wait) | Redis Distributed Lock | Multi-pod Kubernetes clusters |
| **Approach 3: XFetch** | Cluster-Wide | **Zero Latency Spikes** | Lock-Free Probabilistic | Ultra-high read APIs, Top trending items |
| **Approach 4: Pre-Warm** | Cluster-Wide | None (Background) | Scheduled Worker | Predictable scheduled traffic spikes |

---

## 6. Multi-Approach: Redis Serialization & Schema Evolution

Objects stored in Redis must be marshaled to byte arrays. Selecting an improper serializer leads to security vulnerabilities, performance degradation, or class cast exceptions.

---

### Approach 1: Generic JSON Serialization with Class Metadata (`GenericJackson2JsonRedisSerializer`)

#### Scenario
Standard Spring Boot applications caching diverse domain entities where developers want human-readable JSON inspection in Redis CLI tools.

#### Use-Case
General entity caching across various service repositories.

#### Benefits Over Others / Trade-offs
- **Benefits**: Human-readable JSON; embeds `@class` metadata so polymorphic objects deserialize cleanly without custom adapters.
- **Trade-offs**: Slightly larger payload size due to type metadata; refactoring class names or packages breaks deserialization of existing cache entries.

#### Well-Documented Code Example

```java
package com.example.cache.serialization;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;

@Configuration
public class GenericJsonSerializerConfig {

    @Bean
    public GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer() {
        ObjectMapper mapper = new ObjectMapper();
        // Crucial: Embeds "@class": "com.example.dto.UserDto" into JSON
        mapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );
        return new GenericJackson2JsonRedisSerializer(mapper);
    }
}
```

---

### Approach 2: Strict Typed JSON DTO Serialization (`Jackson2JsonRedisSerializer<T>`)

#### Scenario
High-throughput specialized caches where type metadata bloat must be avoided and DTO structures are decoupled from internal entity refactorings.

#### Use-Case
Caching `ProductDto` payloads for public REST APIs.

#### Benefits Over Others / Trade-offs
- **Benefits**: Clean JSON without `@class` metadata; immune to backend Java class package renaming; backward-compatible when adding fields using `@JsonIgnoreProperties(ignoreUnknown = true)`.
- **Trade-offs**: Must configure a distinct serializer per target DTO class type.

#### Well-Documented Code Example

```java
package com.example.cache.serialization;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;

@Configuration
public class StrictTypedSerializerConfig {

    public record ProductDto(Long id, String name, Double price) {}

    @Bean
    public Jackson2JsonRedisSerializer<ProductDto> productDtoRedisSerializer() {
        ObjectMapper mapper = new ObjectMapper();
        // Prevents failure if new fields are added to ProductDto in subsequent releases
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        return new Jackson2JsonRedisSerializer<>(mapper, ProductDto.class);
    }
}
```

---

### Approach 3: Binary Protocol Buffers (Protobuf) Serialization

#### Scenario
Massive scale architectures where network bandwidth and Redis memory footprint are critical cost drivers.

#### Use-Case
High-frequency mobile app telemetry payloads, streaming financial ticks.

#### Benefits Over Others / Trade-offs
- **Benefits**: **Smallest memory footprint (up to 70% smaller than JSON)**; fastest serialization and deserialization speeds; strict schema backward-compatibility.
- **Trade-offs**: Binary format is not human-readable in Redis CLI; requires `.proto` schema compilation.

---

### Approach 4: Legacy Java Serialization (`JdkSerializationRedisSerializer`) & Hazards

#### Scenario
Default out-of-the-box behavior when no serializer is configured in Spring Data Redis.

#### Hazards & Why It Should Be Avoided:
1. **Security Vulnerability**: High risk of remote code execution (RCE) via Java deserialization exploits.
2. **Brittle Evolution**: Any change to `serialVersionUID` or adding fields causes `InvalidClassException`.
3. **Bloat**: Large binary headers consume 3x more Redis RAM than compressed JSON.

---

### Serialization Strategies Comparison Matrix

| Serializer | Human-Readable? | Payload Size | Serialization Speed | Schema Evolution Safety | Recommended? |
|---|---|---|---|---|---|
| **Approach 1: Generic JSON** | **Yes** | Moderate | Fast | Moderate (Package tied) | **Yes (General Caching)** |
| **Approach 2: Strict Typed JSON**| **Yes** | Small | Fast | **High (`ignoreUnknown`)** | **Yes (Microservice DTOs)** |
| **Approach 3: Protobuf** | No (Binary) | **Smallest** | **Ultra-Fast** | **Highest (Field IDs)** | High Scale / Mobile APIs |
| **Approach 4: JDK Binary** | No | Largest | Slow | **Dangerous** | **NEVER in Production** |

---

## 7. Multi-Approach: Cache Invalidation & Consistency Architectures

Invalidating stale data is the most critical operational requirement of caching.

---

### Approach 1: Time-To-Live (TTL) & Time-To-Idle (TTI) Auto-Expiration

#### Scenario
Self-healing caches where temporary eventual consistency is acceptable.

#### Use-Case
Product reviews, view counts, weather forecasts.

#### Benefits Over Others / Trade-offs
- **Benefits**: Simple; guaranteed eventual consistency; prevents abandoned orphaned keys from permanently consuming RAM.
- **Trade-offs**: Data remains stale until TTL expires.

#### Well-Documented Code Example

```yaml
# application.yml
spring:
  cache:
    redis:
      time-to-live: 10m # Global 10 minute TTL
      cache-null-values: false
```

---

### Approach 2: Transactional Event-Driven Invalidation (`@TransactionalEventListener`)

#### Scenario
Systems where cache eviction must only occur **after the relational database transaction successfully commits**.

#### Use-Case
Order placement, invoice generation.

#### Benefits Over Others / Trade-offs
- **Benefits**: Guarantees cache is not evicted if the database transaction rolls back; eliminates dirty cache updates.
- **Trade-offs**: Requires publishing application events.

#### Well-Documented Code Example

```java
package com.example.cache.invalidation;

import org.springframework.cache.CacheManager;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Service
public class OrderCheckoutService {

    public record OrderCreatedEvent(Long orderId, Long customerId) {}

    private final ApplicationEventPublisher eventPublisher;
    private final CacheManager cacheManager;

    public OrderCheckoutService(ApplicationEventPublisher eventPublisher, CacheManager cacheManager) {
        this.eventPublisher = eventPublisher;
        this.cacheManager = cacheManager;
    }

    @Transactional
    public void placeOrder(Long customerId) {
        // Database mutation
        Long generatedOrderId = 9981L;

        // Publish event — eviction will wait for DB commit!
        eventPublisher.publishEvent(new OrderCreatedEvent(generatedOrderId, customerId));
    }

    /**
     * Executes AFTER_COMMIT: If transaction rolls back, this NEVER runs!
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreatedEvictCache(OrderCreatedEvent event) {
        var cache = cacheManager.getCache("customerOrders");
        if (cache != null) {
            cache.evict(event.customerId());
        }
    }
}
```

---

### Approach 3: Distributed Cross-Service Invalidation via Kafka / Redis Pub/Sub

#### Scenario
Microservice architectures where Service A updates a table that Service B caches in its local memory.

#### Use-Case
Inventory service updating stock levels; Search service invalidating cached product cards.

#### Benefits Over Others / Trade-offs
- **Benefits**: Near real-time cross-service cache coherence.
- **Trade-offs**: Asynchronous network delivery; event broker dependency.

#### Well-Documented Code Example

```java
package com.example.cache.invalidation.kafka;

import org.springframework.cache.CacheManager;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class DistributedCacheSyncService {

    public record CacheEvictMessage(String cacheName, String key) {}

    private final KafkaTemplate<String, CacheEvictMessage> kafkaTemplate;
    private final CacheManager cacheManager;

    public DistributedCacheSyncService(KafkaTemplate<String, CacheEvictMessage> kafkaTemplate, CacheManager cacheManager) {
        this.kafkaTemplate = kafkaTemplate;
        this.cacheManager = cacheManager;
    }

    public void broadcastEviction(String cacheName, String key) {
        kafkaTemplate.send("cache-evictions-topic", new CacheEvictMessage(cacheName, key));
    }

    @KafkaListener(topics = "cache-evictions-topic", groupId = "#{T(java.util.UUID).randomUUID().toString()}")
    public void onEvictMessageReceived(CacheEvictMessage message) {
        var cache = cacheManager.getCache(message.cacheName());
        if (cache != null) {
            cache.evict(message.key());
        }
    }
}
```

---

### Approach 4: Version-Tagged Key Namespaces (`v2:products:101`)

#### Scenario
Zero-downtime application deployments with breaking data model changes.

#### Use-Case
Deploying Version 2.0 of a service where `Product` DTO structure changes fundamentally from Version 1.0.

#### Benefits Over Others / Trade-offs
- **Benefits**: Instant atomic invalidation without deleting keys; zero cache collision between old and new application versions during blue-green deployment.
- **Trade-offs**: Old keys remain in Redis until their TTL expires.

#### Well-Documented Code Example

```java
package com.example.cache.invalidation.versioning;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class VersionTaggedService {

    // Bumping "v1" to "v2" instantly isolates cache from old pod deployments
    @Cacheable(value = "v2::products", key = "#id")
    public String getProductV2(Long id) {
        return "Product-V2-" + id;
    }
}
```

---

### Cache Invalidation Comparison Matrix

| Strategy | Speed of Invalidation | Transaction Safe? | Cross-Service? | Complexity |
|---|---|---|---|---|
| **Approach 1: TTL Auto-Expire** | Eventual (After TTL) | Yes | Yes | **Lowest** |
| **Approach 2: `@TransactionalEventListener`** | Immediate on Commit | **100% Yes** | No (Local Service) | Low |
| **Approach 3: Kafka / Pub/Sub** | Near Real-Time (~10ms) | Yes | **Yes (Cross-Service)** | Moderate |
| **Approach 4: Version Key Tagging** | **Instantaneous** | Yes | Yes (Deploy boundary) | Low |

---

## 8. Production Observability, Micrometer Metrics, & Resilient Error Handlers

### Resilient Cache Error Handler
By default, if Redis crashes, all `@Cacheable` methods fail and throw exceptions. In production, caching should fail **gracefully**, falling back to the database.

```java
package com.example.cache.resiliency;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.Cache;
import org.springframework.cache.interceptor.CacheErrorHandler;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ResilientCacheErrorHandler implements CacheErrorHandler {

    private static final Logger log = LoggerFactory.getLogger(ResilientCacheErrorHandler.class);

    @Override
    public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
        // DO NOT rethrow! Log warning and allow method to execute against the database!
        log.warn("Cache GET failure on [{}], key [{}]. Falling back to DB: {}", cache.getName(), key, exception.getMessage());
    }

    @Override
    public void handleCachePutError(RuntimeException exception, Cache cache, Object key, Object value) {
        log.warn("Cache PUT failure on [{}], key [{}]: {}", cache.getName(), key, exception.getMessage());
    }

    @Override
    public void handleCacheEvictError(RuntimeException exception, Cache cache, Object key) {
        log.error("Cache EVICT failure on [{}], key [{}]: {}", cache.getName(), key, exception.getMessage());
    }

    @Override
    public void handleCacheClearError(RuntimeException exception, Cache cache) {
        log.error("Cache CLEAR failure on [{}]: {}", cache.getName(), exception.getMessage());
    }
}
```

---

## 9. Interview Traps, Anti-Patterns, & Gotchas

### Trap 1: The Same-Class Self-Invocation Trap
Calling `@Cacheable` from within the same class bypasses the CGLIB proxy. Caching never executes.

### Trap 2: The Silent Null Caching Memory Trap
Failing to declare `unless = "#result == null"` or `disableCachingNullValues()` results in caching `null`. Missing records stay "not found" even after being inserted into the database.

### Trap 3: The Unbounded `ConcurrentMapCacheManager` in Production
Using default in-memory caching in production causes unbounded heap accumulation and inevitable `OutOfMemoryError` because entries never expire.

### Trap 4: Serializing `Page<T>` Objects to Redis
Attempting to cache Spring Data `Page<T>` in Redis throws serialization errors due to inner pageable and sort references. Always cache a plain DTO list (`List<T>`) or custom pagination wrapper.

### Trap 5: Redis Blocking Commands (`KEYS *`)
Using `redisTemplate.keys("prefix*")` executes an $O(N)$ single-threaded scan across millions of keys, completely stalling the Redis server. Always use `SCAN` with cursor batches.

---

## 10. Summary & Architectural Decision Trees

```
                           CACHING DECISION TREE
                                     │
      ┌──────────────────────────────┴──────────────────────────────┐
      ▼                                                             ▼
[Single-Node / Ultra Latency]                             [Clustered Multi-Pod]
  └── In-Memory: Caffeine                                   ├── Standard Shared  ──> RedisCacheManager
      - Size-bound: Window TinyLFU                           ├── Two-Tier L1/L2   ──> Caffeine L1 + Redis L2 + Pub/Sub
      - Stampede: sync = true                                ├── Stampede Shield  ──> Redisson Distributed tryLock
      - TTL: expireAfterWrite                                └── Fail-Safe        ──> ResilientCacheErrorHandler
```

---

## 11. Practice Questions & Interview Scenarios

1. **The Self-Invocation Bug**: A developer annotates `public User findUser(Long id)` with `@Cacheable("users")`. Inside `public User findAndNotify(Long id)` in the same service, they call `this.findUser(id)`. Why does the database get queried every single time?
   - *Answer*: Spring caching operates via AOP proxies. Direct `this` invocations bypass the proxy wrapper. Fix by extracting the cached method into a dedicated bean or injecting `self`.

2. **The Redis Outage Survival**: Redis crashes in production. Suddenly, all REST APIs return HTTP 500. How do you prevent cache infrastructure outages from taking down your application?
   - *Answer*: Configure a custom `CacheErrorHandler`. In `handleCacheGetError`, suppress the exception so execution falls back to the database.

3. **Caffeine vs Redis**: When would you deliberately choose Caffeine over Redis in an enterprise microservice?
   - *Answer*: When response latency must be in sub-microseconds (nanoseconds), data is strictly read-only reference data (tax rates), and network round trips to Redis (~1-2ms) violate SLAs.
