# Rate Limiting & Traffic Management in Spring Boot

> A production-grade deep dive into protecting your APIs, managing traffic, and building resilient distributed systems.

---

## Table of Contents

1. [Why Rate Limiting?](#1-why-rate-limiting)
2. [Rate Limiting Algorithms](#2-rate-limiting-algorithms)
3. [Filter-Based Rate Limiting](#3-filter-based-rate-limiting)
4. [Bucket4j — The Go-To Library](#4-bucket4j--the-go-to-library)
5. [Spring Cloud Gateway Rate Limiting](#5-spring-cloud-gateway-rate-limiting)
6. [Implementing Rate Limit Tiers](#6-implementing-rate-limit-tiers)
7. [Rate Limit Headers & Client Communication](#7-rate-limit-headers--client-communication)
8. [Distributed Rate Limiting Architecture](#8-distributed-rate-limiting-architecture)
9. [Circuit Breaker with Resilience4j](#9-circuit-breaker-with-resilience4j)
10. [Load Balancing Basics](#10-load-balancing-basics)
11. [Throttling vs Rate Limiting](#11-throttling-vs-rate-limiting)
12. [⚠️ Interview Traps & Gotchas](#12-️-interview-traps--gotchas)
13. [📝 Summary & Key Takeaways](#13--summary--key-takeaways)
14. [❓ Practice Questions](#14--practice-questions)

---

## 1. Why Rate Limiting?

Rate limiting is a **traffic control technique** that restricts the number of requests a client can make to an API within a given time window. It is one of the most critical mechanisms in modern API design.

### 1.1 Protection from Abuse, DDoS & Scraping

Without rate limiting, your API is completely open to:

| Threat | Description | Impact |
|---|---|---|
| **DDoS** | Distributed Denial of Service — flooding your servers with requests | Service outage |
| **Brute Force** | Repeated login or credential-stuffing attempts | Account compromise |
| **Web Scraping** | Automated bots harvesting all your data | IP/data theft, resource drain |
| **API Abuse** | Misuse by legitimate users who loop incorrectly | Degraded service for all |
| **Resource Exhaustion** | Downstream DB or third-party API hammered | Cascading failures |

A DDoS attack without rate limiting can cost thousands of dollars in cloud bills in minutes and take down production systems for hours.

### 1.2 API Monetization Tiers

Rate limiting is the backbone of **API-as-a-product** business models:

```
Free Tier:       100 requests/hour
Starter Tier:    1,000 requests/hour
Pro Tier:        10,000 requests/hour
Enterprise Tier: Unlimited (custom SLA)
```

Companies like Stripe, Twilio, OpenAI, and GitHub all use rate limiting to enforce these tiers. Rate limiting thus becomes a **revenue mechanism**, not just a protection mechanism.

### 1.3 Fair Usage Policies

Without rate limiting, a single power user can consume 90% of your infrastructure, starving all other users. Rate limiting enforces **fairness** — every user gets their equitable share of resources.

Consider an analogy: a highway without speed limits and with unlimited on-ramps causes gridlock. Rate limiting is the on-ramp metering system.

### 1.4 Backend Protection

Even internal APIs need rate limiting:

- Protect downstream databases from query storms
- Shield third-party APIs with external call limits (you pay per call)
- Prevent one microservice from overwhelming another during a traffic spike
- Allow graceful degradation instead of hard failures

> [!IMPORTANT]
> Rate limiting should be applied at **multiple layers**: the API Gateway, application layer, and even at the database connection pool level. Defense in depth.

---

## 2. Rate Limiting Algorithms

Understanding the algorithms is critical for interviews and for choosing the right implementation. Each has distinct trade-offs.

### 2.1 Token Bucket Algorithm

**How it works:**

Imagine a bucket that holds tokens. Tokens are added at a fixed rate (the *refill rate*). Each incoming request consumes one token. If there are tokens available, the request is allowed. If the bucket is empty, the request is denied (429) or queued.

```
┌─────────────────────────────────────────────────────────┐
│                    TOKEN BUCKET                         │
│                                                         │
│  Refill Rate: 10 tokens/sec                             │
│  Capacity:    100 tokens (max burst)                    │
│                                                         │
│   ┌──────────────────────┐                              │
│   │  ●●●●●●●●●●●●●●●●●● │ ← 20 tokens currently       │
│   │  ●●●●●●●●●●●●●●●●●● │                              │
│   └──────────────────────┘                              │
│         ↑                    ↓                          │
│   +10 tokens/sec         -1 token/request               │
│                                                         │
│   Request arrives → Tokens available? → ALLOW           │
│                   → Bucket empty?     → DENY (429)      │
└─────────────────────────────────────────────────────────┘
```

**Key Properties:**
- Allows **burst traffic** up to the bucket capacity
- Smoothly handles varying request rates
- Token count can be stored as a floating point for sub-second precision

**Pros:**
- ✅ Handles burst traffic naturally (bucket fills up during idle periods)
- ✅ Simple to implement
- ✅ Works well for most API rate limiting scenarios

**Cons:**
- ❌ Requires tracking last refill timestamp per client
- ❌ Slight implementation complexity vs fixed window

---

### 2.2 Leaky Bucket Algorithm

**How it works:**

Requests enter a **queue** (the bucket). The queue drains at a **constant rate** regardless of how many requests are in it. If the queue is full, new requests overflow and are rejected.

```
┌─────────────────────────────────────────────────────────┐
│                    LEAKY BUCKET                         │
│                                                         │
│  Incoming: Variable rate    Outgoing: Fixed 10 req/sec  │
│                                                         │
│  ┌────────────────┐                                     │
│  │ req req req req│ ← Queue (capacity = 50)             │
│  │ req req req req│                                     │
│  └───────┬────────┘                                     │
│          │ drip at constant rate                        │
│          ▼                                              │
│   [PROCESSED] → 10/sec always                          │
│                                                         │
│  Queue full? → DROP request (429)                       │
└─────────────────────────────────────────────────────────┘
```

**Key Difference from Token Bucket:**

| Aspect | Token Bucket | Leaky Bucket |
|---|---|---|
| **Burst Handling** | ✅ Allows bursts up to capacity | ❌ Output is always constant rate |
| **Use Case** | APIs needing burst tolerance | Network traffic shaping |
| **Output Rate** | Variable (up to max) | Always constant |
| **Implementation** | Counter + timestamp | Queue-based |

Leaky bucket is preferred in **network traffic shaping** (e.g., QoS policies) where you need a guaranteed constant output rate. Token bucket is preferred for **API rate limiting**.

---

### 2.3 Fixed Window Counter

**How it works:**

Time is divided into fixed windows (e.g., 1 minute). A counter tracks requests in the current window. When the counter exceeds the limit, requests are rejected until the next window starts.

```
Window: 0s ──────────── 60s | 60s ──────────── 120s
         [Request 1]          [Request 1]
         [Request 2]          [Request 2]
         ...                  ...
         [Request 100]        [Request 100]
         [Request 101] → 429  ...
```

**The Boundary Problem (Race Condition at Window Edge):**

```
Window 1: 50s ──────────── 60s   Window 2: 60s ──────────── 70s
               [90 requests]              [90 requests in first 10s]
```

A client can send 90 requests in the last 10 seconds of window 1, then 90 requests in the first 10 seconds of window 2 — effectively 180 requests in 20 seconds despite a 100/minute limit. This is the **boundary problem**.

**Pros:** ✅ Extremely simple, O(1) memory per client
**Cons:** ❌ Boundary problem, allows 2x burst at window boundaries

---

### 2.4 Sliding Window Log

**How it works:**

Keeps a **timestamped log** of every request. When a new request arrives, it removes all entries older than the window size, then checks if the count exceeds the limit.

```
Window Size: 60 seconds
Limit: 100 requests

Log for user "alice":
[10:00:01, 10:00:03, 10:00:15, ..., 10:00:59, 10:01:00]
                                         ↑
                         New request at 10:01:02
                         Remove entries < 10:00:02
                         Count remaining entries
                         If count < 100 → ALLOW
```

**Pros:** ✅ Perfectly accurate, no boundary problem
**Cons:** ❌ Memory-intensive (stores every request timestamp), O(n) cleanup

---

### 2.5 Sliding Window Counter (Hybrid)

**How it works:**

Combines fixed window with a weighted calculation. Uses two window counters and interpolates based on position within the current window.

```
Previous Window:  [count=80]  (full 60s)
Current Window:   [count=40]  (started 30s ago, 50% complete)

Effective count = (80 × 0.5) + 40 = 40 + 40 = 80
                          ↑ weight = (1 - elapsed/window)
```

If the effective count ≤ 100, the request is allowed. This approximation is accurate enough for most use cases and requires only 2 counters per client.

**Pros:** ✅ Good accuracy, O(1) memory, no boundary problem
**Cons:** ❌ Approximate (not perfectly accurate)

---

### 2.6 Algorithm Comparison Table

| Algorithm | Accuracy | Memory | Burst Handling | Complexity | Best For |
|---|---|---|---|---|---|
| **Token Bucket** | High | O(1) | ✅ Yes | Medium | API rate limiting |
| **Leaky Bucket** | High | O(n) queue | ❌ No | Medium | Traffic shaping |
| **Fixed Window** | Low (boundary bug) | O(1) | ❌ Boundary burst | Low | Simple cases |
| **Sliding Window Log** | Perfect | O(n) | N/A | High | High-accuracy needs |
| **Sliding Window Counter** | High (approx.) | O(1) | Partial | Medium | Production APIs |

> [!TIP]
> **Bucket4j** uses the Token Bucket algorithm. **Redis** + **Lua scripts** typically implement Sliding Window Counter. **Spring Cloud Gateway** uses Token Bucket via Redis. For most production use cases, Token Bucket (via Bucket4j) is the right choice.

---

## 3. Filter-Based Rate Limiting

The simplest approach: implement rate limiting as a `Filter` or `HandlerInterceptor`. This is **per-instance** (not distributed) and useful for understanding the fundamentals.

### 3.1 In-Memory Rate Limiter with ConcurrentHashMap

```java
package com.example.ratelimit.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Simple in-memory Fixed Window rate limiter.
 * WARNING: This is per-instance only. Not suitable for distributed deployments.
 * Use Bucket4j + Redis for production distributed rate limiting.
 */
@Component
public class InMemoryRateLimitFilter implements Filter {

    private static final int MAX_REQUESTS_PER_WINDOW = 100;
    private static final long WINDOW_SIZE_MS = 60_000L; // 1 minute

    // Key: client identifier (IP), Value: [requestCount, windowStartMs]
    private final ConcurrentHashMap<String, long[]> requestCounts = new ConcurrentHashMap<>();

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String clientId = extractClientId(httpRequest);
        long now = Instant.now().toEpochMilli();

        // computeIfAbsent is atomic for map entry creation
        long[] windowData = requestCounts.computeIfAbsent(
            clientId, k -> new long[]{0L, now}
        );

        synchronized (windowData) {
            long windowStart = windowData[1];
            long requestCount = windowData[0];

            // If the current window has expired, reset
            if (now - windowStart > WINDOW_SIZE_MS) {
                windowData[0] = 1L;
                windowData[1] = now;
                chain.doFilter(request, response);
                return;
            }

            // Check against limit
            if (requestCount >= MAX_REQUESTS_PER_WINDOW) {
                long resetTimeSeconds = (windowStart + WINDOW_SIZE_MS - now) / 1000;
                httpResponse.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
                httpResponse.setHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS_PER_WINDOW));
                httpResponse.setHeader("X-RateLimit-Remaining", "0");
                httpResponse.setHeader("X-RateLimit-Reset", String.valueOf(
                    (windowStart + WINDOW_SIZE_MS) / 1000
                ));
                httpResponse.setHeader("Retry-After", String.valueOf(resetTimeSeconds));
                httpResponse.setContentType("application/json");
                httpResponse.getWriter().write("""
                    {
                        "error": "Too Many Requests",
                        "message": "Rate limit exceeded. Please retry after %d seconds.",
                        "retryAfter": %d
                    }
                    """.formatted(resetTimeSeconds, resetTimeSeconds));
                return;
            }

            // Allow request, increment counter
            windowData[0]++;
            long remaining = MAX_REQUESTS_PER_WINDOW - windowData[0];
            httpResponse.setHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS_PER_WINDOW));
            httpResponse.setHeader("X-RateLimit-Remaining", String.valueOf(remaining));
            httpResponse.setHeader("X-RateLimit-Reset",
                String.valueOf((windowStart + WINDOW_SIZE_MS) / 1000));
        }

        chain.doFilter(request, response);
    }

    private String extractClientId(HttpServletRequest request) {
        // Check X-Forwarded-For first (client behind proxy/load balancer)
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isBlank()) {
            return xForwardedFor.split(",")[0].trim(); // First IP in chain
        }
        return request.getRemoteAddr();
    }
}
```

### 3.2 Registering the Filter with Ordering

```java
package com.example.ratelimit.config;

import com.example.ratelimit.filter.InMemoryRateLimitFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<InMemoryRateLimitFilter> rateLimitFilter(
            InMemoryRateLimitFilter filter) {
        FilterRegistrationBean<InMemoryRateLimitFilter> registration =
            new FilterRegistrationBean<>();
        registration.setFilter(filter);
        registration.addUrlPatterns("/api/*");  // Only rate-limit API endpoints
        registration.setOrder(1);               // Run before other filters
        return registration;
    }
}
```

> [!WARNING]
> The `synchronized (windowData)` block creates a bottleneck under high concurrency. For production, use Bucket4j which is built on lock-free algorithms internally. The in-memory approach also fails completely in multi-instance deployments — each instance has its own counter.

---

## 4. Bucket4j — The Go-To Library

**Bucket4j** is the most popular Java library for rate limiting. It implements the **Token Bucket** algorithm and supports:
- Local (in-memory) rate limiting
- Distributed rate limiting with Redis, Hazelcast, Infinispan, etc.
- Spring Boot autoconfigure integration
- Asynchronous operation

### 4.1 Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Core Bucket4j -->
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-core</artifactId>
        <version>8.10.1</version>
    </dependency>

    <!-- Bucket4j Spring Boot Starter (autoconfigure) -->
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-spring-boot-starter</artifactId>
        <version>0.10.0</version>
    </dependency>

    <!-- For distributed rate limiting with Redis -->
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-redis</artifactId>
        <version>8.10.1</version>
    </dependency>

    <!-- Lettuce Redis client (already in spring-boot-starter-data-redis) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

### 4.2 Local Rate Limiting with Bucket4j

```java
package com.example.ratelimit.service;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class LocalRateLimitService {

    // One Bucket per client — stored in memory
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    /**
     * Returns the bucket for a given key (IP, userId, apiKey, etc.).
     * Creates one if it doesn't exist.
     */
    public Bucket resolveBucket(String key) {
        return buckets.computeIfAbsent(key, this::createNewBucket);
    }

    private Bucket createNewBucket(String key) {
        // Standard limit: 100 requests per minute
        Bandwidth standardLimit = Bandwidth.builder()
            .capacity(100)
            .refillGreedy(100, Duration.ofMinutes(1))
            .build();

        // Burst limit: allow 20 requests instantly at start
        Bandwidth burstLimit = Bandwidth.builder()
            .capacity(20)
            .refillIntervally(20, Duration.ofSeconds(10))
            .initialTokens(20) // Start with full burst capacity
            .build();

        return Bucket.builder()
            .addLimit(standardLimit)
            .addLimit(burstLimit) // Both limits must be satisfied
            .build();
    }

    public boolean tryConsume(String key) {
        return resolveBucket(key).tryConsume(1);
    }

    public boolean tryConsume(String key, long tokens) {
        return resolveBucket(key).tryConsume(tokens);
    }

    public long getAvailableTokens(String key) {
        return resolveBucket(key).getAvailableTokens();
    }
}
```

### 4.3 Distributed Rate Limiting with Bucket4j + Redis

This is the **production-ready** approach. Redis stores the bucket state, ensuring all application instances share the same rate limit.

```java
package com.example.ratelimit.config;

import io.github.bucket4j.distributed.proxy.ProxyManager;
import io.github.bucket4j.redis.lettuce.cas.LettuceBasedProxyManager;
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.codec.ByteArrayCodec;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;

@Configuration
public class Bucket4jRedisConfig {

    @Bean
    public ProxyManager<String> lettuceBasedProxyManager(
            LettuceConnectionFactory lettuceConnectionFactory) {

        // Get the underlying Lettuce client
        RedisClient redisClient = (RedisClient)
            lettuceConnectionFactory.getNativeClient();

        StatefulRedisConnection<byte[], byte[]> connection =
            redisClient.connect(ByteArrayCodec.INSTANCE);

        return LettuceBasedProxyManager.builderFor(connection)
            .withExpirationAfterWriteStrategy(
                // TTL for bucket state in Redis
                io.github.bucket4j.distributed.ExpirationAfterWriteStrategy
                    .basedOnTimeForRefillingBucketUpToMax(Duration.ofMinutes(5))
            )
            .build();
    }
}
```

```java
package com.example.ratelimit.service;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.BucketConfiguration;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

@Service
public class DistributedRateLimitService {

    private final ProxyManager<String> proxyManager;

    public DistributedRateLimitService(ProxyManager<String> proxyManager) {
        this.proxyManager = proxyManager;
    }

    /**
     * Returns a distributed Bucket backed by Redis.
     * The same bucket is shared across ALL application instances.
     */
    public io.github.bucket4j.Bucket resolveBucket(String key, RateLimitTier tier) {
        Supplier<BucketConfiguration> configSupplier = () -> BucketConfiguration.builder()
            .addLimit(tier.getBandwidth())
            .build();

        // proxyManager.builder() is thread-safe and returns the existing bucket
        // from Redis, or creates a new one atomically
        return proxyManager.builder()
            .build(key, configSupplier);
    }
}
```

```java
package com.example.ratelimit.service;

import io.github.bucket4j.Bandwidth;

import java.time.Duration;

public enum RateLimitTier {

    FREE(Bandwidth.builder()
        .capacity(100)
        .refillGreedy(100, Duration.ofHours(1))
        .build()),

    STARTER(Bandwidth.builder()
        .capacity(1_000)
        .refillGreedy(1_000, Duration.ofHours(1))
        .build()),

    PRO(Bandwidth.builder()
        .capacity(10_000)
        .refillGreedy(10_000, Duration.ofHours(1))
        .build()),

    ENTERPRISE(Bandwidth.builder()
        .capacity(Long.MAX_VALUE / 2) // Effectively unlimited
        .refillGreedy(Long.MAX_VALUE / 2, Duration.ofHours(1))
        .build());

    private final Bandwidth bandwidth;

    RateLimitTier(Bandwidth bandwidth) {
        this.bandwidth = bandwidth;
    }

    public Bandwidth getBandwidth() {
        return bandwidth;
    }
}
```

### 4.4 Complete Production-Ready Rate Limiting Filter

```java
package com.example.ratelimit.filter;

import com.example.ratelimit.service.DistributedRateLimitService;
import com.example.ratelimit.service.RateLimitTier;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.ConsumptionProbe;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.security.Principal;

@Component
public class RateLimitingFilter implements Filter {

    private final DistributedRateLimitService rateLimitService;

    public RateLimitingFilter(DistributedRateLimitService rateLimitService) {
        this.rateLimitService = rateLimitService;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // 1. Determine WHO is making the request
        String clientKey = resolveClientKey(httpRequest);

        // 2. Determine their tier
        RateLimitTier tier = resolveClientTier(httpRequest);

        // 3. Get their distributed bucket
        Bucket bucket = rateLimitService.resolveBucket(clientKey, tier);

        // 4. Try to consume a token — ConsumptionProbe gives us detailed info
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        // 5. Set standard rate limit headers (always, not just on rejection)
        long limit = tier.getBandwidth().getCapacity();
        httpResponse.setHeader("X-RateLimit-Limit", String.valueOf(limit));

        if (probe.isConsumed()) {
            // Request allowed
            httpResponse.setHeader("X-RateLimit-Remaining",
                String.valueOf(probe.getRemainingTokens()));
            httpResponse.setHeader("X-RateLimit-Reset",
                String.valueOf(System.currentTimeMillis() / 1000 +
                    probe.getNanosToWaitForRefill() / 1_000_000_000));

            chain.doFilter(request, response);
        } else {
            // Request denied — 429 Too Many Requests
            long waitSeconds = probe.getNanosToWaitForRefill() / 1_000_000_000;

            httpResponse.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            httpResponse.setContentType(MediaType.APPLICATION_JSON_VALUE);
            httpResponse.setHeader("X-RateLimit-Remaining", "0");
            httpResponse.setHeader("X-RateLimit-Reset",
                String.valueOf(System.currentTimeMillis() / 1000 + waitSeconds));
            httpResponse.setHeader("Retry-After", String.valueOf(waitSeconds));

            String errorBody = """
                {
                    "status": 429,
                    "error": "Too Many Requests",
                    "message": "API rate limit exceeded for your current plan (%s).",
                    "retryAfter": %d,
                    "documentation": "https://api.example.com/docs/rate-limiting"
                }
                """.formatted(tier.name(), waitSeconds);

            httpResponse.getWriter().write(errorBody);
        }
    }

    /**
     * Determines a unique key to identify the client.
     * Priority: API Key > Authenticated User > IP Address
     */
    private String resolveClientKey(HttpServletRequest request) {
        // 1. API Key-based identification
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null && !apiKey.isBlank()) {
            return "apikey:" + apiKey;
        }

        // 2. Authenticated user identification
        Principal principal = request.getUserPrincipal();
        if (principal != null) {
            return "user:" + principal.getName();
        }

        // 3. IP-based identification (fallback for anonymous)
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isBlank()) {
            return "ip:" + xForwardedFor.split(",")[0].trim();
        }
        return "ip:" + request.getRemoteAddr();
    }

    /**
     * Determines the rate limit tier for the client.
     * In production, this would look up the user's subscription in a database or JWT claim.
     */
    private RateLimitTier resolveClientTier(HttpServletRequest request) {
        // Check JWT claim (simplified; in production, parse the JWT)
        String planHeader = request.getHeader("X-User-Plan");
        if (planHeader != null) {
            try {
                return RateLimitTier.valueOf(planHeader.toUpperCase());
            } catch (IllegalArgumentException e) {
                // Unknown plan, fall through to default
            }
        }
        return RateLimitTier.FREE; // Default tier
    }
}
```

### 4.5 Rate Limiting by API Key with AOP (Alternative Approach)

```java
package com.example.ratelimit.annotation;

import java.lang.annotation.*;

/**
 * Annotation to apply rate limiting to individual controller methods.
 * Usage: @RateLimit(capacity = 100, refillTokens = 100, refillSeconds = 60)
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimit {
    long capacity() default 100;
    long refillTokens() default 100;
    long refillSeconds() default 60;
    String keyPrefix() default "";
}
```

```java
package com.example.ratelimit.aspect;

import com.example.ratelimit.annotation.RateLimit;
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.BucketConfiguration;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import jakarta.servlet.http.HttpServletRequest;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.server.ResponseStatusException;

import java.time.Duration;
import java.util.function.Supplier;

@Aspect
@Component
public class RateLimitAspect {

    private final ProxyManager<String> proxyManager;

    public RateLimitAspect(ProxyManager<String> proxyManager) {
        this.proxyManager = proxyManager;
    }

    @Around("@annotation(rateLimit)")
    public Object enforceRateLimit(ProceedingJoinPoint joinPoint,
                                    RateLimit rateLimit) throws Throwable {

        HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.currentRequestAttributes()).getRequest();

        String clientIp = getClientIp(request);
        String bucketKey = rateLimit.keyPrefix().isBlank()
            ? clientIp
            : rateLimit.keyPrefix() + ":" + clientIp;

        final long capacity = rateLimit.capacity();
        final long refillTokens = rateLimit.refillTokens();
        final long refillSeconds = rateLimit.refillSeconds();

        Supplier<BucketConfiguration> configSupplier = () ->
            BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(capacity)
                    .refillGreedy(refillTokens, Duration.ofSeconds(refillSeconds))
                    .build())
                .build();

        Bucket bucket = proxyManager.builder().build(bucketKey, configSupplier);

        if (!bucket.tryConsume(1)) {
            throw new ResponseStatusException(
                HttpStatus.TOO_MANY_REQUESTS,
                "Rate limit exceeded. Please slow down your requests."
            );
        }

        return joinPoint.proceed();
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        return (xff != null) ? xff.split(",")[0].trim() : request.getRemoteAddr();
    }
}
```

```java
// Usage in a controller:
@RestController
@RequestMapping("/api/v1")
public class SearchController {

    @GetMapping("/search")
    @RateLimit(capacity = 30, refillTokens = 30, refillSeconds = 60, keyPrefix = "search")
    public ResponseEntity<SearchResult> search(@RequestParam String q) {
        // Endpoint-specific rate limit: 30 searches per minute
        return ResponseEntity.ok(searchService.search(q));
    }

    @PostMapping("/export")
    @RateLimit(capacity = 5, refillTokens = 5, refillSeconds = 3600, keyPrefix = "export")
    public ResponseEntity<byte[]> export(@RequestBody ExportRequest request) {
        // Heavy operation: only 5 exports per hour
        return ResponseEntity.ok(exportService.export(request));
    }
}
```

---

## 5. Spring Cloud Gateway Rate Limiting

Spring Cloud Gateway is the recommended API Gateway for Spring microservices. It has built-in Redis-based rate limiting using the **RequestRateLimiter** filter.

### 5.1 Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
</dependencies>
```

### 5.2 YAML Configuration

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                # Number of requests allowed per second (steady state)
                redis-rate-limiter.replenishRate: 10
                # Max burst capacity (token bucket max size)
                redis-rate-limiter.burstCapacity: 20
                # Tokens consumed per request (default 1)
                redis-rate-limiter.requestedTokens: 1
                # Bean name of the KeyResolver
                key-resolver: "#{@ipKeyResolver}"

        - id: heavy-endpoint
          uri: lb://report-service
          predicates:
            - Path=/api/reports/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 2
                redis-rate-limiter.burstCapacity: 5
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@userKeyResolver}"

  redis:
    host: localhost
    port: 6379
```

### 5.3 KeyResolver Implementations

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Configuration
public class RateLimiterConfig {

    /**
     * Rate limit by IP address (for anonymous/public APIs)
     */
    @Bean
    @Primary
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            String ip = exchange.getRequest().getHeaders()
                .getFirst("X-Forwarded-For");
            if (ip != null && !ip.isBlank()) {
                ip = ip.split(",")[0].trim();
            } else {
                ip = exchange.getRequest().getRemoteAddress() != null
                    ? exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                    : "unknown";
            }
            return Mono.just(ip);
        };
    }

    /**
     * Rate limit by authenticated user ID (from JWT sub claim)
     */
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            // Extract user from JWT token in Authorization header
            String authHeader = exchange.getRequest().getHeaders()
                .getFirst("Authorization");
            if (authHeader != null && authHeader.startsWith("Bearer ")) {
                String token = authHeader.substring(7);
                // In production, decode JWT and extract subject
                String userId = extractUserIdFromJwt(token);
                return Mono.just("user:" + userId);
            }
            // Fallback to IP if not authenticated
            return ipKeyResolver().resolve(exchange).map(ip -> "anon:" + ip);
        };
    }

    /**
     * Rate limit by API Key
     */
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> {
            String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
            if (apiKey != null && !apiKey.isBlank()) {
                return Mono.just("apikey:" + apiKey);
            }
            // Deny if no API key present (or return IP-based)
            return Mono.error(new IllegalStateException("Missing X-API-Key header"));
        };
    }

    private String extractUserIdFromJwt(String token) {
        // Simplified JWT parsing — use io.jsonwebtoken or spring-security-oauth2 in prod
        // This is just to illustrate the concept
        try {
            String[] parts = token.split("\\.");
            String payload = new String(java.util.Base64.getUrlDecoder().decode(parts[1]));
            // Parse JSON and extract "sub" field
            // In production use: Jwts.parserBuilder()...
            return "user_extracted_from_jwt";
        } catch (Exception e) {
            return "unknown";
        }
    }
}
```

### 5.4 Custom RedisRateLimiter

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayRoutesConfig {

    /**
     * Programmatic route configuration with rate limiting.
     * replenishRate: tokens added per second
     * burstCapacity: max tokens (burst allowance)
     * requestedTokens: tokens consumed per request
     */
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder,
                                            KeyResolver userKeyResolver) {
        return builder.routes()
            .route("free-tier-route", r -> r
                .path("/api/free/**")
                .filters(f -> f
                    .requestRateLimiter(config -> {
                        config.setRateLimiter(new RedisRateLimiter(
                            10,   // replenishRate: 10 req/sec
                            20,   // burstCapacity: max 20 in a burst
                            1     // requestedTokens: 1 per request
                        ));
                        config.setKeyResolver(userKeyResolver);
                        config.setDenyEmptyKey(true); // 403 if key is empty
                        config.setEmptyKeyStatus("403");
                    })
                )
                .uri("lb://free-service")
            )
            .route("pro-tier-route", r -> r
                .path("/api/pro/**")
                .filters(f -> f
                    .requestRateLimiter(config -> {
                        config.setRateLimiter(new RedisRateLimiter(
                            100,  // replenishRate: 100 req/sec
                            200,  // burstCapacity
                            1     // requestedTokens
                        ));
                        config.setKeyResolver(userKeyResolver);
                    })
                )
                .uri("lb://pro-service")
            )
            .build();
    }
}
```

> [!NOTE]
> Spring Cloud Gateway stores rate limit state in Redis with keys like `request_rate_limiter.{key}.tokens` and `request_rate_limiter.{key}.timestamp`. The underlying Lua script ensures atomic token bucket operations.

---

## 6. Implementing Rate Limit Tiers

### 6.1 Storing User Plan in JWT Claims

```java
package com.example.ratelimit.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.Map;

@Service
public class JwtService {

    private static final String SECRET_KEY = "your-256-bit-secret-key-here";
    private static final long EXPIRATION_MS = 86_400_000L; // 24 hours

    /**
     * Generate JWT with rate limit tier embedded as a claim.
     */
    public String generateToken(String userId, String email, String plan) {
        return Jwts.builder()
            .setSubject(userId)
            .addClaims(Map.of(
                "email", email,
                "plan", plan,           // "FREE", "STARTER", "PRO", "ENTERPRISE"
                "rateLimit", getRateLimitForPlan(plan) // e.g. 100, 1000, 10000
            ))
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_MS))
            .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
            .compact();
    }

    /**
     * Extract rate limit tier from JWT without database lookup.
     */
    public String extractPlan(String token) {
        Claims claims = Jwts.parser()
            .setSigningKey(SECRET_KEY)
            .parseClaimsJws(token)
            .getBody();
        return claims.get("plan", String.class);
    }

    public String extractUserId(String token) {
        return Jwts.parser()
            .setSigningKey(SECRET_KEY)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    private int getRateLimitForPlan(String plan) {
        return switch (plan.toUpperCase()) {
            case "FREE" -> 100;
            case "STARTER" -> 1_000;
            case "PRO" -> 10_000;
            case "ENTERPRISE" -> Integer.MAX_VALUE;
            default -> 100;
        };
    }
}
```

### 6.2 Dynamic Rate Limits from Database

For cases where you need per-user custom limits (enterprise contracts, special accounts):

```java
package com.example.ratelimit.model;

import jakarta.persistence.*;

@Entity
@Table(name = "api_subscriptions")
public class ApiSubscription {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String userId;

    @Column(nullable = false)
    private String plan; // FREE, STARTER, PRO, ENTERPRISE

    @Column(nullable = false)
    private Integer requestsPerHour; // Custom override

    @Column(nullable = false)
    private Integer burstCapacity;

    @Column(nullable = false)
    private Boolean active = true;

    // Getters, setters, constructors...
}
```

```java
package com.example.ratelimit.service;

import com.example.ratelimit.model.ApiSubscription;
import com.example.ratelimit.repository.ApiSubscriptionRepository;
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.BucketConfiguration;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

@Service
public class DynamicRateLimitService {

    private final ProxyManager<String> proxyManager;
    private final ApiSubscriptionRepository subscriptionRepository;

    public DynamicRateLimitService(ProxyManager<String> proxyManager,
                                    ApiSubscriptionRepository subscriptionRepository) {
        this.proxyManager = proxyManager;
        this.subscriptionRepository = subscriptionRepository;
    }

    public io.github.bucket4j.Bucket resolveBucketForUser(String userId) {
        ApiSubscription subscription = getSubscription(userId);

        Supplier<BucketConfiguration> configSupplier = () ->
            BucketConfiguration.builder()
                .addLimit(Bandwidth.builder()
                    .capacity(subscription.getRequestsPerHour())
                    .refillGreedy(subscription.getRequestsPerHour(), Duration.ofHours(1))
                    .initialTokens(subscription.getBurstCapacity())
                    .build())
                .build();

        return proxyManager.builder().build("user:" + userId, configSupplier);
    }

    /**
     * Cache subscription data to avoid DB hit on every request.
     * Use Spring Cache with a short TTL (e.g., 5 minutes).
     */
    @Cacheable(value = "subscriptions", key = "#userId")
    public ApiSubscription getSubscription(String userId) {
        return subscriptionRepository.findByUserId(userId)
            .orElseGet(() -> {
                // Default to FREE tier if no subscription found
                ApiSubscription free = new ApiSubscription();
                free.setUserId(userId);
                free.setPlan("FREE");
                free.setRequestsPerHour(100);
                free.setBurstCapacity(20);
                return free;
            });
    }
}
```

> [!TIP]
> Cache subscription data with `@Cacheable` (Redis or Caffeine) to avoid a database lookup on every single API request. A 5-minute TTL is usually acceptable — plan upgrades take effect within 5 minutes, which is a reasonable user expectation.

---

## 7. Rate Limit Headers & Client Communication

Following standard conventions for rate limit headers ensures clients can implement **adaptive retry logic**.

### 7.1 Standard Headers

| Header | Description | Example |
|---|---|---|
| `X-RateLimit-Limit` | Max requests allowed in the window | `100` |
| `X-RateLimit-Remaining` | Requests remaining in current window | `42` |
| `X-RateLimit-Reset` | Unix timestamp when the window resets | `1714521600` |
| `Retry-After` | Seconds to wait before retrying (on 429) | `47` |
| `X-RateLimit-Policy` | Human-readable policy description | `100;w=3600` |

### 7.2 Response Model for 429

```java
package com.example.ratelimit.model;

import com.fasterxml.jackson.annotation.JsonInclude;

import java.time.Instant;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record RateLimitErrorResponse(
    int status,
    String error,
    String message,
    long retryAfter,
    String resetAt,        // ISO 8601 timestamp for human readability
    String documentation
) {
    public static RateLimitErrorResponse of(long nanosToWait, String docUrl) {
        long secondsToWait = nanosToWait / 1_000_000_000;
        Instant resetInstant = Instant.now().plusSeconds(secondsToWait);
        return new RateLimitErrorResponse(
            429,
            "Too Many Requests",
            "API rate limit exceeded. Please reduce your request rate.",
            secondsToWait,
            resetInstant.toString(),
            docUrl
        );
    }
}
```

### 7.3 Complete Exception Handler

```java
package com.example.ratelimit.exception;

import com.example.ratelimit.model.RateLimitErrorResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.server.ResponseStatusException;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<RateLimitErrorResponse> handleRateLimit(
            RateLimitExceededException ex) {

        RateLimitErrorResponse body = RateLimitErrorResponse.of(
            ex.getNanosToWait(),
            "https://api.example.com/docs/rate-limits"
        );

        long retryAfterSeconds = ex.getNanosToWait() / 1_000_000_000;

        return ResponseEntity
            .status(HttpStatus.TOO_MANY_REQUESTS)
            .header("X-RateLimit-Limit", String.valueOf(ex.getLimit()))
            .header("X-RateLimit-Remaining", "0")
            .header("X-RateLimit-Reset",
                String.valueOf(System.currentTimeMillis() / 1000 + retryAfterSeconds))
            .header("Retry-After", String.valueOf(retryAfterSeconds))
            .body(body);
    }
}
```

```java
package com.example.ratelimit.exception;

public class RateLimitExceededException extends RuntimeException {

    private final long nanosToWait;
    private final long limit;

    public RateLimitExceededException(long nanosToWait, long limit) {
        super("Rate limit exceeded");
        this.nanosToWait = nanosToWait;
        this.limit = limit;
    }

    public long getNanosToWait() { return nanosToWait; }
    public long getLimit() { return limit; }
}
```

### 7.4 Client-Side Handling (Java Example)

```java
// Client-side adaptive retry logic
public class ApiClient {

    private final HttpClient httpClient = HttpClient.newHttpClient();
    private static final int MAX_RETRIES = 3;

    public String callApi(String endpoint) throws Exception {
        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(endpoint))
                .header("Authorization", "Bearer " + getToken())
                .GET()
                .build();

            HttpResponse<String> response = httpClient.send(
                request, HttpResponse.BodyHandlers.ofString()
            );

            if (response.statusCode() == 200) {
                return response.body();
            }

            if (response.statusCode() == 429) {
                // Respect the Retry-After header
                String retryAfter = response.headers()
                    .firstValue("Retry-After")
                    .orElse("60");
                long waitSeconds = Long.parseLong(retryAfter);

                System.out.println("Rate limited. Waiting " + waitSeconds + "s...");
                Thread.sleep(waitSeconds * 1000 + (attempt * 1000L)); // Jitter

                if (attempt == MAX_RETRIES - 1) {
                    throw new RuntimeException("Rate limit exceeded after retries");
                }
                continue;
            }

            throw new RuntimeException("API error: " + response.statusCode());
        }
        throw new RuntimeException("Max retries exceeded");
    }

    private String getToken() { return "your-jwt-token"; }
}
```

---

## 8. Distributed Rate Limiting Architecture

### 8.1 Why Local Rate Limiting Fails at Scale

```
WITHOUT DISTRIBUTED RATE LIMITING:

Client (limit: 100 req/min)
        │
        ├──→ Instance 1: Counter = 34/100 ← different counters!
        ├──→ Instance 2: Counter = 33/100
        └──→ Instance 3: Counter = 33/100
        
Total actual requests: 100 (across instances, limit "per instance" = 300 effective)
```

With 3 instances and per-instance rate limiting, a client effectively gets 3x the intended limit. For rate limiting to be meaningful, it must be **shared state** across all instances.

### 8.2 Redis as Shared State

```
WITH DISTRIBUTED RATE LIMITING (Redis):

Client (limit: 100 req/min)
        │
        ├──→ Instance 1 ─→ Redis: INCR user:alice:counter = 45
        ├──→ Instance 2 ─→ Redis: INCR user:alice:counter = 46
        └──→ Instance 3 ─→ Redis: INCR user:alice:counter = 47
        
Total shared counter in Redis = 47/100 (accurate!)
```

### 8.3 Redis Lua Script for Atomic Sliding Window

Redis Lua scripts execute **atomically** — no two scripts run simultaneously, eliminating race conditions without application-level locking.

```lua
-- sliding_window_rate_limit.lua
-- KEYS[1] = rate limit key (e.g., "ratelimit:user:alice")
-- ARGV[1] = current timestamp in milliseconds
-- ARGV[2] = window size in milliseconds
-- ARGV[3] = max requests in window
-- Returns: {allowed (0/1), current_count, ttl_ms}

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local window_start = now - window

-- Remove expired entries (older than window)
redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

-- Count current entries in window
local count = redis.call('ZCARD', key)

if count < limit then
    -- Add current request with timestamp as score
    redis.call('ZADD', key, now, now .. '-' .. math.random(1000000))
    -- Set TTL to expire the key after the window
    redis.call('PEXPIRE', key, window)
    return {1, count + 1, window}
else
    -- Get time until oldest entry expires
    local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
    local reset_in = window - (now - tonumber(oldest[2]))
    return {0, count, reset_in}
end
```

```java
package com.example.ratelimit.redis;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;

@Service
public class RedisRateLimitService {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<List> slidingWindowScript;

    public RedisRateLimitService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;

        // Load the Lua script
        this.slidingWindowScript = new DefaultRedisScript<>();
        this.slidingWindowScript.setScriptText(SLIDING_WINDOW_LUA);
        this.slidingWindowScript.setResultType(List.class);
    }

    public RateLimitResult checkRateLimit(String key, int maxRequests,
                                           Duration window) {
        long now = System.currentTimeMillis();
        long windowMs = window.toMillis();

        @SuppressWarnings("unchecked")
        List<Long> result = (List<Long>) redisTemplate.execute(
            slidingWindowScript,
            List.of("ratelimit:" + key),
            String.valueOf(now),
            String.valueOf(windowMs),
            String.valueOf(maxRequests)
        );

        boolean allowed = result.get(0) == 1L;
        long currentCount = result.get(1);
        long resetInMs = result.get(2);

        return new RateLimitResult(allowed, maxRequests, currentCount, resetInMs);
    }

    public record RateLimitResult(
        boolean allowed,
        long limit,
        long currentCount,
        long resetInMs
    ) {
        public long remaining() {
            return Math.max(0, limit - currentCount);
        }
        public long retryAfterSeconds() {
            return resetInMs / 1000;
        }
    }

    private static final String SLIDING_WINDOW_LUA = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])
        local window_start = now - window
        redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
        local count = redis.call('ZCARD', key)
        if count < limit then
            redis.call('ZADD', key, now, now .. math.random(1000000))
            redis.call('PEXPIRE', key, window)
            return {1, count + 1, window}
        else
            local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
            local reset_in = window - (now - tonumber(oldest[2]))
            return {0, count, reset_in}
        end
        """;
}
```

### 8.4 Rate Limiting at API Gateway vs Application Level

| Concern | API Gateway | Application Level |
|---|---|---|
| **DDoS Protection** | ✅ Best — stops traffic before hitting apps | ❌ Traffic already hits app |
| **Business Logic** | ❌ Limited (no access to DB/business rules) | ✅ Full access to user data |
| **Per-Endpoint Limits** | ⚠️ Route-based only | ✅ Method/annotation level |
| **Dynamic Limits** | ❌ Usually static config | ✅ From DB, JWT, user plan |
| **Performance** | ✅ No app overhead | ⚠️ App processes filter chain |
| **Recommendation** | Global/IP limits | User/plan-specific limits |

> [!IMPORTANT]
> **Best Practice**: Layer your rate limiting. Use API Gateway for **coarse-grained global protection** (e.g., max 1000 req/min per IP) and application-level Bucket4j for **fine-grained business rules** (e.g., Free tier = 100/hour, Pro = 10,000/hour).

---

## 9. Circuit Breaker with Resilience4j

The circuit breaker is a **related but distinct** pattern from rate limiting. Where rate limiting controls *how many* requests come in, the circuit breaker controls *whether to even attempt* a downstream call.

### 9.1 Dependencies

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 9.2 Circuit Breaker States

```
┌──────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STATES                    │
│                                                              │
│  ┌─────────┐   failure rate     ┌──────────┐                │
│  │  CLOSED  │ ─────exceeds───→  │   OPEN   │                │
│  │ (normal) │    threshold       │ (rejects │                │
│  └─────────┘                   │ all calls)│                │
│       ↑                         └──────────┘                │
│       │                               │                     │
│  some succeed                    wait timeout               │
│       │                               │                     │
│  ┌───────────┐                        ▼                     │
│  │ HALF-OPEN │ ←─────────────── allow limited              │
│  │ (probe)   │                   test calls                 │
│  └───────────┘                                              │
└──────────────────────────────────────────────────────────────┘
```

| State | Behavior | Transition |
|---|---|---|
| **CLOSED** | All requests pass through | → OPEN when failure rate > threshold |
| **OPEN** | All requests fail immediately (fast fail) | → HALF_OPEN after wait duration |
| **HALF_OPEN** | Limited requests allowed through as a probe | → CLOSED if probe succeeds, → OPEN if fails |

### 9.3 Configuration

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        # Sliding window size (number of calls or seconds)
        slidingWindowSize: 10
        slidingWindowType: COUNT_BASED  # or TIME_BASED
        # Open circuit when >50% of calls fail
        failureRateThreshold: 50
        # Minimum calls before failure rate is calculated
        minimumNumberOfCalls: 5
        # Wait before transitioning OPEN → HALF_OPEN
        waitDurationInOpenState: 10s
        # Number of calls allowed in HALF_OPEN state
        permittedNumberOfCallsInHalfOpenState: 3
        # Slow call threshold
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 100  # 100% slow calls = open circuit
        # Exceptions that trigger the circuit breaker
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        # Exceptions that are ignored
        ignoreExceptions:
          - java.lang.IllegalArgumentException

  retry:
    instances:
      payment-service:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException

  bulkhead:
    instances:
      payment-service:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms

  ratelimiter:
    instances:
      payment-service:
        limitForPeriod: 50
        limitRefreshPeriod: 1s
        timeoutDuration: 100ms
```

### 9.4 Using Resilience4j Annotations

```java
package com.example.resilience.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class PaymentServiceClient {

    private static final Logger log = LoggerFactory.getLogger(PaymentServiceClient.class);
    private final RestClient restClient;

    public PaymentServiceClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("http://payment-service")
            .build();
    }

    /**
     * Circuit Breaker + Retry stacked together.
     * Order matters: @CircuitBreaker wraps @Retry.
     * If circuit is OPEN, @Retry won't even attempt the call.
     */
    @CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
    @Retry(name = "payment-service")
    @Bulkhead(name = "payment-service", type = Bulkhead.Type.SEMAPHORE)
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment for order: {}", request.orderId());

        return restClient.post()
            .uri("/api/payments")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }

    /**
     * Fallback method when circuit is OPEN or all retries exhausted.
     * Method signature must match the original + a Throwable parameter.
     */
    public PaymentResult paymentFallback(PaymentRequest request, Throwable ex) {
        log.warn("Payment service unavailable, using fallback. Error: {}", ex.getMessage());

        // Options:
        // 1. Return a cached/default response
        // 2. Queue for async processing
        // 3. Return a meaningful degraded response
        return PaymentResult.pending(request.orderId(),
            "Payment queued for processing. You'll receive confirmation shortly.");
    }

    /**
     * Rate limiting with Resilience4j (application-level).
     * Different from Bucket4j — this is for outbound call rate limiting.
     */
    @RateLimiter(name = "payment-service", fallbackMethod = "rateLimitFallback")
    public PaymentStatus getPaymentStatus(String paymentId) {
        return restClient.get()
            .uri("/api/payments/{id}", paymentId)
            .retrieve()
            .body(PaymentStatus.class);
    }

    public PaymentStatus rateLimitFallback(String paymentId, Throwable ex) {
        return PaymentStatus.unknown(paymentId);
    }
}
```

### 9.5 Circuit Breaker Event Listeners

```java
package com.example.resilience.monitoring;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.annotation.PostConstruct;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class CircuitBreakerMonitor {

    private static final Logger log = LoggerFactory.getLogger(CircuitBreakerMonitor.class);

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final MeterRegistry meterRegistry;

    public CircuitBreakerMonitor(CircuitBreakerRegistry circuitBreakerRegistry,
                                  MeterRegistry meterRegistry) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
        this.meterRegistry = meterRegistry;
    }

    @PostConstruct
    public void registerEventListeners() {
        circuitBreakerRegistry.getAllCircuitBreakers()
            .forEach(this::registerListeners);

        // Also register for dynamically created circuit breakers
        circuitBreakerRegistry.getEventPublisher()
            .onEntryAdded(entry -> registerListeners(entry.getAddedEntry()));
    }

    private void registerListeners(CircuitBreaker circuitBreaker) {
        String name = circuitBreaker.getName();

        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> {
                log.warn("Circuit Breaker [{}] state changed: {} → {}",
                    name,
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()
                );
                // Alert your on-call team here (PagerDuty, Slack, etc.)
                sendAlert(name, event.getStateTransition().getToState().name());
            })
            .onFailureRateExceeded(event ->
                log.error("Circuit Breaker [{}] failure rate: {}%",
                    name, event.getFailureRate())
            )
            .onCallNotPermitted(event ->
                log.debug("Circuit Breaker [{}] rejected call (OPEN state)", name)
            );
    }

    private void sendAlert(String circuitBreakerName, String newState) {
        // Integration with alerting system
        meterRegistry.counter("circuit_breaker.state_changes",
            "name", circuitBreakerName,
            "state", newState
        ).increment();
    }
}
```

### 9.6 Rate Limiting vs Circuit Breaker — Key Differences

| Aspect | Rate Limiting | Circuit Breaker |
|---|---|---|
| **Purpose** | Control incoming request volume | Protect against failing downstream services |
| **Direction** | Inbound requests | Outbound calls to dependencies |
| **Trigger** | Number of requests per time window | Error rate / slow response rate |
| **Behavior** | Reject excess requests with 429 | Fast-fail all calls with fallback |
| **State** | Counter/bucket | CLOSED / OPEN / HALF_OPEN |
| **Recovery** | Automatic (window resets) | Automatic (HALF_OPEN probe) |
| **Use Case** | API protection, fair usage | Resilience, cascading failure prevention |

### 9.7 Bulkhead Pattern

The **Bulkhead** pattern (named after ship compartments) isolates failures to prevent cascading.

```java
// Semaphore Bulkhead: limits concurrent calls
@Bulkhead(name = "payment-service", type = Bulkhead.Type.SEMAPHORE)
public PaymentResult processPayment(PaymentRequest request) { ... }

// Thread Pool Bulkhead: isolates in a dedicated thread pool
@Bulkhead(name = "report-service", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<Report> generateReport(ReportRequest request) { ... }
```

```yaml
resilience4j:
  bulkhead:
    instances:
      payment-service:
        maxConcurrentCalls: 10      # Max parallel calls
        maxWaitDuration: 100ms      # How long to wait for a permit

  thread-pool-bulkhead:
    instances:
      report-service:
        maxThreadPoolSize: 4
        coreThreadPoolSize: 2
        queueCapacity: 2
        keepAliveDuration: 20ms
```

---

## 10. Load Balancing Basics

### 10.1 Client-Side vs Server-Side Load Balancing

| Aspect | Client-Side | Server-Side |
|---|---|---|
| **Where logic runs** | In the client application | In a dedicated proxy/gateway |
| **Discovery** | Client queries service registry | Proxy handles discovery |
| **Examples** | Spring Cloud LoadBalancer, Ribbon | Nginx, HAProxy, AWS ALB, API Gateway |
| **Pros** | No single point of failure, faster | Simpler clients, centralized control |
| **Cons** | Heavier clients, multiple registries to update | SPOF if proxy fails, extra network hop |

```
Client-Side:                          Server-Side:

 Client App                            Client App
    │                                      │
    ▼                                      ▼
 Service Registry                       Load Balancer
  (Eureka/Consul)       ┌─── Nginx/Gateway/ALB ───┐
    │                   │         │         │      │
    ▼                   ▼         ▼         ▼      ▼
 Choose instance     Instance1 Inst2 Inst3 Inst4
 directly
    │
    ▼
 Instance (direct)
```

### 10.2 Spring Cloud LoadBalancer (Ribbon Replacement)

```xml
<!-- pom.xml - Spring Cloud LoadBalancer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
<!-- Service Discovery with Eureka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```java
package com.example.loadbalancer.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;
import org.springframework.web.client.RestClient;

@Configuration
// Per-service custom load balancer config
@LoadBalancerClient(name = "payment-service", configuration = RandomLBConfig.class)
public class LoadBalancerConfig {

    /**
     * @LoadBalanced makes RestTemplate/RestClient use Spring Cloud LoadBalancer
     * to resolve service names (e.g., "http://payment-service/api/pay")
     */
    @Bean
    @LoadBalanced
    public RestClient.Builder loadBalancedRestClientBuilder() {
        return RestClient.builder();
    }
}

// Custom load balancer strategy for a specific service
class RandomLBConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME
        );
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}
```

```java
// Using the load-balanced client — service name is resolved automatically
@Service
public class OrderService {

    private final RestClient restClient;

    public OrderService(@LoadBalanced RestClient.Builder builder) {
        // "payment-service" is resolved via Eureka/Consul
        this.restClient = builder.baseUrl("http://payment-service").build();
    }

    public PaymentResult pay(PaymentRequest request) {
        // Spring Cloud LoadBalancer picks an instance of "payment-service"
        return restClient.post()
            .uri("/api/payments")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }
}
```

### 10.3 Ribbon (Deprecated) vs Spring Cloud LoadBalancer

| Feature | Netflix Ribbon (Deprecated) | Spring Cloud LoadBalancer |
|---|---|---|
| **Status** | ❌ Deprecated (Spring Cloud 2020.x+) | ✅ Current standard |
| **Reactive** | ❌ Blocking only | ✅ Reactive + blocking |
| **Algorithm** | Round Robin, Random, WeightedResponse | Round Robin, Random |
| **Configuration** | Property-based, complex | Simple, extensible |
| **Maintenance** | No longer maintained | Actively maintained |

> [!WARNING]
> **Never use Ribbon in new projects.** It's been deprecated since Spring Cloud 2020.0.x (Ilford) and removed in Spring Cloud 2022.x. Use **Spring Cloud LoadBalancer** instead.

---

## 11. Throttling vs Rate Limiting

These terms are often used interchangeably, but they have distinct meanings:

| Aspect | Rate Limiting | Throttling |
|---|---|---|
| **Definition** | Hard limit — requests over limit are **rejected** | Soft limit — excess requests are **slowed down/queued** |
| **Response** | 429 Too Many Requests (immediate rejection) | Delayed response (added latency) |
| **Client Experience** | Error response, must retry | Slower response, eventually succeeds |
| **Use Case** | API monetization, abuse prevention | Traffic shaping, backend protection |
| **Mechanism** | Counter comparison → reject | Queue + delay → process eventually |

### 11.1 When to Throttle Instead of Rate Limit

**Throttle when:**
- You can tolerate higher latency but not errors
- Background jobs / batch processing (a slow response is OK)
- You want to smooth out traffic spikes without losing requests
- The operation is idempotent and can safely be delayed

**Rate limit when:**
- Client-facing APIs with SLA guarantees
- Security-sensitive endpoints (login, OTP, password reset)
- API monetization (paid tiers)
- Fair usage enforcement

### 11.2 Simple Throttling Implementation

```java
package com.example.throttle;

import org.springframework.stereotype.Service;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * Throttle concurrent access to a resource.
 * Unlike rate limiting (rejects), throttling WAITS.
 */
@Service
public class ThrottlingService {

    // Max 10 concurrent operations
    private final Semaphore semaphore = new Semaphore(10, true); // fair=true

    public <T> T throttled(java.util.concurrent.Callable<T> operation)
            throws Exception {

        boolean acquired = semaphore.tryAcquire(
            5, TimeUnit.SECONDS // Wait up to 5 seconds
        );

        if (!acquired) {
            throw new ThrottlingException("Too many concurrent requests. Please retry.");
        }

        try {
            return operation.call();
        } finally {
            semaphore.release(); // Always release!
        }
    }
}
```

---

## 12. ⚠️ Interview Traps & Gotchas

### Trap 1: Rate Limiting Per Instance vs Distributed

**Wrong answer:** "I'd use a `ConcurrentHashMap` to track request counts."

**Correct answer:** "Per-instance rate limiting breaks with horizontal scaling. If we have 5 instances with a 100 req/min limit each, a client can make 500 req/min by spreading requests across instances. We need **distributed rate limiting** with shared state in Redis using Bucket4j or Spring Cloud Gateway's RedisRateLimiter."

---

### Trap 2: Token Bucket Burst vs Leaky Bucket

**Q: "A client should be allowed bursts of up to 20 requests immediately, but no more than 100 per minute. Which algorithm?"**

**Answer:** **Token Bucket.** The bucket starts with 20 tokens (burst capacity), allowing immediate bursts. The leaky bucket has a **constant output rate** and cannot handle bursts — all requests are processed at the same steady pace regardless of when they arrive.

---

### Trap 3: HTTP Status Code for Rate Limiting

**Wrong:** 403 Forbidden, 503 Service Unavailable

**Correct:** **429 Too Many Requests** (RFC 6585)

The response MUST include a `Retry-After` header indicating when the client can retry. A 503 implies the server is down; a 429 specifically communicates rate limiting.

---

### Trap 4: Circuit Breaker vs Rate Limiter

| | Rate Limiter | Circuit Breaker |
|---|---|---|
| **Protects** | Your service from being overwhelmed | Your service from failing dependencies |
| **Reacts to** | Request volume | Error rate / response time |
| **State** | Stateless (counter per window) | Stateful (CLOSED/OPEN/HALF_OPEN) |
| **Analogy** | Nightclub bouncer (controls entry) | Electrical circuit breaker (cuts power on overload) |

**Common wrong answer:** Saying circuit breaker controls how many requests come in. It controls whether **outbound calls** are even attempted.

---

### Trap 5: Authenticated vs Anonymous Rate Limiting

**Q: "How do you rate limit authenticated users differently from anonymous users?"**

**Answer:**
```
1. Extract identity:
   - Anonymous: Rate limit by IP address ("ip:192.168.1.1")
   - Authenticated: Rate limit by user ID or API key ("user:alice")

2. Apply different limits:
   - Anonymous: 100 req/hour (restrictive — can't verify identity)
   - Free tier: 1,000 req/hour
   - Pro tier: 10,000 req/hour

3. Use separate Redis keys:
   - "ratelimit:anon:ip:192.168.1.1"
   - "ratelimit:user:alice:free"
   - "ratelimit:user:bob:pro"
```

This is why you shouldn't use IP-only rate limiting for authenticated APIs — multiple users behind a corporate NAT share the same IP.

---

### Trap 6: Thread Safety of Rate Limit Counters

**Q: "Your rate limit counter is stored in a HashMap. What's wrong?"**

**Answer:** HashMap is not thread-safe. Under concurrent access:
1. Two threads read counter = 99
2. Both check 99 < 100 → both allowed
3. Both increment to 100
4. 101st request also reads 100, gets rejected (but 2 requests slipped through)

**Solutions:**
1. `ConcurrentHashMap` with `synchronized` blocks per key
2. `AtomicLong` for the counter
3. Bucket4j (uses compare-and-set internally — lock-free)
4. Redis with Lua scripts (single-threaded execution)

---

### Trap 7: The Sliding Window Boundary Problem

**Q: "Fixed window counter allows 100 requests per minute. Is 100 requests the true maximum?"**

**Answer:** No! A client can make 100 requests in the last second of window 1, and 100 requests in the first second of window 2 — resulting in **200 requests in 2 seconds** while technically never exceeding 100 in any single window. This is why sliding window algorithms exist.

---

### Trap 8: What About the `X-RateLimit-Reset` Header Format?

Two conventions exist:
- **Unix timestamp** (seconds since epoch): Most common, used by GitHub, Twitter
- **Seconds remaining**: Used by some APIs

Both are acceptable, but you MUST document which format you use. Clients interpret them differently!

---

### Trap 9: Rate Limiting Granularity

**Q: "Should you rate limit per IP, per user, per API key, or globally?"**

**Answer:** All of the above, layered:
1. **Global** — total requests across all clients (protect infrastructure)
2. **Per IP** — stop single-source DDoS
3. **Per User/API Key** — enforce plan limits
4. **Per Endpoint** — heavy operations (export, report) get stricter limits than simple reads

---

### Trap 10: Bucket4j `tryConsume` vs `consume`

```java
// tryConsume: non-blocking, returns false immediately if no tokens
boolean allowed = bucket.tryConsume(1); // Returns immediately

// consume: BLOCKS until tokens are available (don't use in web apps!)
bucket.consume(1); // ⚠️ Blocks the request thread!

// tryConsumeAndReturnRemaining: returns a ConsumptionProbe with details
ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
probe.isConsumed()          // was request allowed?
probe.getRemainingTokens()  // tokens left
probe.getNanosToWaitForRefill() // time until next token available
```

> [!WARNING]
> Never use `bucket.consume()` (blocking) in a servlet-based Spring Boot application. It blocks the request thread while waiting for tokens, which can exhaust your thread pool under load. Always use `tryConsume()` or `tryConsumeAndReturnRemaining()`.

---

### Trap 11: Memory Leak with Per-Client Buckets

Storing one bucket per client in a `ConcurrentHashMap` will grow indefinitely as new IPs/users appear. After a DDoS attack, you could have millions of entries.

**Solution:**
```java
// Use Caffeine cache with expiration
@Bean
public Cache<String, Bucket> bucketCache() {
    return Caffeine.newBuilder()
        .maximumSize(100_000)           // Max 100k unique clients
        .expireAfterAccess(Duration.ofHours(1)) // Evict inactive clients
        .build();
}
```

---

## 13. 📝 Summary & Key Takeaways

| Topic | Key Point |
|---|---|
| **Why Rate Limit?** | DDoS protection, monetization, fair usage, backend protection |
| **Best Algorithm** | Token Bucket for APIs, Sliding Window for precision |
| **Library Choice** | Bucket4j for Spring Boot; Spring Cloud Gateway for API Gateway layer |
| **Local vs Distributed** | Always use distributed (Redis) in multi-instance deployments |
| **Status Code** | 429 Too Many Requests with `Retry-After` header |
| **Circuit Breaker** | Protects outbound calls; different from rate limiting |
| **Load Balancer** | Spring Cloud LoadBalancer replaces deprecated Ribbon |
| **Throttling** | Delays requests; rate limiting rejects them |
| **Thread Safety** | Use Bucket4j or Redis Lua scripts; never raw HashMap |
| **Tiers** | Store plan in JWT for O(1) tier lookup; cache DB lookups |

### Architecture Decision Tree

```
Need to control traffic?
│
├── Incoming API requests → RATE LIMITING
│   ├── Single instance → Bucket4j (local)
│   ├── Multi-instance → Bucket4j + Redis (distributed)
│   └── API Gateway → Spring Cloud Gateway RedisRateLimiter
│
├── Outbound calls failing? → CIRCUIT BREAKER (Resilience4j)
│
├── Too many concurrent operations? → BULKHEAD (Semaphore/Thread Pool)
│
├── Need to slow down (not reject)? → THROTTLING
│
└── Multiple instances need same logic? → LOAD BALANCER (Spring Cloud LoadBalancer)
```

---

## 14. ❓ Practice Questions

### Conceptual Questions

1. **Explain the difference between the Token Bucket and Leaky Bucket algorithms. When would you use each?**

2. **Your API has a rate limit of 100 requests per minute using a Fixed Window Counter. A client makes 100 requests at 12:59 and 100 more at 13:00. How many requests did they effectively make in a 1-minute period? How would you fix this?**

3. **You deploy your Spring Boot application to 3 instances behind a load balancer. Your in-memory rate limiter allows 100 req/min per user. What's the actual effective rate limit? How would you fix it?**

4. **What HTTP status code should be returned for a rate-limited request? What headers should accompany it?**

5. **Explain the CLOSED, OPEN, and HALF_OPEN states of a circuit breaker. What triggers each transition?**

6. **What is the difference between rate limiting and throttling? Give a real-world scenario for each.**

7. **How would you implement different rate limits for free vs premium users without hitting a database on every request?**

8. **Why is it dangerous to use `bucket.consume()` (blocking) in a Spring MVC application?**

### Design Questions

9. **Design a rate limiting system for an API with the following tiers:**
   - Anonymous: 100 req/hour
   - Free users: 1,000 req/hour  
   - Pro users: 10,000 req/hour
   - Enterprise: 100,000 req/hour
   
   Consider: distributed deployment, JWT-based tier identification, proper response headers.

10. **You're building a login endpoint. Design a rate limiting strategy that:**
    - Limits by IP: 10 login attempts per 15 minutes
    - Limits by username: 5 login attempts per 15 minutes
    - Returns 429 with Retry-After on breach
    - Does not count successful logins against the limit

11. **You have a microservice that calls 3 downstream services. Service B is flaky (30% error rate). Design a resilience strategy using circuit breakers, retries, and fallbacks.**

### Code Questions

12. **Write a Bucket4j filter that:**
    - Rates limits by API key (from header `X-API-Key`)
    - Falls back to IP-based limiting if no API key present
    - Returns proper 429 response with all rate limit headers

13. **What's wrong with this code?**
    ```java
    private Map<String, Integer> counters = new HashMap<>();
    
    public boolean isAllowed(String userId) {
        int count = counters.getOrDefault(userId, 0);
        if (count >= 100) return false;
        counters.put(userId, count + 1);
        return true;
    }
    ```

14. **How would you test rate limiting logic in a Spring Boot integration test?**

---

> [!NOTE]
> **References:**
> - [Bucket4j GitHub](https://github.com/bucket4j/bucket4j)
> - [Spring Cloud Gateway Rate Limiter](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#the-requestratelimiter-gatewayfilter-factory)
> - [Resilience4j Documentation](https://resilience4j.readme.io/docs)
> - [RFC 6585 — 429 Too Many Requests](https://tools.ietf.org/html/rfc6585)
> - [IETF RateLimit Headers Draft](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-ratelimit-headers)
