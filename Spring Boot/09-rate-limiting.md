# 09 — Rate Limiting & Traffic Management in Spring Boot

> A comprehensive, production-grade guide to API protection, rate limiting algorithms, architectural enforcement tiers, Bucket4j integration, Spring Cloud Gateway reactive filters, Resilience4j circuit breakers, bulkheads, and distributed Redis coordination in Spring Boot 3.x and Java 17/21+.

---

## Table of Contents

1. [Traffic Management Core Fundamentals](#1-traffic-management-core-fundamentals)
   - [1.1 Why Rate Limit? (DDoS, Noisy Neighbors, API Monetization)](#11-why-rate-limit-ddos-noisy-neighbors-api-monetization)
   - [1.2 RFC 6585 & Standard Rate Limit HTTP Headers](#12-rfc-6585--standard-rate-limit-http-headers)
2. [Multi-Approach: Core Rate Limiting Algorithms](#2-multi-approach-core-rate-limiting-algorithms)
   - [Approach 1: Token Bucket Algorithm (Refill Rate + Burst Allowance)](#approach-1-token-bucket-algorithm-refill-rate--burst-allowance)
   - [Approach 2: Leaky Bucket Algorithm (Constant Egress Traffic Shaping)](#approach-2-leaky-bucket-algorithm-constant-egress-traffic-shaping)
   - [Approach 3: Sliding Window Log (Exact Timestamp Inspection)](#approach-3-sliding-window-log-exact-timestamp-inspection)
   - [Approach 4: Sliding Window Counter (Memory-Efficient Rolling Approximation)](#approach-4-sliding-window-counter-memory-efficient-rolling-approximation)
   - [Approach 5: Fixed Window Counter (Simple Boundaries & The 2x Burst Vulnerability)](#approach-5-fixed-window-counter-simple-boundaries--the-2x-burst-vulnerability)
   - [Rate Limiting Algorithms Comparison Matrix](#rate-limiting-algorithms-comparison-matrix)
3. [Multi-Approach: Architectural Enforcement Layers](#3-multi-approach-architectural-enforcement-layers)
   - [Approach 1: Ingress API Gateway Rate Limiting (Spring Cloud Gateway + Reactive Redis)](#approach-1-ingress-api-gateway-rate-limiting-spring-cloud-gateway--reactive-redis)
   - [Approach 2: Servlet Filter Layer Enforcement (Bucket4j + Distributed Lettuce Proxy)](#approach-2-servlet-filter-layer-enforcement-bucket4j--distributed-lettuce-proxy)
   - [Approach 3: Method-Level Declarative AOP (`@RateLimited` Custom Aspect)](#approach-3-method-level-declarative-aop-ratelimited-custom-aspect)
   - [Approach 4: Client-Side Egress Rate Limiting (Outbound 3rd-Party Gateway Protection)](#approach-4-client-side-egress-rate-limiting-outbound-3rd-party-gateway-protection)
   - [Architectural Enforcement Comparison Matrix](#architectural-enforcement-comparison-matrix)
4. [Multi-Approach: Client Identification & Key Resolution Strategies](#4-multi-approach-client-identification--key-resolution-strategies)
   - [Approach 1: Client IP Resolution with Reverse-Proxy Validation (`X-Forwarded-For`)](#approach-1-client-ip-resolution-with-reverse-proxy-validation-x-forwarded-for)
   - [Approach 2: Authenticated User Principal & JWT Claim Resolution](#approach-2-authenticated-user-principal--jwt-claim-resolution)
   - [Approach 3: API Key & Organization Token Resolution (`X-API-Key`)](#approach-3-api-key--organization-token-resolution-x-api-key)
   - [Approach 4: Multi-Dimensional Compound Key Resolution (`tenant:tier:endpoint:client`)](#approach-4-multi-dimensional-compound-key-resolution-tenanttierendpointclient)
   - [Key Resolution Comparison Matrix](#key-resolution-comparison-matrix)
5. [Multi-Approach: Resilience Patterns with Resilience4j](#5-multi-approach-resilience-patterns-with-resilience4j)
   - [Approach 1: Inbound Rate Limiter (Throttling Excess Demand)](#approach-1-inbound-rate-limiter-throttling-excess-demand)
   - [Approach 2: Circuit Breaker (`CLOSED` -> `OPEN` -> `HALF_OPEN` Tripping)](#approach-2-circuit-breaker-closed---open---half_open-tripping)
   - [Approach 3: Bulkhead Isolation (Semaphore vs. Thread-Pool Isolation)](#approach-3-bulkhead-isolation-semaphore-vs-thread-pool-isolation)
   - [Approach 4: Soft Throttling & Adaptive Concurrency (Queue Delay vs. Hard 429)](#approach-4-soft-throttling--adaptive-concurrency-queue-delay-vs-hard-429)
   - [Resilience Patterns Comparison Matrix](#resilience-patterns-comparison-matrix)
6. [Multi-Approach: Distributed Redis Coordination Backends](#6-multi-approach-distributed-redis-coordination-backends)
   - [Approach 1: Bucket4j with Lettuce CAS Proxy Manager](#approach-1-bucket4j-with-lettuce-cas-proxy-manager)
   - [Approach 2: Atomic Redis Lua Scripting (Single-Roundtrip Rolling Sorted Set)](#approach-2-atomic-redis-lua-scripting-single-roundtrip-rolling-sorted-set)
   - [Distributed Coordination Comparison Matrix](#distributed-coordination-comparison-matrix)
7. [Production Monitoring, Micrometer Telemetry, & Alerting](#7-production-monitoring-micrometer-telemetry--alerting)
8. [Interview Traps, Anti-Patterns, & Gotchas](#8-interview-traps-anti-patterns--gotchas)
9. [Summary & Architectural Decision Trees](#9-summary--architectural-decision-trees)
10. [Practice Questions & Interview Scenarios](#10-practice-questions--interview-scenarios)

---

## 1. Traffic Management Core Fundamentals

### 1.1 Why Rate Limit? (DDoS, Noisy Neighbors, API Monetization)

In modern microservice architectures, rate limiting serves three distinct operational objectives:
1. **Security & Stability**: Protecting downstream relational databases and internal services from Denial of Service (DoS) floods, credential brute-forcing, and resource exhaustion.
2. **Fair Resource Allocation (Noisy Neighbor Mitigation)**: Preventing a single misconfigured consumer or bulk scraping bot from consuming 95% of connection pools.
3. **Monetization & API Tiers**: Enforcing business contracts where subscription tiers dictate throughput quotas (e.g., Free = 100 req/hr, Enterprise = 50,000 req/min).

### 1.2 RFC 6585 & Standard Rate Limit HTTP Headers

When a client breaches their allotted quota, the HTTP specification dictates returning **HTTP 429 (Too Many Requests)**. The response must include standardized diagnostic headers so clients can implement exponential backoff:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714521600
Retry-After: 42

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "Quota exceeded for Tier [FREE]. Reset in 42 seconds.",
  "instance": "/api/v1/orders"
}
```

---

## 2. Multi-Approach: Core Rate Limiting Algorithms

```
                          ALGORITHM ACCURACY VS OVERHEAD
 Low Memory Overhead                                             Perfect Accuracy
 Approximated Windows                                            High Memory / CPU
       │                                                                │
       ▼                                                                ▼
[Approach 5: Fixed Window] ──> [Approach 1: Token Bucket] ──> [Approach 4: Sliding Counter] ──> [Approach 3: Sliding Log]
(Counter + Reset)              (Refill + Burst)                (Weighted Windows)               (Timestamp ZSet)
```

---

### Approach 1: Token Bucket Algorithm (Refill Rate + Burst Allowance)

#### Scenario
The industry standard for REST APIs, SaaS platforms, and public gateways where clients should be permitted to burst during initial page loads, but restricted to a smooth average refill cadence over time.

#### Use-Case
Stripe, GitHub, or AWS API gateways where a user is permitted 10 requests per second steady-state, but allowed an initial burst of 50 requests.

#### Benefits Over Others / Trade-offs
- **Benefits**: Handles traffic bursts naturally; memory efficient ($O(1)$ per client storing just token count and last refill timestamp); computationally fast.
- **Trade-offs**: Does not guarantee a constant egress rate (downstream dependencies may receive sudden traffic spikes during bursts).

#### Well-Documented Code Example

```java
package com.example.ratelimit.algorithm;

import java.time.Instant;

/**
 * Approach 1: Conceptual Token Bucket Implementation
 */
public class TokenBucket {
    private final long capacity;          // Maximum burst limit
    private final double refillRatePerSecond; // Tokens added per second
    private double availableTokens;
    private long lastRefillTimestamp;

    public TokenBucket(long capacity, double refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        this.availableTokens = capacity;
        this.lastRefillTimestamp = System.currentTimeMillis();
    }

    public synchronized boolean tryConsume(int tokensRequested) {
        refill();

        if (availableTokens >= tokensRequested) {
            availableTokens -= tokensRequested;
            return true; // Request allowed
        }
        return false; // Quota exhausted -> 429
    }

    private void refill() {
        long now = System.currentTimeMillis();
        double elapsedSeconds = (now - lastRefillTimestamp) / 1000.0;
        double tokensToAdd = elapsedSeconds * refillRatePerSecond;

        if (tokensToAdd > 0) {
            availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
            lastRefillTimestamp = now;
        }
    }
}
```

---

### Approach 2: Leaky Bucket Algorithm (Constant Egress Traffic Shaping)

#### Scenario
Protecting fragile downstream legacy servers, third-party payment gateways, or physical printers that crash if requests arrive in sudden bursts.

#### Use-Case
Outbound SMS dispatching where the telecommunication carrier hard-throttles connections exceeding exactly 10 SMS per second.

#### Benefits Over Others / Trade-offs
- **Benefits**: Enforces a strictly constant, smoothed egress rate; completely eliminates traffic spikes to downstream dependencies.
- **Trade-offs**: Rejects bursts even when system resources are idle; increases latency for queued requests.

#### Well-Documented Code Example

```java
package com.example.ratelimit.algorithm;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

/**
 * Approach 2: Leaky Bucket Traffic Shaper
 */
public class LeakyBucketShaper {
    private final BlockingQueue<Runnable> requestQueue;

    public LeakyBucketShaper(int queueCapacity, long leakIntervalMs) {
        this.requestQueue = new ArrayBlockingQueue<>(queueCapacity);

        // Background worker drains the bucket at a strictly fixed cadence
        Thread leakerThread = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Runnable task = requestQueue.poll(leakIntervalMs, TimeUnit.MILLISECONDS);
                    if (task != null) {
                        task.run(); // Process at constant leak rate
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        leakerThread.setDaemon(true);
        leakerThread.start();
    }

    public boolean enqueueRequest(Runnable task) {
        // Returns false immediately if queue is full (bucket overflow -> 429)
        return requestQueue.offer(task);
    }
}
```

---

### Approach 3: Sliding Window Log (Exact Timestamp Inspection)

#### Scenario
High-security endpoints (credit card checkouts, password reset OTP submissions) where approximate rate limits are unacceptable and the sliding window must be mathematically exact to the millisecond.

#### Use-Case
Preventing brute-force attacks on `/api/v1/auth/login` (strictly max 5 attempts per rolling 15-minute window).

#### Benefits Over Others / Trade-offs
- **Benefits**: 100% mathematically precise; completely eliminates the window boundary 2x burst vulnerability.
- **Trade-offs**: High memory footprint ($O(N)$ where $N$ is the number of request timestamps stored per client); expensive memory cleanup.

---

### Approach 4: Sliding Window Counter (Memory-Efficient Rolling Approximation)

#### Scenario
Large-scale production platforms serving millions of active users who need sliding window accuracy without the memory bloat of timestamp logs.

#### Use-Case
Cloudflare-style or GitHub-style public API rate limits (e.g. 5,000 requests per rolling hour).

#### Benefits Over Others / Trade-offs
- **Benefits**: Fixed memory footprint ($O(1)$ storing only two integer counters: previous window and current window); smooths out boundary bursts.
- **Trade-offs**: Approximation formula assumes uniform request distribution within the prior window (negligible ~0.05% margin of error).

#### Well-Documented Code Example

```java
package com.example.ratelimit.algorithm;

/**
 * Approach 4: Sliding Window Counter Approximation Formula
 * Effective Count = (Previous Window Count * (1 - Percentage of Current Window Elapsed)) + Current Window Count
 */
public class SlidingWindowCounter {
    private final long windowSizeMs;
    private final int limit;
    private long currentWindowStartMs;
    private int previousCount;
    private int currentCount;

    public SlidingWindowCounter(long windowSizeMs, int limit) {
        this.windowSizeMs = windowSizeMs;
        this.limit = limit;
        this.currentWindowStartMs = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        long elapsedInCurrentWindow = now - currentWindowStartMs;

        // Transition window if time has passed
        if (elapsedInCurrentWindow >= windowSizeMs) {
            previousCount = currentCount;
            currentCount = 0;
            currentWindowStartMs = now;
            elapsedInCurrentWindow = 0;
        }

        double percentRemainingInPrevWindow = 1.0 - ((double) elapsedInCurrentWindow / windowSizeMs);
        double estimatedCount = (previousCount * percentRemainingInPrevWindow) + currentCount;

        if (estimatedCount < limit) {
            currentCount++;
            return true;
        }
        return false;
    }
}
```

---

### Approach 5: Fixed Window Counter (Simple Boundaries & The 2x Burst Vulnerability)

#### Scenario
Low-concurrency internal microservices or educational prototypes where absolute simplicity is prioritized over burst precision.

#### The 2x Boundary Burst Vulnerability Explained:
If an API limits consumers to 100 requests per minute:
- The consumer sends 100 requests at 11:59:50 (last 10 seconds of Window 1).
- The window rolls over at 12:00:00.
- The consumer immediately sends 100 requests at 12:00:10 (first 10 seconds of Window 2).
- **Result**: The consumer sent **200 requests within a 20-second span**, completely defeating the intended protection rate.

---

### Rate Limiting Algorithms Comparison Matrix

| Algorithm | Memory Usage | CPU Overhead | Burst Handling | Boundary Safe? | Best Use Case |
|---|---|---|---|---|---|
| **Approach 1: Token Bucket** | **$O(1)$** | **Lowest** | **Yes (Up to Capacity)** | Yes | **Public REST APIs (Industry Standard)** |
| **Approach 2: Leaky Bucket** | $O(N)$ Queue | Low | **No (Constant Output)**| Yes | Outbound SMS / Payment Gateway Shapers |
| **Approach 3: Sliding Log** | $O(N)$ Timestamps | High | Yes | **100% Exact** | Login / OTP / Security Endpoints |
| **Approach 4: Sliding Counter** | **$O(1)$** | Low | Smooths Bursts | **~99.9% Safe** | High-Scale Distributed Platforms |
| **Approach 5: Fixed Window** | $O(1)$ | Lowest | **2x Vulnerability** | **No** | Simple Internal Jobs Only |

---

## 3. Multi-Approach: Architectural Enforcement Layers

Rate limiting can be enforced at multiple tiers in an enterprise request lifecycle.

```
                              ENFORCEMENT LAYERS
       Client Request
             │
             ▼
[Approach 1: API Gateway Layer]  ──> Spring Cloud Gateway (DDoS / Global IP Shield)
             │
             ▼
[Approach 2: Servlet Filter Layer] ─> OncePerRequestFilter + Bucket4j (Tenant / Auth)
             │
             ▼
[Approach 3: Method Aspect Layer] ──> @RateLimited Controller Aspect (Heavy Endpoints)
             │
             ▼
      Backend Service
             │
             ▼
[Approach 4: Egress Client Layer] ──> Outbound RestClient Shaper (3rd-Party APIs)
```

---

### Approach 1: Ingress API Gateway Rate Limiting (Spring Cloud Gateway + Reactive Redis)

#### Scenario
Distributed microservice architectures where malicious floods and abusive traffic must be dropped at the perimeter before touching internal application pods.

#### Use-Case
Dropping unauthenticated brute-force requests at the edge; global IP protection.

#### Benefits Over Others / Trade-offs
- **Benefits**: Non-blocking Netty event loop; stops abusive traffic before entering the internal network mesh; zero CPU impact on downstream microservices.
- **Trade-offs**: Limited access to deep relational business rules or complex domain entities.

#### Well-Documented Code Example

```xml
<!-- pom.xml inside Spring Cloud Gateway Project -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import reactor.core.publisher.Mono;

@Configuration
public class GatewayPerimeterRateLimiterConfig {

    /**
     * Resolves client identity from X-Forwarded-For or Remote Socket Address.
     */
    @Bean
    @Primary
    public KeyResolver ipAddressKeyResolver() {
        return exchange -> {
            String ip = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
            if (ip != null && !ip.isBlank()) {
                return Mono.just(ip.split(",")[0].trim());
            }
            return Mono.justOrEmpty(exchange.getRequest().getRemoteAddress())
                .map(addr -> addr.getAddress().getHostAddress())
                .defaultIfEmpty("anonymous");
        };
    }

    @Bean
    public RouteLocator configuredRoutes(RouteLocatorBuilder builder, KeyResolver ipAddressKeyResolver) {
        return builder.routes()
            .route("order-service-route", r -> r
                .path("/api/v1/orders/**")
                .filters(f -> f
                    .requestRateLimiter(c -> {
                        // replenishRate: 20 tokens/sec; burstCapacity: 50 tokens
                        c.setRateLimiter(new RedisRateLimiter(20, 50));
                        c.setKeyResolver(ipAddressKeyResolver);
                    })
                )
                .uri("lb://order-service")
            )
            .build();
    }
}
```

---

### Approach 2: Servlet Filter Layer Enforcement (Bucket4j + Distributed Lettuce Proxy)

#### Scenario
Monolithic Spring Boot applications or domain microservices enforcing fine-grained user subscription tiers (Free, Pro, Enterprise) based on authenticated user IDs.

#### Use-Case
Enforcing customer subscription quotas across multiple stateless pods.

#### Benefits Over Others / Trade-offs
- **Benefits**: Direct access to Spring Security `Authentication`, headers, and user session attributes; returns customized RFC 7807 problem payloads.
- **Trade-offs**: Incoming requests must pass through the servlet container before rejection.

#### Well-Documented Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-redis</artifactId>
    <version>8.10.1</version>
</dependency>
```

```java
package com.example.ratelimit.filter;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.BucketConfiguration;
import io.github.bucket4j.ConsumptionProbe;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.security.Principal;
import java.time.Duration;

@Component
public class DistributedTenantRateLimitingFilter extends OncePerRequestFilter {

    private final ProxyManager<String> proxyManager;

    public DistributedTenantRateLimitingFilter(ProxyManager<String> proxyManager) {
        this.proxyManager = proxyManager;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {

        // 1. Resolve Client Key (User ID -> API Key -> Remote IP)
        String clientKey = resolveClientKey(request);

        // 2. Resolve Bucket Configuration (e.g. Pro tier: 1,000 req/hr)
        BucketConfiguration bucketConfig = BucketConfiguration.builder()
            .addLimit(Bandwidth.builder()
                .capacity(1000)
                .refillGreedy(1000, Duration.ofHours(1))
                .build())
            .build();

        // 3. Atomically consume token from Redis
        Bucket bucket = proxyManager.builder().build(clientKey, () -> bucketConfig);
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        if (probe.isConsumed()) {
            // Success: attach telemetry headers
            response.setHeader("X-RateLimit-Limit", "1000");
            response.setHeader("X-RateLimit-Remaining", String.valueOf(probe.getRemainingTokens()));
            filterChain.doFilter(request, response);
        } else {
            // Failure: 429 Too Many Requests
            long waitSeconds = probe.getNanosToWaitForRefill() / 1_000_000_000L;
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("Retry-After", String.valueOf(waitSeconds));
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {
                    "status": 429,
                    "error": "Too Many Requests",
                    "message": "Hourly quota exceeded. Please retry after %d seconds."
                }
            """.formatted(waitSeconds));
        }
    }

    private String resolveClientKey(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null && !apiKey.isBlank()) return "apikey:" + apiKey;

        Principal principal = request.getUserPrincipal();
        if (principal != null) return "user:" + principal.getName();

        return "ip:" + request.getRemoteAddr();
    }
}
```

---

### Approach 3: Method-Level Declarative AOP (`@RateLimited` Custom Aspect)

#### Scenario
Specific heavy endpoints (exporting large CSV reports, recalculating tax balances, PDF rendering) requiring custom rate limits far stricter than standard endpoints.

#### Use-Case
Protecting `/api/v1/reports/export` with a maximum limit of 5 requests per hour.

#### Benefits Over Others / Trade-offs
- **Benefits**: Fine-grained, endpoint-specific configuration; keeps rate-limiting rules co-located with controller methods.
- **Trade-offs**: Incurs Spring AOP reflection overhead; subject to self-invocation bypass.

#### Well-Documented Code Example

```java
package com.example.ratelimit.aop;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimited {
    long capacity();
    long refillTokens();
    long refillDurationSeconds();
    String keyPrefix() default "endpoint";
}
```

```java
package com.example.ratelimit.aop;

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

@Aspect
@Component
public class DeclarativeRateLimitAspect {

    private final ProxyManager<String> proxyManager;

    public DeclarativeRateLimitAspect(ProxyManager<String> proxyManager) {
        this.proxyManager = proxyManager;
    }

    @Around("@annotation(rateLimited)")
    public Object enforceLimit(ProceedingJoinPoint pjp, RateLimited rateLimited) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String clientIp = request.getRemoteAddr();
        String bucketKey = rateLimited.keyPrefix() + ":" + clientIp;

        BucketConfiguration config = BucketConfiguration.builder()
            .addLimit(Bandwidth.builder()
                .capacity(rateLimited.capacity())
                .refillGreedy(rateLimited.refillTokens(), Duration.ofSeconds(rateLimited.refillDurationSeconds()))
                .build())
            .build();

        Bucket bucket = proxyManager.builder().build(bucketKey, () -> config);

        if (!bucket.tryConsume(1)) {
            throw new ResponseStatusException(HttpStatus.TOO_MANY_REQUESTS, "Endpoint specific quota exceeded");
        }

        return pjp.proceed();
    }
}
```

```java
// Controller Usage
@RestController
@RequestMapping("/api/v1/analytics")
public class HeavyReportController {

    @PostMapping("/export")
    @RateLimited(capacity = 5, refillTokens = 5, refillDurationSeconds = 3600, keyPrefix = "export")
    public ResponseEntity<byte[]> exportMassiveReport() {
        return ResponseEntity.ok(new byte[0]);
    }
}
```

---

### Approach 4: Client-Side Egress Rate Limiting (Outbound 3rd-Party Gateway Protection)

#### Scenario
A Spring Boot application calling an external vendor API (e.g., Twilio, OpenAI, Stripe) where the vendor bills heavily for breaches or strictly enforces client rate limits.

#### Use-Case
Throttling outbound SMS or AI model queries to stay within third-party SLA contracts.

#### Benefits Over Others / Trade-offs
- **Benefits**: Prevents 429 penalties and account bans from upstream partners.
- **Trade-offs**: Slows down internal caller threads when outbound limits are reached.

#### Well-Documented Code Example

```java
package com.example.ratelimit.egress;

import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

import java.time.Duration;

@Service
public class ThirdPartyVendorClient {

    private final RestClient restClient;
    private final RateLimiter outboundLimiter;

    public ThirdPartyVendorClient(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("https://api.partner.com").build();

        // Enforce maximum 10 requests per second to external vendor
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .limitForPeriod(10)
            .timeoutDuration(Duration.ofMillis(500)) // Wait up to 500ms for a permit before failing
            .build();

        this.outboundLimiter = RateLimiter.of("partnerGatewayLimiter", config);
    }

    public String callPartnerApi() {
        // Decorate call with Resilience4j RateLimiter
        return RateLimiter.decorateSupplier(outboundLimiter, () -> 
            restClient.get()
                .uri("/v1/telemetry")
                .retrieve()
                .body(String.class)
        ).get();
    }
}
```

---

### Architectural Enforcement Comparison Matrix

| Enforcement Tier | Primary Location | Access to Domain Entity? | Performance Overhead | Best For |
|---|---|---|---|---|
| **Approach 1: Gateway** | Edge / Gateway | No (Header only) | **Minimal (< 0.5ms)** | Perimeter DDoS, Global IP rules |
| **Approach 2: Servlet Filter**| Application Ingress | Yes (Auth Context) | Low (~1ms) | Tenant & Subscription Tiers |
| **Approach 3: Method AOP** | Controller Action | **Full Domain Access** | Moderate (Reflection) | Expensive exports, PDF generators |
| **Approach 4: Egress Limiter** | Outbound Client | N/A (Client Side) | Low | Vendor API SLA Compliance |

---

## 4. Multi-Approach: Client Identification & Key Resolution Strategies

A rate limiter is only as reliable as its client identification algorithm.

---

### Approach 1: Client IP Resolution with Reverse-Proxy Validation (`X-Forwarded-For`)

#### Scenario
Public unauthenticated endpoints (signup forms, password resets, product catalogs).

#### Hazards & Header Spoofing:
A malicious client can send a fake header: `X-Forwarded-For: 1.1.1.1`. If your code naively reads the first IP in the chain, attackers can bypass rate limits simply by changing this header on every request!

#### Secure IP Extraction:
Always configure Tomcat or Spring to use `RemoteIpValve` or take the **last untrusted proxy IP** in the chain.

---

### Approach 2: Authenticated User Principal & JWT Claim Resolution

#### Scenario
Secured REST APIs with token-based authentication (OAuth2 / OpenID Connect).

#### Benefits:
- Multiple users behind the same corporate office NAT/VPN do not block each other.
- Plan tiers (`FREE`, `ENTERPRISE`) can be embedded directly inside the JWT payload, allowing $O(1)$ tier resolution without database lookups.

---

### Approach 3: API Key & Organization Token Resolution (`X-API-Key`)

#### Scenario
B2B developer platforms and webhook ingestion gateways.

#### Key Feature:
Rate limits apply to the **organization account**, ensuring that 5 developers using the same API key share the corporate quota.

---

### Approach 4: Multi-Dimensional Compound Key Resolution (`tenant:tier:endpoint:client`)

#### Scenario
Large enterprise platforms with hierarchical quotas.

#### Example Key Construction:
`tenant:104::tier:PRO::endpoint:SEARCH::user:881`

---

### Key Resolution Comparison Matrix

| Strategy | Security Robustness | NAT Safe? | Requires Auth? | Best For |
|---|---|---|---|---|
| **Approach 1: Client IP** | Moderate (Spoof risk) | **No** (Collides on NAT) | No | Unauthenticated Public APIs |
| **Approach 2: JWT Claim** | **High (Cryptographic)**| **Yes** | Yes | Mobile & SPA Web Apps |
| **Approach 3: API Key** | **High (Hashed Token)** | **Yes** | Yes | B2B Partner Integrations |
| **Approach 4: Compound** | **Highest** | **Yes** | Yes | Multi-Tenant Enterprise SaaS |

---

## 5. Multi-Approach: Resilience Patterns with Resilience4j

Resilience4j provides complementary traffic management patterns that work in tandem with rate limiters.

---

### Approach 1: Inbound Rate Limiter (Throttling Excess Demand)
Rejects excess requests with HTTP 429 when client request frequency exceeds contract limits.

---

### Approach 2: Circuit Breaker (`CLOSED` -> `OPEN` -> `HALF_OPEN` Tripping)

#### Scenario
Protecting your service when a downstream dependency (e.g., Payment Gateway) begins timing out or failing.

#### States:
- **CLOSED**: Normal operation.
- **OPEN**: Failure rate exceeds threshold (e.g., > 50%). All requests fail immediately with fallback without hitting the downstream server.
- **HALF_OPEN**: Periodically sends probe requests to determine if the downstream service has recovered.

#### Well-Documented Code Example

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        slidingWindowSize: 20
        failureRateThreshold: 50
        waitDurationInOpenState: 15s
        permittedNumberOfCallsInHalfOpenState: 5
```

```java
package com.example.resilience;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;

@Service
public class ResilientPaymentService {

    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "executePaymentFallback")
    public String executePayment(Double amount) {
        // External HTTP call
        return "SUCCESS";
    }

    public String executePaymentFallback(Double amount, Throwable t) {
        return "DEGRADED: Payment queued for retry.";
    }
}
```

---

### Approach 3: Bulkhead Isolation (Semaphore vs. Thread-Pool Isolation)

#### Scenario
Preventing one slow endpoint from consuming all 200 Tomcat servlet worker threads.

- **Semaphore Bulkhead**: Limits concurrent executions across the same thread.
- **ThreadPool Bulkhead**: Dispatches tasks to an isolated thread pool with its own queue.

---

### Approach 4: Soft Throttling & Adaptive Concurrency (Queue Delay vs. Hard 429)

#### Scenario
Internal batch tasks or message consumers where requests should not be rejected, but gracefully delayed.

---

### Resilience Patterns Comparison Matrix

| Pattern | Direction | Trigger Condition | Reaction |
|---|---|---|---|
| **Rate Limiter** | Inbound / Outbound | High request volume | Rejects with **HTTP 429** |
| **Circuit Breaker** | Outbound | Downstream error rate > 50% | **Fast-fails** to fallback |
| **Bulkhead** | Internal Threads | Concurrent threads saturated | Restricts execution pool |
| **Throttling** | Inbound / Outbound | High queue depth | **Delays / Sleeps** |

---

## 6. Multi-Approach: Distributed Redis Coordination Backends

---

### Approach 1: Bucket4j with Lettuce CAS Proxy Manager

#### Scenario
Production Spring Boot clusters running on Redis where token bucket evaluation requires zero custom Lua scripts.

```java
package com.example.ratelimit.redis;

import io.github.bucket4j.distributed.ExpirationAfterWriteStrategy;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import io.github.bucket4j.redis.lettuce.cas.LettuceBasedProxyManager;
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.codec.ByteArrayCodec;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;

import java.time.Duration;

@Configuration
public class DistributedLettuceBucket4jConfig {

    @Bean
    public ProxyManager<String> lettuceProxyManager(LettuceConnectionFactory connectionFactory) {
        RedisClient client = (RedisClient) connectionFactory.getNativeClient();
        StatefulRedisConnection<byte[], byte[]> connection = client.connect(ByteArrayCodec.INSTANCE);

        return LettuceBasedProxyManager.builderFor(connection)
            .withExpirationAfterWriteStrategy(
                ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(Duration.ofHours(2))
            )
            .build();
    }
}
```

---

### Approach 2: Atomic Redis Lua Scripting (Single-Roundtrip Rolling Sorted Set)

#### Scenario
Custom Redis setups where adding third-party Bucket4j dependencies is forbidden.

```lua
-- sliding_window_rate_limiter.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local clearBefore = now - window

-- 1. Remove expired timestamps
redis.call('ZREMRANGEBYSCORE', key, 0, clearBefore)

-- 2. Count current elements in window
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    -- 3. Add current timestamp
    redis.call('ZADD', key, now, now .. '-' .. math.random(100000))
    redis.call('PEXPIRE', key, window)
    return 1 -- Allowed
else
    return 0 -- Denied
end
```

---

### Distributed Coordination Comparison Matrix

| Backend Strategy | Maintenance Overhead | Operational Flexibility | Atomicity Guarantee |
|---|---|---|---|
| **Approach 1: Bucket4j + Lettuce** | **Lowest (Library handles CAS)** | High (Built-in bandwidths) | Redis Compare-And-Swap |
| **Approach 2: Redis Lua Script** | Moderate (Script maintenance) | Custom sliding set | **100% Single-Threaded Atomic** |

---

## 7. Production Monitoring, Micrometer Telemetry, & Alerting

```java
package com.example.ratelimit.telemetry;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Component;

@Component
public class RateLimitMetricsTracker {

    private final MeterRegistry registry;

    public RateLimitMetricsTracker(MeterRegistry registry) {
        this.registry = registry;
    }

    public void recordAllowed(String tier) {
        registry.counter("api.ratelimit.requests", "status", "allowed", "tier", tier).increment();
    }

    public void recordRejected(String tier, String clientKey) {
        registry.counter("api.ratelimit.requests", "status", "rejected", "tier", tier).increment();
    }
}
```

---

## 8. Interview Traps, Anti-Patterns, & Gotchas

### Trap 1: In-Memory Rate Limiting in Kubernetes
Using `ConcurrentHashMap` in a multi-pod Kubernetes deployment gives clients $N \times \text{limit}$ requests because each pod maintains an isolated counter. Always use distributed Redis coordination.

### Trap 2: The `bucket.consume()` Thread Starvation Trap
Calling `bucket.consume(1)` blocks the calling thread until tokens are refilled. Under heavy load, this stalls all 200 Tomcat servlet threads, crashing the server. Always use non-blocking `bucket.tryConsume()`.

### Trap 3: Unbounded Memory Growth on IP Key Stores
Storing buckets in memory using raw client IP strings without eviction causes a memory leak during DDoS attacks. Always back local stores with Caffeine caches featuring `expireAfterAccess()`.

### Trap 4: Naive Reverse Proxy IP Extraction
Blindly trusting `request.getHeader("X-Forwarded-For")` allows attackers to bypass rate limits by spoofing random IPs in request headers.

### Trap 5: Confusing Rate Limiting with Circuit Breaking
Rate limiting restricts inbound traffic volume; circuit breakers protect against failing downstream dependencies.

---

## 9. Summary & Architectural Decision Trees

```
                       TRAFFIC CONTROL DECISION TREE
                                     │
      ┌──────────────────────────────┴──────────────────────────────┐
      ▼                                                             ▼
[Inbound Traffic Control]                                     [Outbound Dependency Health]
  ├── Edge / Perimeter  ──> Spring Cloud Gateway + Redis        ├── Failing Downstream ──> Circuit Breaker (Resilience4j)
  ├── User Tiers (App)  ──> Bucket4j + Redis Proxy              ├── Slowdown Worker    ──> Bulkhead Isolation
  └── Heavy Methods     ──> @RateLimited Aspect AOP             └── External SLA Limit ──> Egress RateLimiter
```

---

## 10. Practice Questions & Interview Scenarios

1. **The Cloud Cluster Leak**: An application deployed on 4 Kubernetes pods has an in-memory rate limit of 100 requests/minute. Why are power users able to execute up to 400 requests/minute?
   - *Answer*: Each pod maintains an independent in-memory counter. Load balancers distribute requests across pods. Resolving this requires shared state via Bucket4j Redis or Spring Cloud Gateway.

2. **Token Bucket vs Leaky Bucket**: A partner API requires constant traffic flow without any micro-bursts to protect legacy hardware. Which algorithm should be chosen?
   - *Answer*: Leaky Bucket. While Token Bucket permits bursts up to capacity, Leaky Bucket enforces a strictly uniform egress rate.

3. **HTTP 429 Headers**: What response headers must accompany an HTTP 429 response to enable standard client backoff?
   - *Answer*: `Retry-After` (wait duration in seconds), `X-RateLimit-Limit`, `X-RateLimit-Remaining` (0), and `X-RateLimit-Reset` (Unix timestamp).
