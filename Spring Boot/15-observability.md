# Module 15: Observability, Monitoring & Actuator in Spring Boot

Modern microservices architectures demand robust visibility. When a system spans dozens of services, thousands of instances, and handles millions of requests, answering the simple question "Why is this request slow?" becomes incredibly difficult without the right tools.

This module provides a comprehensive deep dive into **Observability in Spring Boot**. We will cover the "Three Pillars of Observability": **Metrics, Logs, and Traces**, and how Spring Boot Ecosystem (Actuator, Micrometer, Micrometer Tracing) seamlessly integrates with industry-standard tools like Prometheus, Grafana, OpenTelemetry, Zipkin, and the ELK stack.

---

## 1. The Three Pillars of Observability

Before diving into code, let's establish the terminology:

1.  **Metrics**: Aggregated data over a period of time. E.g., "We received 500 requests per second with an average latency of 45ms." Metrics are cheap to store and alert on, but lack granular details of a specific request.
2.  **Logs**: Discrete, immutable records of events that happened over time. E.g., "User 123 failed to log in because of invalid credentials." Logs are detailed but can be expensive to store at high volume.
3.  **Distributed Traces**: A representation of a single request's journey across multiple services. E.g., "Request A took 200ms total: 50ms in Service X, 100ms in Service Y (database query), and 50ms in Service Z."

Spring Boot provides first-class support for all three.

---

## 2. Spring Boot Actuator: The Command Center

Spring Boot Actuator brings production-ready features to your application. It provides built-in endpoints to monitor and manage your application, giving you insights into application health, metrics, environment variables, thread dumps, and more.

### 2.1. Setup and Enabling Endpoints

To use Actuator, add the dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

By default, only `/health` and `/info` are exposed over HTTP. To expose more endpoints, configure your `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        # Expose all endpoints EXCEPT env and threaddump (as an example)
        # Use "*" to expose all, but be careful!
        include: "*"
        exclude: "env,threaddump"
  endpoint:
    health:
      show-details: always # Shows detailed health of database, disk space, etc.
      show-components: always
```

> [!CAUTION]
> Never expose `management.endpoints.web.exposure.include: "*"` on a public-facing interface without authentication. Endpoints like `/env` can leak database passwords or API keys, and `/heapdump` can leak entire memory contents, including user PII.

### 2.2. Key Actuator Endpoints

| Endpoint | Description | Production Use Case |
| :--- | :--- | :--- |
| `/actuator/health` | Application health information. | Used by Load Balancers and Kubernetes probes to know if traffic should be routed here. |
| `/actuator/info` | Arbitrary application info. | Displaying git commit hash, build version, or team ownership. |
| `/actuator/metrics` | Exposes Micrometer metrics. | Querying specific JVM or custom metrics directly. |
| `/actuator/env` | Exposes properties from `Environment`. | Debugging why a property wasn't picked up (e.g., checking if a profile is active). |
| `/actuator/threaddump` | Performs a thread dump. | Debugging deadlocks or threads stuck on I/O. |
| `/actuator/heapdump` | Downloads an `hprof` heap dump file. | Investigating memory leaks offline via Eclipse MAT. |
| `/actuator/loggers` | Shows and modifies log levels. | Dynamically enabling `DEBUG` logs for a specific package without restarting the app. |
| `/actuator/mappings` | Shows all `@RequestMapping` paths. | Discovering what REST endpoints exist on the application. |

### 2.3. Security Implications

Because Actuator endpoints reveal sensitive internal details, they **must** be secured. Often, companies expose actuator on a completely different HTTP port that is only accessible inside the private VPC.

```yaml
management:
  server:
    port: 8081 # Actuator runs on 8081, main app on 8080
    address: 127.0.0.1 # Bind only to localhost (or internal network interface)
```

Alternatively, secure it using Spring Security:

```java
import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.config.Customizer;

@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(authorize -> authorize
                // Allow /health and /info to anyone (Load Balancer)
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                // Require ADMIN role for all other actuator endpoints
                .anyRequest().hasRole("ADMIN")
            )
            .httpBasic(Customizer.withDefaults()); // Basic Auth for scraping/admin
        
        return http.build();
    }
}
```

### 2.4. Custom HealthIndicators

The `/health` endpoint automatically aggregates the status of all `HealthIndicator` beans. Spring Boot provides auto-configured indicators for DataSources, Redis, MongoDB, RabbitMQ, disk space, etc.

You can write a custom `HealthIndicator` to check external dependencies (e.g., a third-party API your service relies on).

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

@Component("paymentGatewayHealth")
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;

    public PaymentGatewayHealthIndicator(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    public Health health() {
        try {
            // Ping the external service
            String status = restTemplate.getForObject("https://api.paymentgateway.com/ping", String.class);
            if ("OK".equalsIgnoreCase(status)) {
                return Health.up()
                             .withDetail("service", "PaymentGateway")
                             .withDetail("latency", "45ms")
                             .build();
            } else {
                return Health.down()
                             .withDetail("service", "PaymentGateway")
                             .withDetail("error", "Received non-OK status: " + status)
                             .build();
            }
        } catch (RestClientException e) {
            return Health.down(e)
                         .withDetail("service", "PaymentGateway")
                         .build();
        }
    }
}
```

### 2.5. Custom InfoContributors

The `/info` endpoint is great for displaying metadata. While you can add static info via `application.yml` under the `info.*` key, you can also inject dynamic info.

```java
import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;
import java.util.Map;

@Component
public class SystemInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("system", Map.of(
            "osName", System.getProperty("os.name"),
            "javaVersion", System.getProperty("java.version"),
            "activeThreads", Thread.activeCount()
        ));
    }
}
```

### 2.6. Kubernetes Probes (Liveness and Readiness)

When running in Kubernetes, k8s needs to know two things:
1.  **Liveness**: Is the container completely dead/deadlocked and needs to be restarted?
2.  **Readiness**: Is the container done initializing and ready to receive HTTP traffic?

Spring Boot Actuator integrates natively with K8s probes. If you run your app in K8s (or set `management.endpoint.health.probes.enabled=true`), Actuator automatically exposes `/actuator/health/liveness` and `/actuator/health/readiness`.

- **Liveness** goes `DOWN` if `LivenessState.BROKEN` is published (e.g., critical internal failure).
- **Readiness** is `DOWN` during startup, goes `UP` when `ApplicationReadyEvent` fires, and can go `DOWN` if the app is overloaded or gracefully shutting down.

Example Kubernetes Deployment config utilizing these endpoints:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
```

You can programmatically change the state:

```java
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.LivenessState;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
public class CircuitBreakerService {

    private final ApplicationEventPublisher eventPublisher;

    public CircuitBreakerService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void onCriticalDatabaseFailure() {
        // Tell Kubernetes we are no longer ready for traffic
        AvailabilityChangeEvent.publish(this.eventPublisher, this, ReadinessState.REFUSING_TRAFFIC);
        
        // OR tell Kubernetes to kill and restart us
        // AvailabilityChangeEvent.publish(this.eventPublisher, this, LivenessState.BROKEN);
    }
}
```

---

## 3. Metrics with Micrometer

### 3.1. What is Micrometer?

Micrometer is to metrics what SLF4J is to logging. It provides a vendor-neutral metrics collection facade. You instrument your code using Micrometer's API, and Spring Boot handles exporting those metrics to a backend of your choice (Prometheus, Datadog, New Relic, etc.).

### 3.2. Core Metric Types

1.  **Counter**: Reports a single value that can only go up. (E.g., Number of orders processed).
2.  **Timer**: Measures both the rate that a short-running event occurs and the distribution of its duration. (E.g., HTTP request latency).
3.  **Gauge**: Shows the current value of something that can go up and down. (E.g., Active sessions, CPU usage, Collection size).
4.  **DistributionSummary**: Tracks the statistical distribution of events. (E.g., The payload size of incoming requests).

### 3.3. Instrumenting Code Programmatically

You can inject `MeterRegistry` to create and update metrics manually:

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;
import java.util.concurrent.atomic.AtomicInteger;
import jakarta.annotation.PostConstruct;

@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;
    private final AtomicInteger activeOrders;

    public OrderService(MeterRegistry registry) {
        // Counter
        this.orderCounter = registry.counter("orders.processed.total", "type", "online");
        
        // Timer
        this.orderTimer = registry.timer("orders.processing.time");
        
        // Gauge
        this.activeOrders = registry.gauge("orders.active.current", new AtomicInteger(0));
    }

    public void processOrder() {
        activeOrders.incrementAndGet();
        
        orderTimer.record(() -> {
            try {
                // Simulate work
                Thread.sleep(150);
                orderCounter.increment();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                activeOrders.decrementAndGet();
            }
        });
    }
}
```

### 3.4. The `@Timed` Annotation

For easier timer instrumentation on methods, you can use `@Timed`. Note that you MUST register a `TimedAspect` bean for this to work in Spring Boot.

```java
import io.micrometer.core.aop.TimedAspect;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

Usage:

```java
import io.micrometer.core.annotation.Timed;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PaymentController {

    @GetMapping("/api/payments")
    @Timed(value = "payments.fetch.time", description = "Time taken to fetch payments")
    public String getPayments() {
        return "Payments list";
    }
}
```

### 3.5. Exporting to Prometheus and Visualizing in Grafana

Prometheus is a popular pull-based monitoring system. To export metrics for Prometheus, add the dependency:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

In `application.yml`, enable the Prometheus endpoint:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

Your metrics will now be available at `/actuator/prometheus` in Prometheus scrape format. You configure Prometheus to scrape this URL every ~15 seconds. You then connect Grafana to Prometheus and build dashboards visualizing metrics like `http_server_requests_seconds_count` or `jvm_memory_used_bytes`.

---

## 4. Distributed Tracing

In a distributed system, a single user click might trigger calls to Service A, which calls Service B and C. If the click takes 5 seconds, which service is the bottleneck?

### 4.1. Trace ID vs Span ID

-   **Trace ID**: A unique identifier for the entire transaction (the entire journey across all microservices).
-   **Span ID**: A unique identifier for a specific unit of work (e.g., an HTTP call to one service, or a database query). A Trace contains a tree of Spans.

### 4.2. Micrometer Tracing (Spring Boot 3)

In Spring Boot 2, tracing was handled by Spring Cloud Sleuth. In Spring Boot 3, Sleuth has been retired and replaced by **Micrometer Tracing**.

Add the dependencies for Micrometer Tracing and an exporter (e.g., Zipkin):

```xml
<!-- Core Micrometer Tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId> <!-- Or bridge-otel for OpenTelemetry -->
</dependency>

<!-- Export to Zipkin -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

Configure `application.yml`:

```yaml
management:
  tracing:
    sampling:
      probability: 1.0 # 1.0 = Sample 100% of requests. In production, use 0.1 (10%)
  zipkin:
    tracing:
      endpoint: "http://localhost:9411/api/v2/spans"
```

Spring Boot auto-configures `RestTemplate`, `WebClient`, `@FeignClient`, and incoming MVC/WebFlux requests to automatically propagate Trace IDs via HTTP headers (e.g., `X-B3-TraceId` or W3C `traceparent`).

### 4.3. OpenTelemetry Integration

OpenTelemetry (OTel) is the modern CNCF standard. You can swap Brave for OTel:

```xml
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
management:
  otlp:
    tracing:
      endpoint: "http://localhost:4318/v1/traces"
```

### 4.4. MDC (Mapped Diagnostic Context) for Logging Correlation

Micrometer Tracing automatically injects the `traceId` and `spanId` into your logging framework's MDC (Mapped Diagnostic Context).

By updating your Logback pattern, you can output these IDs:

```xml
<!-- logback-spring.xml snippet -->
<pattern>%d{yyyy-MM-dd HH:mm:ss} - [%thread] %-5level %logger{36} - [traceId=%X{traceId}, spanId=%X{spanId}] - %msg%n</pattern>
```

Log output:
`2026-10-06 10:15:20 - [http-nio-8080-exec-1] INFO  com.app.OrderService - [traceId=658b1a2e3f4d, spanId=987c6b5a] - Processing order 123`

This allows you to copy a `traceId` from the logs and paste it into Zipkin/Jaeger to see the exact flow.

---

## 5. Structured Logging

### 5.1. Why JSON Logging?

Traditional log files (plain text) are meant for human eyes. However, log aggregators (Elasticsearch, Splunk, Datadog) parse logs better if they are structured as JSON.

Instead of parsing a regex against `[%thread] %-5level`, JSON logs are pre-parsed key-value pairs:
`{"timestamp":"...", "level":"INFO", "thread":"http-nio", "traceId":"...", "message":"..."}`

### 5.2. Logback Configuration for JSON (LogstashEncoder)

To generate JSON logs natively in Spring Boot, use `logstash-logback-encoder`:

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

Create a `logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="CONSOLE_JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
        </encoder>
    </appender>

    <!-- Use plain text for dev profile, JSON for prod -->
    <springProfile name="dev">
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE_JSON" />
        </root>
    </springProfile>
</configuration>
```

### 5.3. Dynamic Log Levels via Actuator

You can change log levels dynamically without a restart!
1. Check current loggers: `GET /actuator/loggers`
2. Change log level via HTTP POST:

```bash
curl -X POST http://localhost:8080/actuator/loggers/com.yourcompany.payments \
     -H "Content-Type: application/json" \
     -d '{"configuredLevel": "DEBUG"}'
```

This is invaluable for debugging a production issue in real-time.

### 5.4. ELK Stack Integration Overview

-   **Logstash / Fluentbit**: Collects the JSON logs emitted by your Spring Boot apps to stdout.
-   **Elasticsearch**: Stores and indexes the JSON logs.
-   **Kibana**: The UI where you search logs. You can filter `level: "ERROR"` and `traceId: "12345"`.

---

## 6. Exception & Error Tracking

### 6.1. Sentry Integration

While logs are great, exception tracking tools like Sentry automatically group identical exceptions and alert you when a new type of error spikes.

```xml
<dependency>
    <groupId>io.sentry</groupId>
    <artifactId>sentry-spring-boot-starter-jakarta</artifactId>
    <version>7.3.0</version>
</dependency>
```

Configure the DSN in `application.yml`:

```yaml
sentry:
  dsn: https://examplePublicKey@o0.ingest.sentry.io/0
  traces-sample-rate: 1.0
  exception-resolver-order: -2147483647
```
Any unhandled exception hitting your REST controllers will now automatically be reported to Sentry, including the stack trace, HTTP request parameters, and user context.

---

## ⚠️ 7. Interview Traps & Gotchas

> [!WARNING] The High Cardinality Trap
> Never use unbounded dynamic values (like user IDs, order IDs, or UUIDs) as tags/labels in Micrometer metrics.
> **BAD**: `registry.counter("login.attempts", "userId", "user-1234")`
> **GOOD**: `registry.counter("login.attempts", "status", "success")`
> *Why?* Every unique combination of tags creates a new Time Series object in memory and in Prometheus. High cardinality (millions of unique user IDs) will cause an OutOfMemoryError (OOM) in your application and crash your Prometheus server.

> [!WARNING] Exposing Sensitive Data in `/env`
> Actuator's `/env` endpoint will show property values. Spring Boot automatically masks keys containing words like `password`, `secret`, `key`, `token`. However, if you name your property `my.custom.db.auth-string`, Spring Boot won't mask it. You must configure `management.endpoint.env.keys-to-sanitize` to include your custom keys.

> [!WARNING] MDC Context Lost in @Async or Reactive Flows
> `MDC` uses `ThreadLocal` under the hood. If you spawn a new thread (e.g., using `@Async`, `CompletableFuture`, or `Project Reactor`), the `traceId` will be lost in the new thread!
> *Solution*: In Spring Boot 3 with Micrometer Tracing, use `ContextSnapshot` for manual propagation, or rely on Hooks for Reactor (`Hooks.enableAutomaticContextPropagation()`). For standard `@Async`, you must configure a `TaskDecorator` to copy MDC data from the calling thread to the executor thread.

> [!IMPORTANT] Trace ID Propagation over Message Queues
> Trace IDs don't propagate magically over Kafka or RabbitMQ. If using Spring AMQP or Spring Kafka, Spring Boot auto-configures the injection of Trace IDs into message headers. However, if you are using raw Apache Kafka clients or native Java clients, the Trace ID context will break unless you manually inject the context into the Kafka Headers before sending, and extract it on the consumer side.

---

## 🔍 8. Deep Dive: How Micrometer Tracing works internally

When an HTTP Request arrives:
1.  **Filter/Interceptor**: A filter (e.g., `ServerHttpObservationFilter`) intercepts the request.
2.  **Extraction**: It looks for standard HTTP headers (like `traceparent` or `X-B3-TraceId`).
    -   If found, it adopts the existing `TraceId` and creates a new child `SpanId`.
    -   If not found, it generates a new `TraceId` and root `SpanId`.
3.  **Scope**: It opens a "Scope" which places these IDs into a `ThreadLocal` storage (and synchronizes it with SLF4J MDC).
4.  **Execution**: Your Controller / Service code runs. All logs emitted will include the IDs from the MDC.
5.  **Injection**: If you make a downstream HTTP call via `RestTemplate`, an interceptor intercepts the outgoing call, reads the IDs from `ThreadLocal`, and injects them as HTTP headers on the outgoing request.
6.  **Completion**: When the request finishes, the filter closes the Span, recording its duration, and asynchronously exports it to Zipkin/Jaeger.

---

## 📝 9. Summary & Key Takeaways

1.  **Observability is non-negotiable** in distributed systems. You need Logs, Metrics, and Traces.
2.  **Actuator** exposes endpoints for health, metrics, and administration. It must be strictly secured.
3.  **Micrometer** abstracts metric collection. Beware of the high cardinality trap with Tags.
4.  **Micrometer Tracing** (formerly Sleuth) automatically propagates Trace and Span IDs across HTTP and Message brokers.
5.  **Structured JSON Logging** combined with MDC ensures every log line can be tied to a specific request trace in ELK.

---

## ❓ 10. Practice Questions

1.  What is the difference between a Trace ID and a Span ID?
2.  How would you prevent the `/heapdump` endpoint from being accessed over the internet, while allowing Kubernetes to access `/health/liveness`?
3.  You notice your JVM memory usage linearly increasing over days until the app crashes. You are using Micrometer to record API usage. What is the most likely cause?
4.  Why does logging the `traceId` stop working when you use `@Async` on a method, and how do you fix it?
5.  Explain the difference between a Liveness probe and a Readiness probe in Kubernetes.
