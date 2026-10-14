# 08 — Caching in Spring Boot

> **Series:** Spring Boot Deep Dives | **Level:** Intermediate → Advanced | **Spring Boot Version:** 3.x / Spring 6.x

---

## Table of Contents

1. [Caching Fundamentals](#1-caching-fundamentals)
2. [Spring Cache Abstraction](#2-spring-cache-abstraction)
3. [@Cacheable](#3-cacheable)
4. [@CachePut](#4-cacheput)
5. [@CacheEvict](#5-cacheevict)
6. [@Caching — Composing Multiple Operations](#6-caching--composing-multiple-operations)
7. [Redis Caching with Spring Boot](#7-redis-caching-with-spring-boot)
8. [Caffeine Cache](#8-caffeine-cache)
9. [Cache Patterns & Strategies](#9-cache-patterns--strategies)
10. [Cache Invalidation Strategies](#10-cache-invalidation-strategies)
11. [Same-Class Call Trap](#11-same-class-call-trap)
12. [Serialization with Redis](#12-serialization-with-redis)
13. [Testing Cached Code](#13-testing-cached-code)
14. [⚠️ Interview Traps & Gotchas](#️-interview-traps--gotchas)
15. [🔍 Deep Dive — Internal Mechanics](#-deep-dive--internal-mechanics)
16. [📝 Summary & Key Takeaways](#-summary--key-takeaways)
17. [❓ Practice Questions](#-practice-questions)

---

## 1. Caching Fundamentals

### Why Cache?

Caching is one of the most impactful performance optimizations in backend engineering. The core premise is simple: **avoid re-computing or re-fetching expensive data by storing the result temporarily**.

```
Without Cache:
  Request → Application → Database (50ms) → Response

With Cache:
  Request → Application → Cache HIT (0.5ms) → Response
  Request → Application → Cache MISS → Database (50ms) → Store in Cache → Response
```

**Motivations for caching:**

| Motivation | Explanation |
|---|---|
| **Reduce latency** | In-memory reads (µs) vs DB reads (ms to hundreds of ms) |
| **Reduce database load** | Offload repeated read queries; fewer connections needed |
| **Increase throughput** | Handle more requests with the same hardware |
| **Cost reduction** | Fewer DB calls → smaller DB instance needed |
| **Resilience** | Cache can serve stale data if the downstream is temporarily unavailable |

### Core Cache Concepts

#### Cache Hit & Miss

- **Cache Hit**: The requested data exists in the cache. No call to the origin (DB/API) is made.
- **Cache Miss**: The data is NOT in the cache. The system must fetch from origin, then typically store the result in the cache.
- **Hit Rate**: `hits / (hits + misses)`. A hit rate of 80%+ is typically considered healthy.

#### Cache Eviction

When the cache is full or an item expires, it must be **evicted**. Common eviction policies:

| Policy | Description | Best For |
|---|---|---|
| **LRU** (Least Recently Used) | Evict the item not accessed for the longest time | General purpose |
| **LFU** (Least Frequently Used) | Evict the least accessed item overall | Popularity-based |
| **FIFO** | Evict the oldest inserted item | Simple queues |
| **TTL** (Time To Live) | Evict after a fixed duration | Time-sensitive data |
| **TTI** (Time To Idle) | Evict if not accessed within a window | Session data |
| **Size-based** | Evict when cache size exceeds a limit | Memory-constrained |

---

### Cache Patterns

#### 1. Cache-Aside (Lazy Loading)
The most common pattern. The application code manages the cache manually.

```
READ:
  1. Check cache.
  2. If hit → return cached value.
  3. If miss → query DB, store in cache, return value.

WRITE:
  1. Write to DB.
  2. Invalidate (or update) cache entry.
```

**Pros:** Only requested data is cached. Cache failures don't break the app.  
**Cons:** Cache miss causes latency spike (thundering herd). Data can be stale between write and invalidation.

```java
// Manual cache-aside (without Spring annotations)
public Product getProduct(Long id) {
    String cacheKey = "product:" + id;
    Product cached = redisTemplate.opsForValue().get(cacheKey);
    if (cached != null) {
        return cached; // Cache HIT
    }
    // Cache MISS
    Product product = productRepository.findById(id)
        .orElseThrow(() -> new ProductNotFoundException(id));
    redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(10));
    return product;
}
```

#### 2. Read-Through
The cache sits in front of the database. On a miss, the **cache itself** fetches data from DB and populates itself.

```
READ:
  1. Application always reads from cache.
  2. On miss, cache fetches from DB and stores it (transparent to app).
```

**Pros:** Simpler application code.  
**Cons:** First-request latency. Requires a cache provider that supports this (e.g., NearCache in Hazelcast).

#### 3. Write-Through
Every write goes to the cache first, and the cache **synchronously** writes to the DB.

```
WRITE:
  1. Application writes to cache.
  2. Cache immediately writes to DB.
  3. Acknowledges success.
```

**Pros:** Cache is always consistent with DB.  
**Cons:** Write latency increases (two writes). Cache may hold data that's rarely read.

#### 4. Write-Behind (Write-Back)
Writes go to the cache, and the cache **asynchronously** flushes to DB later.

```
WRITE:
  1. Application writes to cache (fast acknowledgement).
  2. Cache queues the write and flushes to DB asynchronously.
```

**Pros:** Very low write latency.  
**Cons:** Risk of data loss if cache crashes before flushing. More complex to implement correctly.

---

### When NOT to Cache

> [!WARNING]
> Caching can introduce subtle bugs and data inconsistency. Don't cache blindly.

- **Frequently changing data**: If data changes every second, a cached value will almost always be stale.
- **User-specific sensitive data**: Caching per-user data in a shared cache can lead to data leaks if keys are not isolated properly.
- **Non-idempotent computations**: If the result depends on side effects or external mutable state, caching produces incorrect results.
- **Low-traffic endpoints**: The overhead of cache management may exceed the benefit.
- **Small datasets**: If the entire table fits in memory, just load it at startup — caching adds complexity with no gain.
- **Regulatory requirements**: Some data (PII, financial) may not be permitted to be stored in a shared cache layer.

---

## 2. Spring Cache Abstraction

Spring's cache abstraction (introduced in Spring 3.1, enhanced in Spring Boot) provides a **vendor-agnostic API** for caching. You annotate your methods, and Spring handles the rest using a configured `CacheManager`.

### Enabling Caching

```java
@SpringBootApplication
@EnableCaching  // <-- Required! Activates Spring's caching infrastructure
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

> [!IMPORTANT]
> Without `@EnableCaching`, all caching annotations are silently ignored. This is a very common bug.

`@EnableCaching` works by enabling Spring's `CachingConfigurationSelector`, which registers a `CacheInterceptor` (AOP advice) and a `CacheAspectSupport`. This interceptor wraps annotated methods.

### The CacheManager Abstraction

`CacheManager` is the central interface. It manages named `Cache` instances.

```java
public interface CacheManager {
    Cache getCache(String name);
    Collection<String> getCacheNames();
}

public interface Cache {
    String getName();
    Object getNativeCache();
    ValueWrapper get(Object key);
    <T> T get(Object key, Class<T> type);
    void put(Object key, Object value);
    void evict(Object key);
    void clear();
}
```

Spring Boot **auto-configures** the `CacheManager` based on what's on the classpath:

| Condition | Auto-Configured CacheManager |
|---|---|
| Nothing special | `ConcurrentMapCacheManager` (simple in-memory) |
| `caffeine` on classpath | `CaffeineCacheManager` |
| `spring-data-redis` on classpath | `RedisCacheManager` |
| `ehcache` on classpath | `EhCacheCacheManager` / `JCacheCacheManager` |
| `hazelcast` on classpath | `HazelcastCacheManager` |
| Multiple providers | Must explicitly configure one |

### Cache Provider Comparison

| Provider | Type | Persistence | Distributed | TTL Support | Best Use Case |
|---|---|---|---|---|---|
| **ConcurrentMap** | In-memory | No | No | No | Development / Testing |
| **Caffeine** | In-memory | No | No | Yes | Single-node, high-performance |
| **Redis** | In-memory + disk | Optional | Yes | Yes | Multi-node, microservices |
| **EhCache** | In-memory + disk | Optional | Via Terracotta | Yes | Enterprise Java apps |
| **Hazelcast** | In-memory | Yes | Yes | Yes | Distributed computing |

### Manual CacheManager Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        // Using ConcurrentMapCacheManager with named caches
        ConcurrentMapCacheManager manager = new ConcurrentMapCacheManager();
        manager.setCacheNames(List.of("products", "users", "categories"));
        manager.setAllowNullValues(false); // Throw exception on null cache values
        return manager;
    }
}
```

---

## 3. @Cacheable

`@Cacheable` is the most commonly used caching annotation. When applied to a method, Spring checks the cache **before** invoking the method. If the cache contains a value for the computed key, the method is **not executed** and the cached value is returned.

### Basic Usage

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Cacheable("products")  // Cache name = "products"
    public Product findById(Long id) {
        // This body is only executed on a cache MISS
        System.out.println("Fetching product from DB: " + id);
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
}
```

**First call** `findById(1L)`: cache miss → DB call → stores result in cache under key `1`.  
**Second call** `findById(1L)`: cache hit → returns cached product, no DB call.

### Cache Key Generation

#### Default Key Strategy

By default, Spring uses a `SimpleKeyGenerator`:
- **No args**: `SimpleKey.EMPTY`
- **One arg**: the arg itself (e.g., `1L` for `findById(1L)`)
- **Multiple args**: `new SimpleKey(arg1, arg2, ...)`

```java
@Cacheable("products")
public Product findById(Long id) { ... }
// Key: 1L (the Long value)

@Cacheable("products")
public List<Product> findByCategory(String category, boolean active) { ... }
// Key: SimpleKey("electronics", true)
```

> [!WARNING]
> **Key collision risk**: If two methods use the same cache name and have arguments that produce the same key, they will collide. Always use different cache names or explicit keys.

#### Custom SpEL Keys

Spring Expression Language (SpEL) is supported for fine-grained key control:

```java
// Key is the 'id' field of the product object
@Cacheable(value = "products", key = "#product.id")
public Product update(Product product) { ... }

// Key combines two parameters
@Cacheable(value = "orders", key = "#userId + '-' + #orderId")
public Order getOrder(Long userId, Long orderId) { ... }

// Key uses method name + argument
@Cacheable(value = "reports", key = "#root.methodName + '-' + #year")
public Report generateReport(int year) { ... }

// Key uses target class + argument (good for shared generic methods)
@Cacheable(value = "data", key = "#root.targetClass.simpleName + ':' + #id")
public Object findById(Long id) { ... }
```

**SpEL Context Variables:**

| Variable | Description |
|---|---|
| `#root.method` | The method being invoked |
| `#root.target` | The target object (the bean) |
| `#root.args[0]` | First argument |
| `#root.methodName` | Method name as string |
| `#root.targetClass` | The target class |
| `#result` | The return value (only in `@CachePut` and `unless`) |
| `#paramName` | Direct access by parameter name |

#### Custom KeyGenerator

For complex key generation logic, implement `KeyGenerator`:

```java
@Component("productKeyGenerator")
public class ProductKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        StringBuilder key = new StringBuilder();
        key.append(target.getClass().getSimpleName())
           .append(":")
           .append(method.getName())
           .append(":");
        for (Object param : params) {
            key.append(param).append(",");
        }
        return key.toString();
    }
}
```

```java
@Cacheable(value = "products", keyGenerator = "productKeyGenerator")
public Product findProductWithDetails(Long id, String locale) { ... }
// Key: "ProductService:findProductWithDetails:1,en,"
```

---

### Condition vs Unless

These two attributes look similar but behave very differently.

| Attribute | When Evaluated | Can Access `#result` | Behavior |
|---|---|---|---|
| `condition` | **Before** method execution | ❌ No | If `false`, caching is skipped entirely (cache is not read OR written) |
| `unless` | **After** method execution | ✅ Yes | If `true`, the result is NOT stored in cache (but cache is still checked first) |

```java
// condition: Only cache if id > 0
@Cacheable(value = "products", condition = "#id > 0")
public Product findById(Long id) { ... }

// unless: Don't cache null results or inactive products
@Cacheable(value = "products", unless = "#result == null || !#result.active")
public Product findById(Long id) { ... }

// Combining both: only cache positive IDs, and don't cache null results
@Cacheable(
    value = "products",
    condition = "#id > 0",
    unless = "#result == null"
)
public Product findById(Long id) { ... }
```

> [!TIP]
> Use `unless = "#result == null"` to prevent null values from being stored in the cache. Storing nulls can mask errors and lead to confusing behavior.

---

### sync=true — Preventing Cache Stampede

When multiple threads simultaneously request the same uncached item, they all experience a cache miss and all call the underlying method concurrently. This is the **cache stampede** (or thundering herd) problem.

```java
// Without sync — all concurrent threads calling findById(1L) may all hit the DB
@Cacheable("products")
public Product findById(Long id) { ... }

// With sync=true — only ONE thread executes the method; others wait for the result
@Cacheable(value = "products", sync = true)
public Product findById(Long id) { ... }
```

> [!IMPORTANT]
> `sync = true` uses the underlying cache's locking mechanism. For `CaffeineCacheManager`, it uses Caffeine's built-in load locking. For `RedisCacheManager`, **sync is not fully supported** in distributed mode — Redis doesn't natively support per-key locking for reads. Consider distributed locks (Redisson) for Redis stampede prevention.

**Limitations of `sync = true`:**
- Cannot be combined with `unless` attribute
- Cannot be combined with `condition` on the return value
- Works best with Caffeine (native support)

---

### Null Value Caching Trap

By default, Spring **does** cache null values (to avoid repeated DB calls that return nothing). This can cause subtle bugs:

```java
@Cacheable("products")
public Product findById(Long id) {
    return productRepository.findById(id).orElse(null); // Returns null if not found
}

// First call for id=999: DB call returns null, null is cached
// Second call for id=999: Returns null from cache — even after the product is created!
```

**Solutions:**

```java
// Option 1: Use unless to prevent null caching
@Cacheable(value = "products", unless = "#result == null")
public Product findById(Long id) { ... }

// Option 2: Throw exception instead of returning null
@Cacheable("products")
public Product findById(Long id) {
    return productRepository.findById(id)
        .orElseThrow(() -> new ProductNotFoundException(id)); // Never returns null
}

// Option 3: Configure CacheManager to disallow null values
@Bean
public CacheManager cacheManager() {
    ConcurrentMapCacheManager manager = new ConcurrentMapCacheManager("products");
    manager.setAllowNullValues(false); // Throws if method returns null
    return manager;
}
```

---

## 4. @CachePut

`@CachePut` **always executes the method** and then **updates the cache** with the result. The method is never skipped, unlike `@Cacheable`.

### Use Cases

- After an **update** operation: update the DB and refresh the cache simultaneously
- **Pre-warming** the cache with a fresh value

```java
@Service
public class ProductService {

    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        // Always executes — updates DB
        Product saved = productRepository.save(product);
        // Result is stored in cache under key = product.id
        return saved;
    }
}
```

### @Cacheable vs @CachePut

| Feature | `@Cacheable` | `@CachePut` |
|---|---|---|
| **Checks cache first** | ✅ Yes | ❌ No |
| **Skips method on hit** | ✅ Yes | ❌ No (always executes) |
| **Updates cache after** | ✅ Yes (on miss) | ✅ Always |
| **Use case** | Read operations | Write/update operations |

### CachePut with Consistent Key

```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "users", key = "#user.id")  // Same cache, same key format
    public User updateUser(User user) {
        User saved = userRepository.save(user);
        return saved;
    }

    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

> [!TIP]
> Always ensure `@Cacheable` and `@CachePut` on the same logical entity use **identical key expressions** — otherwise they operate on different cache entries.

---

## 5. @CacheEvict

`@CacheEvict` **removes** one or more entries from a cache. It does NOT update the cache — it simply deletes.

### Single Key Eviction

```java
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id);
}
```

### Evict All Entries

```java
@CacheEvict(value = "products", allEntries = true)
public void clearProductCache() {
    // Evicts ALL entries from the "products" cache
    // Useful after bulk imports, admin operations, etc.
}
```

> [!WARNING]
> `allEntries = true` can cause a **cache stampede**: the next wave of requests will all be cache misses simultaneously. Consider cache warming or gradual invalidation for high-traffic systems.

### beforeInvocation

This is one of the most misunderstood attributes:

| `beforeInvocation` | Eviction Timing | On Method Exception |
|---|---|---|
| `false` (default) | After method successfully completes | Cache is **NOT** evicted (exception rolls back everything) |
| `true` | Before method executes | Cache IS evicted regardless of success or failure |

```java
// DEFAULT (afterInvocation): Evict only if delete succeeds
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id); // If this throws, cache is NOT evicted
}

// BEFORE: Evict before attempting delete — even if delete fails, cache is cleared
@CacheEvict(value = "products", key = "#id", beforeInvocation = true)
public void deleteProduct(Long id) {
    productRepository.deleteById(id); // If this throws, cache was already evicted
}
```

**When to use `beforeInvocation = true`:**
- When you want to guarantee cache eviction regardless of operation outcome
- When the method has side effects that should always invalidate the cache
- In cases where stale cache is worse than a cache miss

---

## 6. @Caching — Composing Multiple Operations

`@Caching` is a grouping annotation that lets you apply multiple caching annotations to a single method.

```java
@Service
public class ProductService {

    // Evict from multiple caches AND put in another on update
    @Caching(
        evict = {
            @CacheEvict(value = "productList", allEntries = true),
            @CacheEvict(value = "categoryProducts", key = "#product.categoryId")
        },
        put = {
            @CachePut(value = "products", key = "#product.id")
        }
    )
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // On delete: evict from product cache AND from the list cache
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "productList", allEntries = true)
    })
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

### Real-World @Caching Example

```java
@Service
public class ArticleService {

    // Reading an article caches it in multiple caches under different keys
    @Caching(cacheable = {
        @Cacheable(value = "articles", key = "#slug"),
        @Cacheable(value = "articlesById", key = "#root.args[0]") // This won't work as expected
    })
    public Article findBySlug(String slug) {
        return articleRepository.findBySlug(slug).orElseThrow();
    }

    // On publish, update the article cache and clear the list cache
    @Caching(
        put = { @CachePut(value = "articles", key = "#article.slug") },
        evict = { @CacheEvict(value = "articleList", allEntries = true) }
    )
    public Article publish(Article article) {
        article.setPublished(true);
        return articleRepository.save(article);
    }
}
```

---

## 7. Redis Caching with Spring Boot

Redis is the de facto standard for distributed caching in Java microservices. Spring Boot's auto-configuration makes setup straightforward.

### Dependencies

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

### Basic Configuration (application.yml)

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: yourpassword   # Optional
      timeout: 2000ms          # Connection timeout
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 1
          max-wait: 1000ms
  cache:
    type: redis
    redis:
      time-to-live: 600000    # 10 minutes in milliseconds
      cache-null-values: false
      key-prefix: "myapp:"    # Global key prefix
      use-key-prefix: true
```

### RedisCacheConfiguration — Java Config

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        // Default configuration for all caches
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))              // Default TTL: 10 minutes
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(jackson2JsonRedisSerializer())
            )
            .prefixCacheNameWith("myapp::")                // Key prefix strategy
            .disableCachingNullValues();                   // Don't cache nulls

        // Per-cache TTL overrides
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();

        cacheConfigurations.put("products",
            defaultConfig.entryTtl(Duration.ofHours(1)));

        cacheConfigurations.put("users",
            defaultConfig.entryTtl(Duration.ofMinutes(15)));

        cacheConfigurations.put("sessions",
            defaultConfig.entryTtl(Duration.ofHours(24)));

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .transactionAware()    // Participate in Spring transactions
            .build();
    }

    @Bean
    public Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // IMPORTANT: Include type information so deserialization knows the target class
        objectMapper.activateDefaultTyping(
            LaissezFaireSubTypeValidator.instance,
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );
        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        return new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);
    }
}
```

### RedisTemplate for Manual Cache Operations

```java
@Service
public class ProductCacheService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String PRODUCT_KEY_PREFIX = "product:";

    public ProductCacheService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void cacheProduct(Product product) {
        String key = PRODUCT_KEY_PREFIX + product.getId();
        redisTemplate.opsForValue().set(key, product, Duration.ofMinutes(30));
    }

    public Optional<Product> getCachedProduct(Long id) {
        String key = PRODUCT_KEY_PREFIX + id;
        Product product = (Product) redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(product);
    }

    public void evictProduct(Long id) {
        String key = PRODUCT_KEY_PREFIX + id;
        redisTemplate.delete(key);
    }

    // Scan pattern — use with caution in production (blocks Redis)
    public void evictAllProducts() {
        Set<String> keys = redisTemplate.keys(PRODUCT_KEY_PREFIX + "*");
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
}
```

> [!WARNING]
> `redisTemplate.keys("pattern*")` is an O(N) blocking operation. **Never use it in production** on large Redis instances. Use `SCAN` cursor-based iteration instead.

### Redis Key Prefix Strategy

Redis stores all keys in a flat namespace. Without prefixes, keys from different applications or environments collide.

```java
// Production best practice for key structure:
// {environment}::{app-name}::{cache-name}::{key}

RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
    .prefixCacheNameWith("prod::myapp::");
// Results in key: "prod::myapp::products::1"
```

### TTL Configuration Per-Cache via application.yml

```yaml
spring:
  cache:
    redis:
      time-to-live: 600000  # Default: 10 minutes
    cache-names:
      - products
      - users
      - sessions
```

For per-cache TTL in YAML (Spring Boot 2.7+):

```yaml
spring:
  cache:
    redis:
      time-to-live: 10m
    # Per-cache config requires Java @Bean configuration
```

---

### Serialization: JDK vs Jackson

```java
// Option 1: JDK Serialization (default) — NOT RECOMMENDED for production
// Pros: Zero config, handles any Serializable class
// Cons: Binary format (not human-readable), class-name tied, slower, larger payloads

// Option 2: Jackson JSON Serialization — RECOMMENDED
// Pros: Human-readable, faster, smaller payload, language-agnostic
// Cons: Requires proper ObjectMapper config, type information needed

// Configuring Jackson serializer
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory cf) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(cf);

    Jackson2JsonRedisSerializer<Object> serializer = jackson2JsonRedisSerializer();

    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(serializer);
    template.setHashKeySerializer(new StringRedisSerializer());
    template.setHashValueSerializer(serializer);
    template.afterPropertiesSet();
    return template;
}
```

---

## 8. Caffeine Cache

Caffeine is a high-performance, near-optimal **in-memory** caching library for single-node applications. It offers a much richer feature set than `ConcurrentHashMap`.

### Dependency

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <!-- Version managed by Spring Boot BOM -->
</dependency>
```

### Configuration via application.yml

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=500,expireAfterWrite=10m
    cache-names:
      - products
      - users
```

### Configuration via Java @Bean

```java
@Configuration
@EnableCaching
public class CaffeineCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(defaultCaffeineConfig());
        manager.setCacheNames(List.of("products", "users", "categories"));
        return manager;
    }

    private Caffeine<Object, Object> defaultCaffeineConfig() {
        return Caffeine.newBuilder()
            .maximumSize(1000)                    // Max 1000 entries
            .expireAfterWrite(10, TimeUnit.MINUTES) // Expire 10 min after write
            .recordStats();                       // Enable statistics
    }
}
```

### Per-Cache Caffeine Configuration

```java
@Configuration
@EnableCaching
public class CaffeineCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager manager = new SimpleCacheManager();

        List<CaffeineCache> caches = List.of(
            buildCache("products",   1000, 60),   // 1000 entries, 60 min TTL
            buildCache("users",       500, 15),   // 500 entries, 15 min TTL
            buildCache("categories",  100, 120),  // 100 entries, 120 min TTL
            buildCacheWithIdle("sessions", 1000, 30) // 30 min idle timeout
        );

        manager.setCaches(caches);
        return manager;
    }

    private CaffeineCache buildCache(String name, int maxSize, int ttlMinutes) {
        return new CaffeineCache(name,
            Caffeine.newBuilder()
                .maximumSize(maxSize)
                .expireAfterWrite(ttlMinutes, TimeUnit.MINUTES)
                .recordStats()
                .build()
        );
    }

    private CaffeineCache buildCacheWithIdle(String name, int maxSize, int idleMinutes) {
        return new CaffeineCache(name,
            Caffeine.newBuilder()
                .maximumSize(maxSize)
                .expireAfterAccess(idleMinutes, TimeUnit.MINUTES) // TTI
                .recordStats()
                .build()
        );
    }
}
```

### Eviction Policies Explained

#### Size-Based Eviction
Caffeine uses a **Window TinyLFU** algorithm (state-of-the-art). It approximates LFU while avoiding its historical bias problem.

```java
Caffeine.newBuilder()
    .maximumSize(1000)       // Evict when count exceeds 1000
    // OR
    .maximumWeight(10_000)   // Evict based on weight
    .weigher((key, value) -> ((Product) value).getDescription().length())
```

#### Time-Based Eviction

```java
// expireAfterWrite: Entry expires X time after it was WRITTEN (regardless of access)
// Good for: Data with a known staleness window (prices, exchange rates)
Caffeine.newBuilder().expireAfterWrite(10, TimeUnit.MINUTES)

// expireAfterAccess: Entry expires X time after last ACCESS (read or write)
// Good for: Session data, user preferences
Caffeine.newBuilder().expireAfterAccess(30, TimeUnit.MINUTES)

// Custom expiry: Different TTLs per entry
Caffeine.newBuilder().expireAfter(new Expiry<String, Product>() {
    @Override
    public long expireAfterCreate(String key, Product product, long currentTime) {
        // VIP products cached longer
        return product.isPremium()
            ? TimeUnit.HOURS.toNanos(1)
            : TimeUnit.MINUTES.toNanos(10);
    }
    @Override
    public long expireAfterUpdate(String key, Product product, long currentTime, long currentDuration) {
        return currentDuration; // Keep same TTL after update
    }
    @Override
    public long expireAfterRead(String key, Product product, long currentTime, long currentDuration) {
        return currentDuration; // Keep same TTL after read
    }
})
```

### Cache Statistics & Monitoring

```java
@Component
public class CacheMetricsLogger {

    private final CacheManager cacheManager;

    public CacheMetricsLogger(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @Scheduled(fixedDelay = 60_000) // Log every minute
    public void logCacheStats() {
        cacheManager.getCacheNames().forEach(cacheName -> {
            CaffeineCache cache = (CaffeineCache) cacheManager.getCache(cacheName);
            if (cache != null) {
                CacheStats stats = cache.getNativeCache().stats();
                log.info("Cache '{}' — HitRate: {:.2f}%, Hits: {}, Misses: {}, " +
                         "Evictions: {}, AvgLoadPenalty: {}ms",
                    cacheName,
                    stats.hitRate() * 100,
                    stats.hitCount(),
                    stats.missCount(),
                    stats.evictionCount(),
                    stats.averageLoadPenalty() / 1_000_000
                );
            }
        });
    }
}
```

### Caffeine with Spring Boot Actuator (Micrometer)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: caches, metrics
  metrics:
    cache:
      caffeine:
        enabled: true
```

Caffeine automatically integrates with Micrometer when `spring-boot-starter-actuator` is on the classpath.

---

## 9. Cache Patterns & Strategies

### Cache Stampede / Thundering Herd

When a popular cache entry expires, many concurrent requests all simultaneously experience a cache miss and all hit the database.

```
Time T: cache key "product:1" expires
T+1ms: 500 concurrent requests for "product:1" all find cache miss
T+1ms: 500 DB queries are issued simultaneously
       → DB overloaded, latency spikes, cascading failures possible
```

**Solutions:**

#### 1. Probabilistic Early Expiration (XFetch)

```java
@Service
public class SmartCacheService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final double BETA = 1.0; // Controls eagerness to recompute

    public Product getWithEarlyExpiry(Long id) {
        String key = "product:" + id;
        CacheEntry<Product> entry = getCacheEntry(key);

        if (entry == null || shouldRecompute(entry)) {
            Product product = productRepository.findById(id).orElseThrow();
            storeCacheEntry(key, product, Duration.ofMinutes(10));
            return product;
        }
        return entry.getValue();
    }

    private boolean shouldRecompute(CacheEntry<?> entry) {
        double ttlRemaining = entry.getExpiryEpoch() - System.currentTimeMillis() / 1000.0;
        double delta = entry.getLastFetchDurationMs() / 1000.0;
        // XFetch: recompute early if -delta * BETA * ln(random) > ttlRemaining
        return -delta * BETA * Math.log(Math.random()) > ttlRemaining;
    }
}
```

#### 2. Locking / Mutex

```java
// Using Redisson distributed lock
@Service
public class LockedCacheService {

    private final RedissonClient redissonClient;
    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;

    public Product getProductWithLock(Long id) {
        String cacheKey = "product:" + id;
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) return (Product) cached;

        // Only one thread reloads; others wait for lock release
        RLock lock = redissonClient.getLock("lock:" + cacheKey);
        try {
            lock.lock(5, TimeUnit.SECONDS);
            // Double-check inside lock (another thread may have loaded it)
            cached = redisTemplate.opsForValue().get(cacheKey);
            if (cached != null) return (Product) cached;

            Product product = productRepository.findById(id).orElseThrow();
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(10));
            return product;
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

#### 3. @Cacheable(sync = true) — Caffeine

```java
// For single-node scenarios with Caffeine — prevents stampede natively
@Cacheable(value = "products", sync = true)
public Product findById(Long id) {
    return productRepository.findById(id).orElseThrow();
}
```

---

### Cache Warming

Pre-populating the cache before it receives traffic, to prevent cold-start misses.

```java
@Component
public class CacheWarmingService {

    private final ProductRepository productRepository;
    private final ProductService productService;

    @EventListener(ApplicationReadyEvent.class)
    public void warmCaches() {
        log.info("Starting cache warm-up...");

        // Strategy 1: Load top-N most accessed items
        List<Long> topProductIds = productRepository.findTop100ByOrderByViewCountDesc()
            .stream().map(Product::getId).toList();

        topProductIds.forEach(id -> {
            try {
                productService.findById(id); // Triggers @Cacheable
            } catch (Exception e) {
                log.warn("Failed to warm cache for product {}", id, e);
            }
        });

        log.info("Cache warm-up complete for {} products", topProductIds.size());
    }
}
```

---

### Cache-Aside Full Implementation

```java
@Service
@Slf4j
public class ProductServiceWithCacheAside {

    private final ProductRepository repository;
    private final RedisTemplate<String, Product> redisTemplate;

    private static final String KEY_PREFIX = "product:v1:";
    private static final Duration DEFAULT_TTL = Duration.ofMinutes(30);

    public Product getProduct(Long id) {
        String key = KEY_PREFIX + id;

        // 1. Check cache
        try {
            Product cached = redisTemplate.opsForValue().get(key);
            if (cached != null) {
                log.debug("Cache HIT for product {}", id);
                return cached;
            }
        } catch (Exception e) {
            log.warn("Cache read failed for product {}. Falling through to DB.", id, e);
            // Cache failure should NOT break the application
        }

        // 2. Cache MISS — load from DB
        log.debug("Cache MISS for product {}", id);
        Product product = repository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));

        // 3. Store in cache (with error handling)
        try {
            redisTemplate.opsForValue().set(key, product, DEFAULT_TTL);
        } catch (Exception e) {
            log.warn("Cache write failed for product {}. Continuing without caching.", id, e);
        }

        return product;
    }

    public Product updateProduct(Product product) {
        Product saved = repository.save(product);
        String key = KEY_PREFIX + saved.getId();

        try {
            // Update cache to avoid stale reads
            redisTemplate.opsForValue().set(key, saved, DEFAULT_TTL);
        } catch (Exception e) {
            log.warn("Failed to update cache for product {}", saved.getId(), e);
            // Evict as fallback — next read will miss and refresh
            redisTemplate.delete(key);
        }

        return saved;
    }

    public void deleteProduct(Long id) {
        repository.deleteById(id);
        String key = KEY_PREFIX + id;
        try {
            redisTemplate.delete(key);
        } catch (Exception e) {
            log.warn("Failed to evict cache for deleted product {}", id, e);
        }
    }
}
```

---

## 10. Cache Invalidation Strategies

> *"There are only two hard problems in Computer Science: cache invalidation and naming things."* — Phil Karlton

### TTL-Based Expiration

The simplest strategy: every cache entry has a fixed time-to-live.

```java
// Via annotation — relies on RedisCacheConfiguration TTL
@Cacheable(value = "products")  // TTL set globally in RedisCacheConfiguration
public Product findById(Long id) { ... }

// Via programmatic RedisTemplate
redisTemplate.opsForValue().set("product:1", product, Duration.ofMinutes(10));
```

**Pros:** Simple, self-healing. **Cons:** Stale data until TTL expires. Choosing the right TTL is an art.

### Event-Driven Invalidation

Use application events or messaging to invalidate caches across nodes.

```java
// Publisher: After product update
@Service
public class ProductService {
    private final ApplicationEventPublisher eventPublisher;

    public Product updateProduct(Product product) {
        Product saved = productRepository.save(product);
        eventPublisher.publishEvent(new ProductUpdatedEvent(saved.getId()));
        return saved;
    }
}

// Listener: Invalidate cache on event
@Component
public class ProductCacheInvalidator {

    private final CacheManager cacheManager;

    @EventListener
    public void onProductUpdated(ProductUpdatedEvent event) {
        Cache cache = cacheManager.getCache("products");
        if (cache != null) {
            cache.evict(event.getProductId());
        }
    }
}
```

For **distributed** invalidation across microservices, use a message broker:

```java
// Publisher: After update, send invalidation message to Kafka/RabbitMQ
@Service
public class ProductService {
    private final KafkaTemplate<String, CacheInvalidationEvent> kafka;

    public Product updateProduct(Product product) {
        Product saved = productRepository.save(product);
        kafka.send("cache-invalidation",
            new CacheInvalidationEvent("products", saved.getId().toString()));
        return saved;
    }
}

// Consumer: Each service instance listens and clears its local cache
@KafkaListener(topics = "cache-invalidation")
public void handleCacheInvalidation(CacheInvalidationEvent event) {
    Cache cache = cacheManager.getCache(event.getCacheName());
    if (cache != null) {
        cache.evict(event.getKey());
    }
}
```

### Version-Based Keys

Embed a version in the cache key so stale data is effectively orphaned (not deleted, just unreachable).

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "'v' + @productVersionService.getVersion() + ':' + #id")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    public Product updateProduct(Product product) {
        Product saved = productRepository.save(product);
        // Increment version for this product — old cache entry becomes unreachable
        productVersionService.incrementVersion(saved.getId());
        return saved;
    }
}
```

**Pros:** No explicit delete needed, great for read-heavy systems.  
**Cons:** Orphaned entries accumulate until TTL expires; requires version tracking.

---

## 11. Same-Class Call Trap

This is one of the most common caching bugs. **`@Cacheable` does NOT work when the annotated method is called from within the same class.**

### Why?

Spring's caching is implemented via **AOP proxies**. When you inject a `@Service` bean, Spring gives you a proxy object. The proxy intercepts method calls and applies caching logic.

```
External call:
  Caller → [Proxy (intercepts call, checks cache)] → ProductService.findById()

Internal call (self-call):
  ProductService.anotherMethod() → this.findById() (BYPASSES PROXY — no caching!)
```

```java
@Service
public class ProductService {

    @Cacheable("products")
    public Product findById(Long id) {
        System.out.println("DB call for: " + id); // Always prints!
        return productRepository.findById(id).orElseThrow();
    }

    public Product findByIdAndLog(Long id) {
        // This calls this.findById() — NOT the proxy's findById()
        // @Cacheable is IGNORED here
        return this.findById(id); // BUG: cache is never used
    }
}
```

### Solutions

#### Solution 1: Self-Injection (ApplicationContext)

```java
@Service
public class ProductService {

    @Autowired
    private ApplicationContext context;

    @Cacheable("products")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    public Product findByIdAndLog(Long id) {
        // Get the proxy from the context, not 'this'
        ProductService proxy = context.getBean(ProductService.class);
        return proxy.findById(id); // Works! Goes through the proxy
    }
}
```

#### Solution 2: Self-Injection via @Autowired

```java
@Service
public class ProductService {

    @Autowired
    @Lazy  // Prevents circular dependency
    private ProductService self;

    @Cacheable("products")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    public Product findByIdAndLog(Long id) {
        return self.findById(id); // Goes through the proxy
    }
}
```

#### Solution 3: Restructure into a Separate Bean (Best Practice)

```java
// The cleanest solution: extract cached methods to a separate service
@Service
public class ProductCacheService {

    @Cacheable("products")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }
}

@Service
public class ProductService {

    @Autowired
    private ProductCacheService productCacheService; // Injecting the proxy bean

    public Product findByIdAndLog(Long id) {
        return productCacheService.findById(id); // Goes through proxy — works!
    }
}
```

> [!TIP]
> Restructuring into separate beans is the most maintainable and testable solution. It also improves separation of concerns.

---

## 12. Serialization with Redis

When storing objects in Redis, they must be serialized to bytes. Choosing the wrong serializer causes bugs that are hard to debug.

### Why Serializable?

Java's default JDK serialization requires classes to implement `java.io.Serializable`. If your cached object is not serializable, you'll get a `SerializationException` at runtime.

```java
// This will FAIL with JDK serialization
public class Product {   // Not Serializable!
    private Long id;
    private String name;
}

// Fix 1: Implement Serializable
public class Product implements Serializable {
    @Serial
    private static final long serialVersionUID = 1L;  // Version control
    private Long id;
    private String name;
}
```

### Serialization Strategies Comparison

| Serializer | Format | Human-Readable | Speed | Size | Class Required |
|---|---|---|---|---|---|
| `JdkSerializationRedisSerializer` | Binary | ❌ | Slow | Large | `Serializable` |
| `Jackson2JsonRedisSerializer` | JSON | ✅ | Fast | Small | None (POJO) |
| `GenericJackson2JsonRedisSerializer` | JSON + type info | ✅ | Fast | Medium | None |
| `StringRedisSerializer` | String | ✅ | Fastest | Smallest | N/A (strings only) |
| `OxmSerializer` | XML | ✅ | Slow | Large | None |

### Jackson Serialization Configuration

```java
@Configuration
public class RedisSerializationConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory cf) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(cf);

        // GenericJackson2JsonRedisSerializer stores type info in JSON
        // Allows polymorphic deserialization without specifying the target class
        GenericJackson2JsonRedisSerializer jsonSerializer =
            new GenericJackson2JsonRedisSerializer();

        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(jsonSerializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(jsonSerializer);

        return template;
    }
}
```

### Handling Class Changes with Cached Data

When you change a class (add/remove fields) that has existing cached instances, deserialization may fail.

```java
// Original class
public class Product implements Serializable {
    @Serial private static final long serialVersionUID = 1L;
    private Long id;
    private String name;
}

// After adding a field:
public class Product implements Serializable {
    @Serial private static final long serialVersionUID = 1L;  // SAME UID = compatible
    private Long id;
    private String name;
    private String description;  // New field — old cached data has no description
    // description will be null after deserialization of old data — handle this!
}

// If you change serialVersionUID, old cached data CANNOT be deserialized
// → All cache reads will throw InvalidClassException
// → Cache effectively invalidates itself (may be desirable on major schema changes)
```

**Best Practices for Evolving Cached Classes:**
1. Always use Jackson JSON serialization — adding fields with `@JsonIgnoreProperties(ignoreUnknown = true)` makes changes backward-compatible.
2. Use **version-based cache keys** to cleanly invalidate on schema change.
3. Implement `@JsonIgnoreProperties(ignoreUnknown = true)` on cached DTOs.

```java
@JsonIgnoreProperties(ignoreUnknown = true)  // Ignore new fields when reading old data
public class ProductDto {
    private Long id;
    private String name;
    // Adding fields here won't break deserialization of old cached values
}
```

---

## 13. Testing Cached Code

Testing caching behavior requires verifying both the happy path and the cache interaction itself.

### Unit Testing with Mock CacheManager

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock
    private ProductRepository productRepository;

    @Mock
    private CacheManager cacheManager;

    @Mock
    private Cache cache;

    @InjectMocks
    private ProductService productService;

    @Test
    void findById_shouldCallRepository_onCacheMiss() {
        Product product = new Product(1L, "Laptop");
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        Product result = productService.findById(1L);

        assertThat(result).isEqualTo(product);
        verify(productRepository, times(1)).findById(1L);
    }
}
```

### Integration Testing with @SpringBootTest

```java
@SpringBootTest
@TestPropertySource(properties = "spring.cache.type=simple")  // Use simple cache for tests
class ProductServiceIntegrationTest {

    @Autowired
    private ProductService productService;

    @Autowired
    private CacheManager cacheManager;

    @MockBean
    private ProductRepository productRepository;

    @BeforeEach
    void setup() {
        // Clear all caches before each test
        cacheManager.getCacheNames().forEach(name ->
            Objects.requireNonNull(cacheManager.getCache(name)).clear()
        );
    }

    @Test
    void findById_shouldCacheResultAfterFirstCall() {
        Product product = new Product(1L, "Laptop");
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        // First call — should hit repository
        productService.findById(1L);
        // Second call — should use cache
        productService.findById(1L);

        // Repository called exactly once despite two service calls
        verify(productRepository, times(1)).findById(1L);

        // Verify cache contains the entry
        Cache productsCache = cacheManager.getCache("products");
        assertThat(productsCache).isNotNull();
        assertThat(productsCache.get(1L, Product.class)).isEqualTo(product);
    }

    @Test
    void updateProduct_shouldRefreshCache() {
        Product original = new Product(1L, "Old Name");
        Product updated = new Product(1L, "New Name");

        when(productRepository.findById(1L)).thenReturn(Optional.of(original));
        when(productRepository.save(any())).thenReturn(updated);

        // Populate cache via @Cacheable
        productService.findById(1L);

        // Update via @CachePut
        productService.updateProduct(updated);

        // Verify cache now has updated value
        Cache cache = cacheManager.getCache("products");
        Product cachedProduct = cache.get(1L, Product.class);
        assertThat(cachedProduct.getName()).isEqualTo("New Name");
    }

    @Test
    void deleteProduct_shouldEvictFromCache() {
        Product product = new Product(1L, "Laptop");
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));
        doNothing().when(productRepository).deleteById(1L);

        // Populate cache
        productService.findById(1L);
        assertThat(cacheManager.getCache("products").get(1L)).isNotNull();

        // Delete — should evict
        productService.deleteProduct(1L);
        assertThat(cacheManager.getCache("products").get(1L)).isNull();
    }
}
```

### Testing Redis Cache with Embedded Redis (Testcontainers)

```java
@SpringBootTest
@Testcontainers
class RedisCacheIntegrationTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    private ProductService productService;

    @MockBean
    private ProductRepository productRepository;

    @Test
    void shouldCacheInRedis() {
        Product product = new Product(1L, "Laptop");
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        productService.findById(1L);
        productService.findById(1L);

        verify(productRepository, times(1)).findById(1L); // Only one DB call
    }
}
```

---

## ⚠️ Interview Traps & Gotchas

### Trap 1: @Cacheable on Private Methods

```java
@Service
public class ProductService {

    @Cacheable("products")  // ❌ DOES NOTHING — private method cannot be proxied
    private Product internalFind(Long id) {
        return productRepository.findById(id).orElseThrow();
    }
}
```

**Why?** Spring AOP proxies only intercept `public` method calls on Spring beans. Private methods are not visible to the proxy.

**Fix:** Make the method `public`, or use AspectJ weaving (compile-time weaving that doesn't rely on proxies).

---

### Trap 2: @Cacheable on Same-Class Call

(See Section 11 in detail.)

```java
// ❌ Caching won't work — internal call bypasses proxy
public void processProducts() {
    this.findById(1L);  // No caching!
}

// ✅ Fix: use self-injected proxy or separate bean
```

---

### Trap 3: Null Return Value Caching

```java
@Cacheable("products")
public Product findById(Long id) {
    return productRepository.findById(id).orElse(null); // Returns null if missing
}
// BUG: null is cached. Even after the product is created, this returns null from cache.
```

**Fix:** Use `unless = "#result == null"` or throw an exception instead of returning null.

---

### Trap 4: Cache Key Collision

```java
@Service
public class ProductService {
    @Cacheable("data")
    public Product getProduct(Long id) { ... }  // Key: 1L
}

@Service
public class OrderService {
    @Cacheable("data")   // SAME cache name!
    public Order getOrder(Long id) { ... }     // Key: 1L — COLLIDES with product!
}

// getOrder(1L) might return a Product object → ClassCastException!
```

**Fix:** Use unique, descriptive cache names per entity: `"products"`, `"orders"`, etc.

---

### Trap 5: @CacheEvict beforeInvocation Misconception

```java
// Most people expect this to "protect" the eviction from exceptions
@CacheEvict(value = "products", key = "#id")  // beforeInvocation = false (default)
public void deleteProduct(Long id) {
    productRepository.deleteById(id);  // If this throws...
    // Cache is NOT evicted — method failed, so cache is preserved (correct behavior)
}

// But this ALWAYS evicts, even if delete fails
@CacheEvict(value = "products", key = "#id", beforeInvocation = true)
public void deleteProduct(Long id) {
    productRepository.deleteById(id);  // If this throws...
    // Cache was already evicted — now cache is gone but DB record still exists!
    // Next read will be a cache miss and load the "deleted" product back into cache
}
```

---

### Trap 6: Which CacheManager if Not Configured?

If you have `@EnableCaching` but **no** explicit `CacheManager` bean and **no** cache provider on the classpath, Spring Boot auto-configures `ConcurrentMapCacheManager`. This is:
- An in-memory `ConcurrentHashMap`-based implementation
- **No TTL support** — entries never expire
- **No size limit** — can cause memory leaks in production
- Only suitable for development/testing

> [!CAUTION]
> `ConcurrentMapCacheManager` in production is a memory leak waiting to happen. Always configure a proper cache provider with explicit TTL and size bounds.

---

### Trap 7: Redis Down — What Happens to @Cacheable?

By default, if Redis is down, `@Cacheable` throws a `RedisConnectionFailureException` and the method call **fails**. The underlying method body is NOT executed as a fallback.

```java
// By default: Redis down → exception thrown → request fails
@Cacheable("products")
public Product findById(Long id) { ... }
```

**Solution 1: Configure error handler**

```java
@Configuration
@EnableCaching
public class ResilientCacheConfig {

    @Bean
    public CacheErrorHandler errorHandler() {
        return new SimpleCacheErrorHandler() {
            @Override
            public void handleCacheGetError(RuntimeException e, Cache cache, Object key) {
                log.warn("Cache GET error on cache '{}' for key '{}': {}", 
                    cache.getName(), key, e.getMessage());
                // DO NOT rethrow — allow the method to execute (cache miss fallback)
            }

            @Override
            public void handleCachePutError(RuntimeException e, Cache cache, Object key, Object value) {
                log.warn("Cache PUT error: {}", e.getMessage());
                // Don't rethrow — the operation succeeded, just caching failed
            }

            @Override
            public void handleCacheEvictError(RuntimeException e, Cache cache, Object key) {
                log.error("Cache EVICT error for key '{}': {}", key, e.getMessage());
            }
        };
    }
}

// This bean must be returned by a CachingConfigurerSupport subclass
@Configuration
@EnableCaching
public class CachingConfig extends CachingConfigurerSupport {

    @Override
    public CacheErrorHandler errorHandler() {
        return new ResilientCacheErrorHandler(); // Logs instead of throwing
    }
}
```

**Solution 2: Circuit breaker pattern around Redis operations.**

---

### Trap 8: @Cacheable + @Transactional Interaction

```java
// When using @Transactional + @Cacheable together, cache is populated BEFORE
// the transaction commits. If the transaction rolls back, the cache has stale data.

@Transactional
@Cacheable("products")
public Product findById(Long id) { ... }

// Fix: Use RedisCacheManager with transactionAware(true)
// This delays cache writes until the transaction commits
RedisCacheManager.builder(connectionFactory)
    .transactionAware()  // Cache writes participate in Spring transactions
    .build();
```

---

### Trap 9: @Cacheable with Pageable

```java
// WRONG: Pageable doesn't have a stable hashCode/equals in all versions
@Cacheable("products")
public Page<Product> findAll(Pageable pageable) { ... }

// RIGHT: Extract the relevant parts for the key
@Cacheable(value = "products", key = "#pageable.pageNumber + '-' + #pageable.pageSize + '-' + #pageable.sort")
public Page<Product> findAll(Pageable pageable) { ... }
```

> [!WARNING]
> `Page` objects contain proxy references and should NOT be serialized to Redis. Cache the `List<Product>` content instead, or use a dedicated DTO.

---

## 🔍 Deep Dive — Internal Mechanics

### How Spring AOP Caching Works

When you annotate a method with `@Cacheable`, Spring creates a **proxy** around your bean using either JDK dynamic proxies (if the class implements an interface) or CGLIB proxies (otherwise).

```
Application Context Startup:
  1. @EnableCaching imports CachingConfigurationSelector
  2. CachingConfigurationSelector registers ProxyCachingConfiguration (for proxy mode)
  3. ProxyCachingConfiguration creates:
     a. BeanFactoryCacheOperationSourceAdvisor (the AOP advisor)
     b. CacheInterceptor (the AOP advice/interceptor)
     c. AnnotationCacheOperationSource (parses @Cacheable, @CachePut, etc.)

Runtime (first invocation of @Cacheable method):
  1. Caller invokes proxy.findById(1L)
  2. CGLIB proxy intercepts the call
  3. CacheInterceptor.invoke() is called
  4. AnnotationCacheOperationSource parses @Cacheable attributes
  5. KeyGenerator generates the cache key
  6. CacheManager.getCache("products") retrieves the Cache object
  7. Cache.get(key) checks for a value:
     a. HIT: return cached value — method body is NOT invoked
     b. MISS: invoke the actual method, get result, Cache.put(key, result), return result
```

### The CacheInterceptor Source (Simplified)

```java
// Simplified view of CacheInterceptor logic
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        CacheOperationInvoker aopAllianceInvoker =
            () -> invokeOperation(invocation::proceed);

        Object target = invocation.getThis();
        return execute(aopAllianceInvoker, target, method, invocation.getArguments());
    }
}

// Inside CacheAspectSupport.execute():
// 1. Find all cache operations for this method
// 2. For @CacheEvict with beforeInvocation=true: evict NOW
// 3. For @Cacheable: check cache
//    - If hit: return cached value (invocation never called)
//    - If miss: continue to step 4
// 4. Invoke the method
// 5. For @CacheEvict with beforeInvocation=false: evict NOW
// 6. For @CachePut: put result in cache
// 7. For @Cacheable (miss): put result in cache
```

### Why JDK Proxy vs CGLIB?

| Proxy Type | Requires | How Created |
|---|---|---|
| JDK Dynamic Proxy | Class implements an interface | `Proxy.newProxyInstance(...)` |
| CGLIB | No interface needed | Generates a subclass bytecode at runtime |

Spring Boot defaults to CGLIB (`proxyTargetClass = true`). This means:
- `@Cacheable` on final methods → **CGLIB cannot subclass and override final methods** → caching silently skipped!

```java
@Service
public class ProductService {

    @Cacheable("products")
    public final Product findById(Long id) { ... } // ❌ final — CGLIB cannot override!
}
```

---

## 📝 Summary & Key Takeaways

| Annotation | Executes Method? | Updates Cache? | Deletes Cache? | Primary Use |
|---|---|---|---|---|
| `@Cacheable` | Only on miss | On miss | ❌ | Read operations |
| `@CachePut` | Always | Always | ❌ | Write/update operations |
| `@CacheEvict` | Always | ❌ | ✅ | Delete operations |
| `@Caching` | Depends | Depends | Depends | Complex multi-cache operations |

### Key Principles

1. **`@EnableCaching` is mandatory** — without it, all caching annotations are no-ops.
2. **Caching works via AOP proxies** — same-class calls and private methods bypass the proxy.
3. **Default `ConcurrentMapCacheManager` has no TTL** — always configure a proper provider for production.
4. **Always handle null results explicitly** — use `unless = "#result == null"` or throw exceptions.
5. **Use distinct cache names** — prevent key collisions between different entity types.
6. **`beforeInvocation`** determines whether eviction is protected by the method's success.
7. **Redis serialization matters** — prefer Jackson JSON over JDK serialization.
8. **Always handle Redis failures gracefully** — use `CacheErrorHandler` to prevent cache outages from breaking your API.
9. **Cache stampede is real** — use `sync = true` (Caffeine) or distributed locks (Redis) for high-traffic keys.
10. **Test caching behavior explicitly** — verify both hit and miss paths with `verify(repository, times(1))`.

---

## ❓ Practice Questions

**Conceptual:**
1. Explain the difference between `condition` and `unless` in `@Cacheable`. Can `condition` access `#result`? Can `unless`?
2. What happens if you annotate a private method with `@Cacheable`? Why?
3. Why doesn't `@Cacheable` work when a method calls another method in the same class?
4. What is a cache stampede? How does `sync = true` help, and why doesn't it fully solve the problem with Redis?
5. Explain write-through vs write-behind caching patterns. What are the trade-offs?
6. What's the difference between `expireAfterWrite` and `expireAfterAccess` in Caffeine?
7. What happens when Redis goes down in a Spring Boot application using `@Cacheable`? How do you make it resilient?

**Scenario-Based:**
8. You have a `@Cacheable` method that returns `null` for non-existent products. Users report that even after a product is created, the API keeps returning null. What's the bug and how do you fix it?
9. You have two services that both use `@Cacheable("data")` — one caches `Product` objects and another caches `Order` objects, both with Long ID keys. Users are getting `ClassCastException`. What's wrong?
10. A senior engineer complains that the cache metrics show a 0% hit rate even though `@Cacheable` is on the service methods. What are three possible root causes you'd investigate?
11. You need to cache a method's result with different TTLs based on the returned object's properties. How would you implement this?
12. You're migrating a service from Caffeine to Redis. What serialization changes do you need to make? What are the risks with existing classes?

**Code Review:**
13. Review this code — what's wrong?
    ```java
    @CacheEvict(value = "users", allEntries = true, beforeInvocation = true)
    @Transactional
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
    ```
14. Is there a bug in this code?
    ```java
    @Service
    public class ReportService {
        @Cacheable("reports")
        public Report generate(int year) { ... }

        public Report generateAndEmail(int year) {
            Report r = this.generate(year);  // Sends email with this report
            emailService.send(r);
            return r;
        }
    }
    ```
15. What's the problem with caching a `Page<Product>` object in Redis, and how would you fix it?
