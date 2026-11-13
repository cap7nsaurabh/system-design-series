# 15 — Observability, Monitoring & Actuator in Spring Boot 3.x

> **Module Goal:** Master modern enterprise observability in Spring Boot 3.x (Java 17/21). From securing Actuator endpoints and multi-phase Kubernetes lifecycle probes, to unified metrics and tracing via `ObservationRegistry`, Prometheus alerting rules, OpenTelemetry bridges, structured JSON logging with MDC trace correlation, and high-cardinality hazard mitigation.

---

## Table of Contents

1. [Architectural Foundations: The Three Pillars of Observability](#1-architectural-foundations-the-three-pillars-of-observability)
2. [Application Health & Readiness Assessment Approaches](#2-application-health--readiness-assessment-approaches)
   - [Approach 1: Standard Synchronous `HealthIndicator`](#approach-1-standard-synchronous-healthindicator)
   - [Approach 2: Reactive Non-Blocking `ReactiveHealthIndicator`](#approach-2-reactive-non-blocking-reactivehealthindicator)
   - [Approach 3: Kubernetes Native Lifecycle Probes (`LivenessState` & `ReadinessState`)](#approach-3-kubernetes-native-lifecycle-probes-livenessstate--readinessstate)
   - [Approach 4: Grouped Health Contributors (`CompositeHealthContributor`)](#approach-4-grouped-health-contributors-compositehealthcontributor)
3. [Metrics Collection & Instrumentation Strategies](#3-metrics-collection--instrumentation-strategies)
   - [Approach 1: Programmatic Instrumentation via `MeterRegistry`](#approach-1-programmatic-instrumentation-via-meterregistry)
   - [Approach 2: Declarative AOP Method Instrumentation (`@Timed` & `@Counted`)](#approach-2-declarative-aop-method-instrumentation-timed--counted)
   - [Approach 3: Spring Boot 3 Unified `ObservationRegistry` & `ObservationHandler`](#approach-3-spring-boot-3-unified-observationregistry--observationhandler)
   - [Approach 4: Active Job Duration Tracking via `LongTaskTimer`](#approach-4-active-job-duration-tracking-via-longtasktimer)
4. [Distributed Tracing & Context Propagation Approaches](#4-distributed-tracing--context-propagation-approaches)
   - [Approach 1: OpenTelemetry Standard Bridge (`micrometer-tracing-bridge-otel`)](#approach-1-opentelemetry-standard-bridge-micrometer-tracing-bridge-otel)
   - [Approach 2: Brave / Zipkin Bridge (`micrometer-tracing-bridge-brave`)](#approach-2-brave--zipkin-bridge-micrometer-tracing-bridge-brave)
   - [Approach 3: Programmatic Spans & Distributed Baggage Propagation](#approach-3-programmatic-spans--distributed-baggage-propagation)
   - [Approach 4: ThreadContext Propagation Across `@Async` & Reactive Streams](#approach-4-threadcontext-propagation-across-async--reactive-streams)
5. [Structured Logging & Runtime Diagnostics](#5-structured-logging--runtime-diagnostics)
   - [Approach 1: Structured JSON Logging with MDC Correlation](#approach-1-structured-json-logging-with-mdc-correlation)
   - [Approach 2: Dynamic Runtime Log Level Mutation via Actuator](#approach-2-dynamic-runtime-log-level-mutation-via-actuator)
6. [Prometheus AlertManager & Grafana Dashboard Architectures](#6-prometheus-alertmanager--grafana-dashboard-architectures)
7. [Comprehensive Configuration Reference](#7-comprehensive-configuration-reference)
8. [⚠️ Interview Traps & Production Gotchas](#️-interview-traps--production-gotchas)
9. [📝 Architectural Comparison Matrix](#-architectural-comparison-matrix)
10. [❓ Practice Questions & Real-World Scenarios](#-practice-questions--real-world-scenarios)

---

## 1. Architectural Foundations: The Three Pillars of Observability

In distributed cloud architectures, answering *"Why did order #9821 take 4.8 seconds?"* requires three correlated telemetry streams:

```
                           The Observability Triad
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
        Metrics                    Logs                     Traces
   - Numeric aggregations    - Discrete, immutable     - Request journey DAG
   - Low storage footprint     event statements          across service hops
   - Fast time-series alerts - High detail/debugging   - Identifies bottlenecks
   - "500 errors spiked"     - "NullPointer in svc B"  - "DB lock held for 4s"
           │                         │                         │
           └─────────────────────────┼─────────────────────────┘
                                     ▼
                  Unified Correlation via Trace ID & MDC
```

In Spring Boot 3.x, **Micrometer 1.11+** and the new **Observation API** unify metrics and distributed traces into a single instrumentation call.

---

## 2. Application Health & Readiness Assessment Approaches

Health endpoints indicate whether an application container is functioning, whether its downstream dependencies are operational, and whether Kubernetes should route ingress traffic to the pod.

---

### Approach 1: Standard Synchronous `HealthIndicator`

#### Scenario
Traditional Spring MVC applications with blocking relational database (JDBC/HikariCP) or cache dependencies that must report health status to internal dashboards or monitoring scrapers.

#### Use-Case
Verifying connectivity to critical synchronous backing services (payment gateways, legacy SOAP services, LDAP) on an administrative `/actuator/health` endpoint.

#### Benefits Over Others
- **Simplicity:** Minimal boilerplate; simply return `Health.up()` or `Health.down()`.
- **Automatic Aggregation:** Spring Boot aggregates all `HealthIndicator` beans into a single top-level health status (`UP`, `DOWN`, `OUT_OF_SERVICE`).

#### Production-Grade Code Example

```java
package com.example.observability.health.standard;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.time.Duration;

@Component("paymentGatewayHealth")
@RequiredArgsConstructor
@Slf4j
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final RestClient restClient;

    @Override
    public Health health() {
        long start = System.currentTimeMillis();
        try {
            // Lightweight ping with short timeout
            String status = restClient.get()
                    .uri("https://api.paymentgateway.internal/health/ping")
                    .retrieve()
                    .body(String.class);

            long latencyMs = System.currentTimeMillis() - start;

            if ("PONG".equalsIgnoreCase(status)) {
                return Health.up()
                        .withDetail("provider", "Stripe-Private-Link")
                        .withDetail("latencyMs", latencyMs)
                        .withDetail("status", "OPERATIONAL")
                        .build();
            } else {
                return Health.down()
                        .withDetail("error", "Unexpected ping response: " + status)
                        .withDetail("latencyMs", latencyMs)
                        .build();
            }
        } catch (Exception ex) {
            log.warn("Payment gateway health probe failed: {}", ex.getMessage());
            return Health.down(ex)
                    .withDetail("provider", "Stripe-Private-Link")
                    .withDetail("failureReason", ex.getMessage())
                    .build();
        }
    }
}
```

---

### Approach 2: Reactive Non-Blocking `ReactiveHealthIndicator`

#### Scenario
High-concurrency Spring WebFlux microservices running on Project Reactor and Netty where health checks must never block event-loop worker threads.

#### Use-Case
Health checks for reactive drivers like R2DBC (reactive PostgreSQL/MySQL), Reactive Redis, or non-blocking HTTP endpoints.

#### Benefits Over Others
- **Zero Event-Loop Blocking:** Avoids freezing Netty threads with synchronous timeouts.
- **Native Publisher Composition:** Returns `Mono<Health>` for seamless integration into reactive pipelines.

#### Production-Grade Code Example

```java
package com.example.observability.health.reactive;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.ReactiveHealthIndicator;
import org.springframework.data.redis.connection.ReactiveRedisConnectionFactory;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.time.Duration;

@Component("reactiveRedisHealth")
@RequiredArgsConstructor
public class ReactiveRedisHealthIndicator implements ReactiveHealthIndicator {

    private final ReactiveRedisConnectionFactory connectionFactory;

    @Override
    public Mono<Health> health() {
        return connectionFactory.getReactiveConnection()
                .ping()
                .timeout(Duration.ofMillis(800))
                .map(response -> Health.up()
                        .withDetail("redisResponse", response)
                        .withDetail("mode", "ReactiveCluster")
                        .build())
                .onErrorResume(ex -> Mono.just(Health.down(ex)
                        .withDetail("error", "Redis reactive ping timed out or failed")
                        .build()));
    }
}
```

---

### Approach 3: Kubernetes Native Lifecycle Probes (`LivenessState` & `ReadinessState`)

#### Scenario
Containerized microservices running in Kubernetes where container restarts (liveness) must be separated from traffic routing (readiness).

#### Use-Case
Preventing restart loops when external dependencies go down. If the database crashes, the application is **not ready** (stop sending traffic), but it is still **alive** (do not restart the container).

#### Benefits Over Others
- **Prevents Cascading Restart Storms:** Protects against restarting 100 pods simultaneously when a downstream database experiences a transient blip.
- **Automated Graceful Shutdown:** Sets readiness to `REFUSING_TRAFFIC` during shutdown while existing requests finish processing.

#### Production-Grade Code Example

```java
package com.example.observability.health.k8s;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.LivenessState;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class KubernetesLifecycleCoordinator {

    private final ApplicationEventPublisher eventPublisher;

    public void onWarmupComplete() {
        log.info("Application caches preloaded. Notifying Kubernetes to start routing ingress traffic.");
        // Moves /actuator/health/readiness -> UP
        AvailabilityChangeEvent.publish(eventPublisher, this, ReadinessState.ACCEPTING_TRAFFIC);
    }

    public void onCriticalMemoryCorruption() {
        log.error("Irrecoverable internal deadlock detected! Triggering Kubernetes pod restart.");
        // Moves /actuator/health/liveness -> DOWN (Kubernetes restarts the pod)
        AvailabilityChangeEvent.publish(eventPublisher, this, LivenessState.BROKEN);
    }

    public void onDatabaseUnavailable() {
        log.warn("Database connection lost! Refusing incoming HTTP traffic but remaining alive.");
        // Moves /actuator/health/readiness -> OUT_OF_SERVICE (Kubernetes removes pod from Service endpoints)
        AvailabilityChangeEvent.publish(eventPublisher, this, ReadinessState.REFUSING_TRAFFIC);
    }
}
```

---

### Approach 4: Grouped Health Contributors (`CompositeHealthContributor`)

#### Scenario
Large enterprise applications with dozens of backing dependencies (relational database, Redis, S3, ERP, third-party payment gateways) where `/actuator/health` responses must be structured hierarchically.

#### Use-Case
Partitioning health indicators into logical groups so operators can query `/actuator/health/external-apis` or `/actuator/health/core-storage` separately.

#### Benefits Over Others
- **Clean Health Domain Partitioning:** Keeps the health response organized.
- **Targeted Monitoring:** Allows monitoring scrapers to target specific subsystem health groups.

#### Production-Grade Code Example

```java
package com.example.observability.health.composite;

import org.springframework.boot.actuate.health.CompositeHealthContributor;
import org.springframework.boot.actuate.health.HealthContributor;
import org.springframework.boot.actuate.health.NamedContributor;
import org.springframework.stereotype.Component;

import java.util.Iterator;
import java.util.Map;

@Component("thirdPartySubsystems")
public class ThirdPartySubsystemsCompositeHealthContributor implements CompositeHealthContributor {

    private final Map<String, HealthContributor> contributors;

    public ThirdPartySubsystemsCompositeHealthContributor(
            PaymentGatewayHealthIndicator paymentHealth,
            EmailDeliveryHealthIndicator emailHealth) {
        this.contributors = Map.of(
                "paymentGateway", paymentHealth,
                "emailDelivery", emailHealth
        );
    }

    @Override
    public HealthContributor getContributor(String name) {
        return contributors.get(name);
    }

    @Override
    public Iterator<NamedContributor<HealthContributor>> iterator() {
        return contributors.entrySet().stream()
                .map(entry -> NamedContributor.of(entry.getKey(), entry.getValue()))
                .iterator();
    }
}

@Component
class EmailDeliveryHealthIndicator implements org.springframework.boot.actuate.health.HealthIndicator {
    @Override
    public org.springframework.boot.actuate.health.Health health() {
        return org.springframework.boot.actuate.health.Health.up().withDetail("smtp", "CONNECTED").build();
    }
}
```

---

## 3. Metrics Collection & Instrumentation Strategies

Measuring latency, error rates, and resource utilization requires selecting the right Micrometer metric type and avoiding the **high cardinality trap**.

---

### Approach 1: Programmatic Instrumentation via `MeterRegistry`

#### Scenario
Fine-grained business metric tracking (e.g., total monetary value processed, active WebSocket connections, cache hit ratios) requiring custom tags and conditional updates.

#### Use-Case
Instrumenting core business transactions where methods require Counters, Gauges, and Timers with runtime metadata.

#### Benefits Over Others
- **Dynamic Tag Resolution:** Attach low-cardinality metadata (e.g., `payment_type=CREDIT_CARD`, `region=EU`) dynamically.
- **Strongly Typed Metric Primitives:** Direct access to `Counter`, `Timer`, `Gauge`, and `DistributionSummary`.

#### Production-Grade Code Example

```java
package com.example.observability.metrics.registry;

import io.micrometer.core.instrument.*;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.util.concurrent.atomic.AtomicInteger;

@Service
public class BillingMetricsRecorder {

    private final Counter successfulPaymentsCounter;
    private final Counter failedPaymentsCounter;
    private final Timer paymentProcessingTimer;
    private final DistributionSummary paymentAmountDistribution;
    // CRITICAL: Hold a strong reference to the Gauge target object!
    // Weak references in MeterRegistry can cause gauges to report NaN after GC.
    private final AtomicInteger activeCheckoutSessionsGauge;

    public BillingMetricsRecorder(MeterRegistry registry) {
        // 1. Monotonically increasing Counters
        this.successfulPaymentsCounter = Counter.builder("billing.payments.completed")
                .description("Total number of successfully settled payments")
                .tag("environment", "prod")
                .register(registry);

        this.failedPaymentsCounter = Counter.builder("billing.payments.failed")
                .description("Total number of rejected payments")
                .tag("environment", "prod")
                .register(registry);

        // 2. Timer with Percentiles and Histogram Buckets
        this.paymentProcessingTimer = Timer.builder("billing.payment.duration")
                .description("Execution latency for payment gateway settlement")
                .publishPercentiles(0.5, 0.95, 0.99) // P50, P95, P99 client-side
                .publishPercentileHistogram()        // Required for histogram_quantile() in Prometheus
                .register(registry);

        // 3. Distribution Summary for Payload/Financial Sizes (non-time data)
        this.paymentAmountDistribution = DistributionSummary.builder("billing.payment.amount")
                .description("Distribution of settled payment amounts")
                .baseUnit("USD")
                .publishPercentiles(0.5, 0.9, 0.99)
                .register(registry);

        // 4. Gauge for active connection/session count
        this.activeCheckoutSessionsGauge = registry.gauge(
                "billing.sessions.active",
                Tags.of("service", "checkout"),
                new AtomicInteger(0)
        );
    }

    public void recordPaymentSettled(double amount, long durationMs) {
        successfulPaymentsCounter.increment();
        paymentAmountDistribution.record(amount);
        paymentProcessingTimer.record(java.time.Duration.ofMillis(durationMs));
    }

    public void incrementActiveSessions() { activeCheckoutSessionsGauge.incrementAndGet(); }
    public void decrementActiveSessions() { activeCheckoutSessionsGauge.decrementAndGet(); }
}
```

---

### Approach 2: Declarative AOP Method Instrumentation (`@Timed` & `@Counted`)

#### Scenario
Standard controller and service methods where developers want to track execution latency and call counts without boilerplate code.

#### Use-Case
Monitoring REST API endpoints and service methods with minimal intrusion on business logic.

#### Benefits Over Others
- **Zero Business Code Pollution:** Purely declarative via annotations.
- **Built-in Error Tagging:** `@Timed` automatically tags exceptions thrown by the method.

#### Production-Grade Code Example

```java
package com.example.observability.metrics.declarative;

import io.micrometer.core.annotation.Counted;
import io.micrometer.core.annotation.Timed;
import io.micrometer.core.aop.CountedAspect;
import io.micrometer.core.aop.TimedAspect;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Configuration
class MetricAspectConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) { return new TimedAspect(registry); }
    @Bean
    public CountedAspect countedAspect(MeterRegistry registry) { return new CountedAspect(registry); }
}

@RestController
@RequestMapping("/api/v1/catalog")
public class CatalogController {

    @GetMapping("/products")
    @Timed(
            value = "catalog.products.request.time",
            description = "Time taken to fetch catalog products",
            percentiles = {0.5, 0.95, 0.99},
            histogram = true
    )
    @Counted(
            value = "catalog.products.invocations",
            description = "Total requests received for product catalog"
    )
    public String listProducts() {
        return "Product list payload";
    }
}
```

---

### Approach 3: Spring Boot 3 Unified `ObservationRegistry` & `ObservationHandler`

#### Scenario
Modern Spring Boot 3.x microservices where every monitored operation should produce **both a metric and a distributed tracing span** simultaneously from a single API call.

#### Use-Case
The recommended modern standard for instrumentation in Spring Boot 3.x, replacing separate Micrometer and Sleuth/Tracing calls.

#### Benefits Over Others
- **Unified Instrumentation:** A single `Observation.observe()` call registers a timer metric in Prometheus AND starts a span in OpenTelemetry/Zipkin.
- **Cardinaity Protection:** Supports `lowCardinalityKeyValue` (metrics + spans) and `highCardinalityKeyValue` (spans only), preventing high-cardinality memory leaks in Prometheus.

#### Production-Grade Code Example

```java
package com.example.observability.metrics.observation;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class UnifiedOrderProcessingService {

    private final ObservationRegistry observationRegistry;

    public String processOrder(String orderId, String customerTier, double amount) {
        // Unified Observation: produces both a Timer metric AND an OpenTelemetry Span!
        return Observation.createNotStarted("order.processing", observationRegistry)
                // Low cardinality: Goes to both Prometheus metrics AND Tracing spans
                .lowCardinalityKeyValue("customer.tier", customerTier)
                .lowCardinalityKeyValue("currency", "USD")
                // High cardinality: Goes ONLY to Tracing spans (protects Prometheus from OOM!)
                .highCardinalityKeyValue("order.id", orderId)
                .highCardinalityKeyValue("amount.exact", String.valueOf(amount))
                .observe(() -> {
                    log.info("Executing order settlement for order '{}'", orderId);
                    // Business logic...
                    return "ORDER-PROCESSED-" + orderId;
                });
    }
}
```

---

### Approach 4: Active Job Duration Tracking via `LongTaskTimer`

#### Scenario
Asynchronous batch processing jobs, CSV data imports, or nightly database re-indexing tasks that run for minutes or hours.

#### Use-Case
Standard `Timer` only records duration *after* the task completes. If a 30-minute batch job gets stuck in an infinite loop, a standard `Timer` reports nothing. `LongTaskTimer` reports active task duration **while the task is running**, enabling alerts on stuck jobs.

#### Benefits Over Others
- **In-Flight Duration Visibility:** Query `long_task_timer_duration_seconds` to see how long an active job has been running right now.

#### Production-Grade Code Example

```java
package com.example.observability.metrics.longtask;

import io.micrometer.core.instrument.LongTaskTimer;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class NightlySettlementBatchJob {

    private final LongTaskTimer batchJobTimer;

    public NightlySettlementBatchJob(MeterRegistry registry) {
        this.batchJobTimer = LongTaskTimer.builder("batch.settlement.job")
                .description("Measures active execution duration of the nightly settlement job")
                .tag("job_type", "reconciliation")
                .register(registry);
    }

    @Scheduled(cron = "0 0 2 * * *") // Daily at 2 AM
    public void executeSettlement() {
        LongTaskTimer.Sample activeSample = batchJobTimer.start();
        try {
            log.info("Starting nightly ledger reconciliation...");
            // Long running batch execution...
        } finally {
            activeSample.stop();
            log.info("Completed nightly ledger reconciliation.");
        }
    }
}
```

---

## 4. Distributed Tracing & Context Propagation Approaches

Distributed tracing links a user's initial HTTP request across all participating downstream microservices, databases, and message brokers using a shared **Trace ID**.

```
Client Request
      │
      ▼
┌──────────────┐     HTTP Headers (traceparent)     ┌──────────────┐
│  API Gateway │ ─────────────────────────────────▶ │ OrderService │
│ TraceId: A1  │                                    │ TraceId: A1  │
│ SpanId: 001  │                                    │ SpanId: 002  │
└──────────────┘                                    └──────┬───────┘
                                                           │ Kafka Header
                                                           ▼
                                                    ┌──────────────┐
                                                    │ StockWorker  │
                                                    │ TraceId: A1  │
                                                    │ SpanId: 003  │
                                                    └──────────────┘
```

---

### Approach 1: OpenTelemetry Standard Bridge (`micrometer-tracing-bridge-otel`)

#### Scenario
Modern cloud-native Kubernetes deployments adopting the CNCF OpenTelemetry (OTel) standard for distributed tracing.

#### Use-Case
Sending traces to OpenTelemetry Collectors, Jaeger, Datadog, or Grafana Tempo via standard OTLP (OpenTelemetry Protocol).

#### Benefits Over Others
- **Vendor Agnostic:** The global standard for distributed observability.
- **W3C `traceparent` Compliance:** Natively propagates W3C trace context headers across microservice boundaries.

#### Production-Grade Code Example

```xml
<!-- Maven dependencies for OpenTelemetry with Spring Boot 3 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 0.1 # Sample 10% of requests in production
  otlp:
    tracing:
      endpoint: "http://otel-collector.monitoring.svc:4318/v1/traces"
```

---

### Approach 2: Brave / Zipkin Bridge (`micrometer-tracing-bridge-brave`)

#### Scenario
Legacy or existing microservice fleets using Zipkin and B3 propagation headers (`X-B3-TraceId`, `X-B3-SpanId`).

#### Use-Case
Drop-in replacement for Spring Cloud Sleuth applications upgrading to Spring Boot 3.x while keeping existing Zipkin server infrastructure.

---

### Approach 3: Programmatic Spans & Distributed Baggage Propagation

#### Scenario
A microservice receives a request with an incoming `X-Tenant-ID` or `X-User-ID` header. That metadata must travel across all downstream HTTP and Kafka calls without polluting every Java method signature.

#### Use-Case
Propagating cross-cutting tenant identifiers and creating manual spans for fine-grained internal operations.

#### Benefits Over Others
- **Transparent Propagation:** Baggage fields attach to the trace context and are automatically serialized into downstream HTTP headers.
- **Automatic MDC Integration:** Configured baggage fields appear automatically in application log output.

#### Production-Grade Code Example

```java
package com.example.observability.tracing.baggage;

import io.micrometer.tracing.BaggageField;
import io.micrometer.tracing.Span;
import io.micrometer.tracing.Tracer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class TracedInventoryService {

    private final Tracer tracer;
    // Define baggage field matching application.yml correlation settings
    private static final BaggageField TENANT_FIELD = BaggageField.create("tenantId");

    public boolean reserveStock(String sku, int quantity, String tenantId) {
        // 1. Attach tenant ID to the active distributed trace baggage
        TENANT_FIELD.set(tenantId);

        // 2. Start a custom programmatic span for fine-grained timing
        Span dbCheckSpan = tracer.nextSpan().name("database-inventory-lock").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(dbCheckSpan)) {
            dbCheckSpan.tag("sku", sku);
            dbCheckSpan.tag("quantity", String.valueOf(quantity));
            dbCheckSpan.event("evaluating-warehouse-bins");

            log.info("Checking inventory allocation for SKU '{}' (Baggage tenant: '{}')",
                    sku, TENANT_FIELD.getValue());

            return executeReservation(sku, quantity);

        } catch (Exception ex) {
            dbCheckSpan.error(ex);
            throw ex;
        } finally {
            dbCheckSpan.end(); // Always close spans in finally blocks!
        }
    }

    private boolean executeReservation(String sku, int qty) { return true; }
}
```

---

### Approach 4: ThreadContext Propagation Across `@Async` & Reactive Streams

#### Scenario
When a service method delegates work to an `@Async` thread pool or a reactive Project Reactor pipeline, the `ThreadLocal` context holding the trace ID and MDC logging context is lost.

#### Use-Case
Preserving `traceId` and log correlation across asynchronous thread boundaries.

#### Benefits Over Others
- **Continuous Logging Correlation:** Prevents un-traceable log lines with empty `[traceId=]`.

#### Production-Grade Code Example

```java
package com.example.observability.tracing.async;

import org.slf4j.MDC;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskDecorator;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.Map;
import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class TracedAsyncExecutorConfig implements AsyncConfigurer {

    @Bean(name = "tracedTaskExecutor")
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(16);
        executor.setThreadNamePrefix("traced-async-");
        // TaskDecorator captures caller ThreadLocal state and restores it on worker thread
        executor.setTaskDecorator(new MdcContextPropagatingTaskDecorator());
        executor.initialize();
        return executor;
    }
}

class MdcContextPropagatingTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        // 1. Capture context from the caller thread
        Map<String, String> callerMdc = MDC.getCopyOfContextMap();
        return () -> {
            try {
                // 2. Restore context on the worker thread
                if (callerMdc != null) {
                    MDC.setContextMap(callerMdc);
                }
                runnable.run();
            } finally {
                // 3. Clear worker thread state to prevent thread pool pollution
                MDC.clear();
            }
        };
    }
}
```

---

## 5. Structured Logging & Runtime Diagnostics

Plain-text log files are difficult for log aggregators (ELK, OpenSearch, Loki) to parse. Modern systems emit structured JSON logs with correlated trace IDs.

---

### Approach 1: Structured JSON Logging with MDC Correlation

#### Scenario
Production microservices streaming logs to Elasticsearch, Datadog, or Grafana Loki.

#### Use-Case
Formatting every log statement as a single-line JSON object containing timestamp, log level, thread, logger, trace ID, span ID, and message.

#### Production-Grade Code Example

```xml
<!-- Dependency: logstash-logback-encoder -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="applicationName" source="spring.application.name"/>

    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                            "severity": "%level",
                            "service": "${applicationName:-unknown}",
                            "thread": "%thread",
                            "logger": "%logger{36}",
                            "message": "%message",
                            "traceId": "%mdc{traceId:-}",
                            "spanId": "%mdc{spanId:-}",
                            "tenantId": "%mdc{tenantId:-}"
                        }
                    </pattern>
                </pattern>
                <stackTrace>
                    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                        <maxDepthPerThrowable>30</maxDepthPerThrowable>
                        <rootCauseFirst>true</rootCauseFirst>
                    </throwableConverter>
                </stackTrace>
            </providers>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE"/>
    </root>
</configuration>
```

---

### Approach 2: Dynamic Runtime Log Level Mutation via Actuator

#### Scenario
A production bug occurs, but the service is configured at `INFO` level. Operators need detailed `DEBUG` logs for a specific package without restarting the application or redeploying.

#### Use-Case
Changing log levels dynamically at runtime via HTTP POST to `/actuator/loggers/{name}`.

#### Production-Grade Code Example

```bash
# 1. Inspect current log level for billing package
curl -X GET http://localhost:8080/actuator/loggers/com.example.billing

# Output:
# {"configuredLevel":"INFO","effectiveLevel":"INFO"}

# 2. Dynamically switch billing package to DEBUG at runtime
curl -X POST http://localhost:8080/actuator/loggers/com.example.billing \
     -H "Content-Type: application/json" \
     -d '{"configuredLevel": "DEBUG"}'

# 3. Once incident investigation is complete, revert back to INFO
curl -X POST http://localhost:8080/actuator/loggers/com.example.billing \
     -H "Content-Type: application/json" \
     -d '{"configuredLevel": "INFO"}'
```

---

## 6. Prometheus AlertManager & Grafana Dashboard Architectures

Exposing metrics is only half the battle; alerting on error budgets and Service Level Objectives (SLOs) ensures fast incident response.

### Production Prometheus Rules (`prometheus-alerts.yml`)

```yaml
groups:
  - name: spring-boot-sla-alerts
    rules:
      # Alert when HTTP 5xx error rate exceeds 2% over a 5-minute window
      - alert: HttpErrorRateExceeded
        expr: |
          (
            sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
            /
            sum(rate(http_server_requests_seconds_count[5m]))
          ) > 0.02
        for: 2m
        labels:
          severity: critical
          team: platform-core
        annotations:
          summary: "HTTP 5xx error rate above 2% on {{ $labels.application }}"
          description: "Service is failing {{ $value | humanizePercentage }} of requests."

      # Alert when P99 latency exceeds 1.5 seconds
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application)
          ) > 1.5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency above 1.5s on {{ $labels.application }}"

      # Alert when HikariCP connection pool is saturated
      - alert: DatabasePoolExhaustion
        expr: hikaricp_connections_pending > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "HikariCP connection pool exhausted on {{ $labels.application }}"
          description: "{{ $value }} threads blocked waiting for database connections."
```

---

## 7. Comprehensive Configuration Reference

```yaml
spring:
  application:
    name: order-service

management:
  server:
    port: 8081 # Expose actuator on internal management port
    address: 0.0.0.0
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,loggers
  endpoint:
    health:
      show-details: always
      show-components: always
      probes:
        enabled: true # Enables /actuator/health/liveness and /readiness
      group:
        readiness:
          include: readinessState,db,diskSpace
        liveness:
          include: livenessState
  tracing:
    sampling:
      probability: 0.1 # 10% sampling
    baggage:
      enabled: true
      remote-fields:
        - tenantId
      correlation:
        fields:
          - tenantId
  metrics:
    tags:
      application: ${spring.application.name}
      region: us-east-1
    distribution:
      percentiles-histogram:
        http.server.requests: true # Enables Prometheus histogram buckets
```

---

## 8. ⚠️ Interview Traps & Production Gotchas

### Trap 1: The High Cardinality Metrics OOM Trap

> [!CAUTION]
> Never use unbounded values (User IDs, Order UUIDs, raw email addresses, full URLs with query parameters) as tags in Micrometer metrics:
> ```java
> // ❌ DISASTROUS ANTI-PATTERN: Creates millions of TimeSeries objects in memory!
> registry.counter("http.requests", "user_id", userId); // OOM crash!
> 
> // ✅ CORRECT: Use bounded, finite enumerations for metrics
> registry.counter("http.requests", "user_tier", "PREMIUM");
> // Put the high-cardinality user_id into tracing spans or baggage instead!
> ```

---

### Trap 2: Slow Dependencies in the Liveness Probe

Adding external dependency checks (database queries, remote REST calls) to the **liveness** probe will cause Kubernetes to restart pods during high database load, worsening the outage. Keep liveness probes lightweight (JVM state only); place dependency checks in the **readiness** probe.

---

### Trap 3: Masking Sensitive Properties in `/actuator/env`

Actuator automatically sanitizes values for keys containing `password`, `secret`, or `key`. If a property is named `app.datasource.credential-token`, Actuator will display it in plain text. Always configure `management.endpoint.env.keys-to-sanitize` explicitly.

---

## 9. 📝 Architectural Comparison Matrix

| Dimension | Prometheus Metrics | Distributed Traces (OTel) | Structured JSON Logs |
|---|---|---|---|
| **Primary Question Answered** | "Is the service healthy?" | "Where did time go?" | "Why did a specific line fail?" |
| **Storage Cost** | Very low (fixed per time-series) | Moderate (governed by sample rate) | High (proportional to log volume) |
| **Cardinality Tolerance** | **Extremely Low** (bounded tags only) | **Extremely High** (UUIDs, user IDs) | **High** (full text search) |
| **Retention Window** | Long (15-90 days) | Short (3-14 days) | Medium (14-30 days) |
| **Query Mechanism** | PromQL | Trace Timeline Gantt Chart | Elasticsearch / Lucene / LogQL |

---

## 10. ❓ Practice Questions & Real-World Scenarios

1. **What is the difference between a Kubernetes Liveness probe and a Readiness probe, and how does Spring Boot map them?**
   - *Answer:* Liveness checks if the JVM is functioning (not deadlocked). If it fails, Kubernetes restarts the pod. Readiness checks if the pod can accept user traffic. If it fails, Kubernetes removes the pod from the Service load balancer without restarting it. Spring Boot maps them to `/actuator/health/liveness` and `/actuator/health/readiness`.

2. **Why does high tag cardinality cause OutOfMemoryErrors in Micrometer and Prometheus?**
   - *Answer:* In Prometheus and Micrometer, every unique combination of metric name and tag key-value pairs creates a completely new, distinct `TimeSeries` object in JVM heap and Prometheus storage. Adding 1,000,000 distinct user IDs generates 1,000,000 active time series, exhausting heap memory.

3. **How do you propagate distributed trace IDs across an `@Async` thread boundary?**
   - *Answer:* Because `ThreadLocal` context does not cross thread boundaries automatically, you configure a `TaskDecorator` on the `ThreadPoolTaskExecutor` to copy the MDC map from the calling thread to the executor worker thread before task execution, and clear it in a `finally` block.
