# 03 — REST API & Web Layer in Spring Boot
[🏠 Course Index](./00-course-index.md) | [← 02 — Annotations](./02-annotations.md) | **Module 03** | [Next: 04 — Exception Handling →](./04-exception-handling.md)

---

> **Series:** Spring Boot Mastery | **Level:** Intermediate → Advanced  
> **Spring Boot Version:** 3.x / Spring Framework 6.x | **Java Baseline:** Java 17/21+ (Jakarta EE 10)  
> **Focus:** Web execution pipeline, request lifecycle, multi-approach interception, modern HTTP clients, and high-throughput async APIs.

---

## Table of Contents

1. [REST Fundamentals & The Web Pipeline](#1-rest-fundamentals--the-web-pipeline)
   - [1.1 DispatcherServlet Architecture & `doDispatch()`](#11-dispatcherservlet-architecture--dodispatch)
   - [1.2 HandlerMapping Resolution Engine](#12-handlermapping-resolution-engine)
   - [1.3 End-to-End Request/Response Lifecycle](#13-end-to-end-requestresponse-lifecycle)
2. [Building RESTful Services](#2-building-restful-services)
   - [2.1 `@RestController` & Content Negotiation](#21-restcontroller--content-negotiation)
   - [2.2 HTTP Methods, Semantics & Idempotency](#22-http-methods-semantics--idempotency)
   - [2.3 Path Routing & Pattern Matching (`PathPatternParser`)](#23-path-routing--pattern-matching-pathpatternparser)
   - [2.4 Jakarta Validation on Request Payloads](#24-jakarta-validation-on-request-payloads)
   - [2.5 `ResponseEntity` Patterns & Headers](#25-responseentity-patterns--headers)
   - [2.6 API Versioning Strategies (4 Approaches)](#26-api-versioning-strategies-4-approaches)
3. [Request Interception Approaches (Filter vs Interceptor vs Aspect)](#3-request-interception-approaches-filter-vs-interceptor-vs-aspect)
   - [3.1 Approach 1: Servlet `OncePerRequestFilter` (Low-Level Boundary)](#31-approach-1-servlet-onceperrequestfilter-low-level-boundary)
   - [3.2 Approach 2: Spring MVC `HandlerInterceptor` (DispatcherServlet Boundary)](#32-approach-2-spring-mvc-handlerinterceptor-dispatcherservlet-boundary)
   - [3.3 Approach 3: Spring AOP Aspect (Method-Level Cross-Cutting)](#33-approach-3-spring-aop-aspect-method-level-cross-cutting)
   - [3.4 Interception Execution Pipeline & Comparison Matrix](#34-interception-execution-pipeline--comparison-matrix)
4. [Cross-Origin Resource Sharing (CORS) Configuration (3 Approaches)](#4-cross-origin-resource-sharing-cors-configuration-3-approaches)
   - [4.1 Approach 1: Controller-Level `@CrossOrigin`](#41-approach-1-controller-level-crossorigin)
   - [4.2 Approach 2: Global `WebMvcConfigurer.addCorsMappings()`](#42-approach-2-global-webmvcconfigureraddcorsmappings)
   - [4.3 Approach 3: Security-Level `CorsFilter` Bean (Pre-Flight Protected)](#43-approach-3-security-level-corsfilter-bean-pre-flight-protected)
   - [4.4 CORS Comparison Matrix & Production Gotchas](#44-cors-comparison-matrix--production-gotchas)
5. [Modern HTTP Clients in Spring Boot 3.x (4 Approaches)](#5-modern-http-clients-in-spring-boot-3x-4-approaches)
   - [5.1 Approach 1: Synchronous Fluent `RestClient` (Boot 3.2+ Standard)](#51-approach-1-synchronous-fluent-restclient-boot-32-standard)
   - [5.2 Approach 2: Declarative HTTP Interfaces (`@HttpExchange`)](#52-approach-2-declarative-http-interfaces-httpexchange)
   - [5.3 Approach 3: Reactive `WebClient` (Project Reactor)](#53-approach-3-reactive-webclient-project-reactor)
   - [5.4 Approach 4: Legacy `RestTemplate` (Maintenance Mode)](#54-approach-4-legacy-resttemplate-maintenance-mode)
   - [5.5 HTTP Client Comparison Matrix](#55-http-client-comparison-matrix)
6. [Asynchronous REST Processing (4 Approaches)](#6-asynchronous-rest-processing-4-approaches)
   - [6.1 Approach 1: `CompletableFuture<ResponseEntity<T>>`](#61-approach-1-completablefutureresponseentityt)
   - [6.2 Approach 2: `DeferredResult<ResponseEntity<T>>` (Long Polling & Callbacks)](#62-approach-2-deferredresultresponseentityt-long-polling--callbacks)
   - [6.3 Approach 3: Server-Sent Events (SSE) via `SseEmitter`](#63-approach-3-server-sent-events-sse-via-sseemitter)
   - [6.4 Approach 4: Virtual Threads (Java 21 Project Loom)](#64-approach-4-virtual-threads-java-21-project-loom)
   - [6.5 Async Processing Comparison Matrix](#65-async-processing-comparison-matrix)
7. [Enterprise DTOs & High-Performance Mapping](#7-enterprise-dtos--high-performance-mapping)
8. [Spring Boot Actuator & Observability Metrics](#8-spring-boot-actuator--observability-metrics)
9. [⚠️ High-Frequency Interview Traps & Web Gotchas](#9-️-high-frequency-interview-traps--web-gotchas)
10. [🔍 Deep Dive — `HttpMessageConverter` Internals](#10--deep-dive--httpmessageconverter-internals)
11. [📝 Architecture Principle Checklist](#11--architecture-principle-checklist)
12. [❓ Practice Questions & Answers](#12--practice-questions--answers)

---

## 1. REST Fundamentals & The Web Pipeline

### 1.1 DispatcherServlet Architecture & `doDispatch()`

Spring MVC is fundamentally designed around the **Front Controller pattern**. In a Spring Boot application, a single master servlet — `DispatcherServlet` — receives all inbound HTTP requests and orchestrates their routing, parameter resolution, interceptor invocation, and serialization.

```
Client HTTP Request
         │
         ▼
 [Embedded Tomcat / Jetty / Undertow]
         │
         ▼
 [Servlet Filter Chain (Security, Logging, CORS)]
         │
         ▼
 DispatcherServlet.doDispatch()
         ├── 1. Multipart Check: Wraps multipart requests
         ├── 2. Handler Mapping: Locates HandlerExecutionChain (Controller + Interceptors)
         ├── 3. Handler Adapter: RequestMappingHandlerAdapter chosen
         ├── 4. applyPreHandle: Runs interceptor preHandle() methods
         ├── 5. Invocation: Controller method executes via HandlerMethod
         │       └── Argument Resolvers -> Validation -> Method Body -> Return Value Handlers
         ├── 6. applyPostHandle: Runs interceptor postHandle() in reverse order
         ├── 7. processDispatchResult: Handles exceptions or renders views
         └── 8. triggerAfterCompletion: Cleans up thread-locals and streams
```

---

### 1.2 HandlerMapping Resolution Engine

During bootstrap, `RequestMappingHandlerMapping` scans all `@Controller` and `@RestController` beans, parses their `@RequestMapping` annotations, and builds a routing lookup table.

Priority Order of Handler Mappings:
1. `RequestMappingHandlerMapping` (Order `0`): `@RequestMapping` controller methods.
2. `SimpleUrlHandlerMapping`: Maps static resources (`/static/**`, `/public/**`).
3. `WelcomePageHandlerMapping`: Resolves `index.html`.

Since Spring Boot 2.6 and Spring 6, **`PathPatternParser`** is the default matching strategy instead of legacy `AntPathMatcher`. `PathPatternParser` pre-parses URI templates into tokenized trees, eliminating regex evaluation latency.

---

### 1.3 End-to-End Request/Response Lifecycle

```
1. Client establishes TCP connection -> TLS Handshake completes.
2. Embedded Tomcat thread accepts HTTP byte buffer -> Parses HTTP/1.1 or HTTP/2 headers.
3. Servlet Filter Chain executes (e.g. SecurityContextPersistenceFilter, JwtAuthFilter).
4. DispatcherServlet matches request to UserController.getUser(Long).
5. HandlerInterceptor.preHandle() executes.
6. RequestResponseBodyMethodProcessor invokes Jackson ObjectMapper to deserialize JSON body.
7. Jakarta Bean Validation validates constraints (@NotNull, @Min).
8. Business Controller method executes and queries database.
9. Return object serialized into JSON via MappingJackson2HttpMessageConverter.
10. Response committed to TCP socket; HandlerInterceptor.afterCompletion() executes.
```

---

## 2. Building RESTful Services

### 2.1 `@RestController` & Content Negotiation

`@RestController` is a composed annotation:
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Controller
@ResponseBody
public @interface RestController {}
```
Because `@ResponseBody` is declared at the type level, return values are directly serialized into the HTTP response body using an appropriate `HttpMessageConverter` (defaulting to Jackson for JSON), bypassing MVC view rendering.

---

### 2.2 HTTP Methods, Semantics & Idempotency

| HTTP Method | Safe? (Read-Only) | Idempotent? | Typical Enterprise Use | Status on Success |
|---|---|---|---|---|
| **GET** | ✅ Yes | ✅ Yes | Fetch entity or query list | `200 OK` |
| **POST** | ❌ No | ❌ No | Create new resource or trigger complex command | `201 Created` / `200 OK` |
| **PUT** | ❌ No | ✅ Yes | Complete replacement of resource | `200 OK` / `204 No Content` |
| **PATCH** | ❌ No | ⚠️ Conditional | Partial modification of specific fields | `200 OK` / `204 No Content` |
| **DELETE** | ❌ No | ✅ Yes | Delete resource by identifier | `204 No Content` / `200 OK` |
| **HEAD** | ✅ Yes | ✅ Yes | Retrieve headers without response body | `200 OK` |
| **OPTIONS** | ✅ Yes | ✅ Yes | CORS Preflight discovery of allowed methods | `200 OK` / `204 No Content` |

---

### 2.3 Path Routing & Pattern Matching (`PathPatternParser`)

```java
@RestController
@RequestMapping("/api/v1/organizations/{orgId:[0-9]+}") // Regex restriction: orgId must be numeric
public class OrganizationController {

    // Matches /api/v1/organizations/42/projects/alpha
    @GetMapping("/projects/{projectName:[a-z-]+}")
    public ResponseEntity<ProjectDto> getProject(
            @PathVariable Long orgId,
            @PathVariable String projectName) {
        return ResponseEntity.ok(projectService.get(orgId, projectName));
    }
}
```

---

### 2.4 Jakarta Validation on Request Payloads

```java
public record CreateOrderRequest(
    @NotNull(message = "customerId is mandatory")
    Long customerId,

    @NotEmpty(message = "Order must contain at least one line item")
    List<@Valid OrderItemRequest> items,

    @DecimalMin(value = "0.50", message = "Order total must be at least $0.50")
    BigDecimal totalAmount
) {}

public record OrderItemRequest(
    @NotBlank(message = "sku is required")
    String sku,

    @Min(value = 1, message = "Quantity must be at least 1")
    int quantity
) {}
```

Controller validation binding:
```java
@PostMapping
public ResponseEntity<OrderResponse> placeOrder(@Valid @RequestBody CreateOrderRequest request) {
    // If validation fails, MethodArgumentNotValidException is thrown before this method runs
    return ResponseEntity.status(HttpStatus.CREATED).body(orderService.create(request));
}
```

---

### 2.5 `ResponseEntity` Patterns & Headers

```java
@PostMapping("/documents")
public ResponseEntity<DocumentDto> uploadDocument(@RequestBody DocumentCreateDto dto) {
    DocumentDto created = documentService.save(dto);

    // Standard RFC-compliant REST: Returns 201 Created with Location header
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.id())
            .toUri();

    return ResponseEntity.created(location)
            .header("X-Resource-Version", "1.0")
            .body(created);
}
```

---

### 2.6 API Versioning Strategies (4 Approaches)

When updating REST API contracts without breaking existing mobile clients or partner integrations, four distinct versioning approaches exist.

---

#### Approach 1: URI Path Versioning (Most Common & Pragmatic)

##### Scenario
- Public APIs, mobile backends, and microservices where versioning must be immediately obvious and easily routed by external API gateways (Kong, Apigee, AWS API Gateway).

##### Use-Case
Upgrading customer endpoints from version 1 to version 2 with backward compatibility for older mobile apps.

##### Benefits Over Others
- **Zero Ambiguity**: The target API version is explicit in every log line, cURL command, and browser URL.
- **Trivial Gateway Routing**: CDN edges and reverse proxies can route `/v1/` to legacy services and `/v2/` to modern services.
- **Swagger / OpenAPI Friendly**: Generates distinct OpenAPI specs per version effortlessly.

##### Documented Code Example

```java
package com.enterprise.api.v1;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/customers")
public class CustomerApiControllerV1 {

    @GetMapping("/{id}")
    public ResponseEntity<CustomerV1Dto> getCustomer(@PathVariable Long id) {
        return ResponseEntity.ok(new CustomerV1Dto(id, "John Doe", "john@example.com"));
    }
}
```

```java
package com.enterprise.api.v2;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v2/customers")
public class CustomerApiControllerV2 {

    @GetMapping("/{id}")
    public ResponseEntity<CustomerV2Dto> getCustomer(@PathVariable Long id) {
        // V2 introduces separate firstName, lastName, and status enum
        return ResponseEntity.ok(new CustomerV2Dto(id, "John", "Doe", "john@example.com", "ACTIVE"));
    }
}
```

---

#### Approach 2: Query Parameter Versioning

##### Scenario
- Internal microservices or single-endpoint search queries where clients default to a baseline version unless an explicit query parameter is appended.

##### Use-Case
An analytics endpoint: `GET /api/reports?version=1` vs `GET /api/reports?version=2`.

##### Benefits Over Others
- Preserves clean resource URIs (`/api/reports`).
- Default parameters can be declared (`defaultValue = "1"`).

##### Documented Code Example

```java
package com.enterprise.api.param;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/reports")
public class ReportVersionController {

    // Dispatches when ?version=1
    @GetMapping(params = "version=1")
    public ResponseEntity<String> getReportV1() {
        return ResponseEntity.ok("Legacy PDF Report format v1");
    }

    // Dispatches when ?version=2
    @GetMapping(params = "version=2")
    public ResponseEntity<String> getReportV2() {
        return ResponseEntity.ok("Modern Real-time JSON Analytics Report v2");
    }
}
```

---

#### Approach 3: Custom Request Header Versioning

##### Scenario
- Enterprise B2B SaaS platforms where URL stability is required across enterprise clients, but versioning is negotiated via client HTTP headers.

##### Use-Case
A banking API requiring `X-API-VERSION: 1` or `X-API-VERSION: 2`.

##### Benefits Over Others
- Clean URLs without query pollution.
- Allows global interceptors or security filters to reject deprecated versions before controller dispatch.

##### Documented Code Example

```java
package com.enterprise.api.headers;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/payments")
public class PaymentHeaderVersioningController {

    @GetMapping(value = "/{id}", headers = "X-API-VERSION=1")
    public ResponseEntity<String> getPaymentV1(@PathVariable Long id) {
        return ResponseEntity.ok("Legacy Payment V1: " + id);
    }

    @GetMapping(value = "/{id}", headers = "X-API-VERSION=2")
    public ResponseEntity<String> getPaymentV2(@PathVariable Long id) {
        return ResponseEntity.ok("Modern Multi-Currency Payment V2: " + id);
    }
}
```

---

#### Approach 4: Content Negotiation / Accept Header Versioning (Strict REST Standard)

##### Scenario
- True REST purists where a URI represents the **resource itself**, and the `Accept` header requests a specific **representation/media type** of that resource.

##### Use-Case
Requesting a customer resource representation:
`Accept: application/vnd.enterprise.customer.v1+json` vs `Accept: application/vnd.enterprise.customer.v2+json`.

##### Benefits Over Others
- Adheres strictly to Roy Fielding's REST architectural style and HTTP specifications.
- Enables rich content negotiation (JSON, XML, Protobuf versions under a single URI).

##### Documented Code Example

```java
package com.enterprise.api.content;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/accounts")
public class AccountMediaVersioningController {

    @GetMapping(value = "/{id}", produces = "application/vnd.enterprise.account.v1+json")
    public ResponseEntity<String> getAccountRepresentationV1(@PathVariable Long id) {
        return ResponseEntity.ok("{\"version\": 1, \"account\": " + id + "}");
    }

    @GetMapping(value = "/{id}", produces = "application/vnd.enterprise.account.v2+json")
    public ResponseEntity<String> getAccountRepresentationV2(@PathVariable Long id) {
        return ResponseEntity.ok("{\"version\": 2, \"accountId\": " + id + ", \"status\": \"VERIFIED\"}");
    }
}
```

---

### API Versioning Comparison Matrix

| Approach | URL Cleanliness | Caching Friendliness | Gateway Routing Simplicity | Developer Ergonomics | Production Standard |
|---|---|---|---|---|---|
| **URI Path (`/v1/`)** | ⚠️ Modified URL | 🌟 **Maximum** | 🌟 **Trivial** | 🌟 **Highest** | 🌟 **Recommended for 90% of APIs** |
| **Query Param (`?v=1`)**| ⚠️ Polluted Query | ⚠️ Requires query caching | ⚠️ Moderate | Moderate | Used for optional feature flags |
| **Custom Header (`X-API`)**| ✅ Completely Clean | ❌ Cache must inspect Vary | ⚠️ Requires header parsing | ⚠️ Difficult in browser URL | B2B Private APIs |
| **Accept Header** | ✅ Completely Clean | ❌ Cache must inspect Vary | ❌ Complex gateway rules | ⚠️ Complex cURL / Postman | High-end Enterprise Media Types |

---

## 3. Request Interception Approaches (Filter vs Interceptor vs Aspect)

A critical architectural decision in Spring Boot is selecting the right interception mechanism for cross-cutting web concerns.

```
Incoming Request
      │
      ▼
┌────────────────────────────────────────┐
│ 1. Servlet Filter Chain                │  <-- Low-level Servlet Container (Tomcat)
│    (OncePerRequestFilter)              │      Access: Raw HttpServletRequest / Response
└──────────────────┬─────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────┐
│ 2. DispatcherServlet                   │
└──────────────────┬─────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────┐
│ 3. HandlerInterceptor (preHandle)      │  <-- Spring MVC Layer
│                                        │      Access: HandlerMethod, Controller target
└──────────────────┬─────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────┐
│ 4. Spring AOP (@Around Aspect)         │  <-- Spring Proxy Layer
│                                        │      Access: Method arguments, parameter types
└──────────────────┬─────────────────────┘
                   │
                   ▼
       [Controller Method Execution]
```

---

### 3.1 Approach 1: Servlet `OncePerRequestFilter` (Low-Level Boundary)

#### Scenario
- Low-level request modifications before Spring MVC is engaged: Request/response stream wrapping, correlation ID injection into SLF4J MDC, security token extraction, compression, or request body caching.

#### Use-Case
A request correlation ID filter that generates an `X-Correlation-ID` header if absent, adds it to the logging MDC, and measures complete network turnaround time.

#### Benefits Over Others
- **Guaranteed Single Execution**: `OncePerRequestFilter` ensures the filter executes exactly once per request, even during asynchronous dispatches or internal forwardings.
- **Stream Wrapping Capability**: The only layer where you can wrap `HttpServletRequest` using `ContentCachingRequestWrapper` to read the request body multiple times.

#### Well-Documented Code Example

```java
package com.enterprise.web.filter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE) // Executes first in the filter chain
public class CorrelationIdMdcFilter extends OncePerRequestFilter {

    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    public static final String MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        // Put into logging MDC so all log statements automatically include [correlationId=...]
        MDC.put(MDC_KEY, correlationId);
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        long startTime = System.currentTimeMillis();
        try {
            // Hand off to the next filter in the chain (or DispatcherServlet)
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            System.out.printf("[%s] %s %s completed in %d ms%n",
                    correlationId, request.getMethod(), request.getRequestURI(), duration);
            // CRITICAL: Clean up MDC to prevent thread pool memory leakage
            MDC.remove(MDC_KEY);
        }
    }
}
```

---

### 3.2 Approach 2: Spring MVC `HandlerInterceptor` (DispatcherServlet Boundary)

#### Scenario
- Cross-cutting concerns that require knowledge of the target controller method (`HandlerMethod`), auditing controller annotations, or modifying the model before view rendering.

#### Use-Case
An API rate-limiting or maintenance-mode interceptor that inspects whether the target controller method is annotated with `@UnderMaintenance`.

#### Benefits Over Others
- Direct access to `Object handler` castable to `HandlerMethod`, allowing runtime reflection on controller methods and method annotations.
- Offers three distinct lifecycle hooks: `preHandle()` (can abort execution), `postHandle()` (before view rendering), and `afterCompletion()` (runs even if exceptions occur).

#### Well-Documented Code Example

```java
package com.enterprise.web.interceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface RequiresSubscription {
    String tier() default "PREMIUM";
}

@Component
public class SubscriptionPlanInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // Only inspect if target handler is a Spring Controller Method
        if (handler instanceof HandlerMethod handlerMethod) {
            RequiresSubscription annotation = handlerMethod.getMethodAnnotation(RequiresSubscription.class);
            if (annotation == null) {
                annotation = handlerMethod.getBeanType().getAnnotation(RequiresSubscription.class);
            }

            if (annotation != null) {
                String requiredTier = annotation.tier();
                String userTier = request.getHeader("X-User-Tier");

                if (!requiredTier.equalsIgnoreCase(userTier)) {
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    response.getWriter().write("Subscription tier '" + requiredTier + "' required.");
                    return false; // ABORT request processing: do NOT invoke controller method!
                }
            }
        }
        return true; // PROCEED with request execution
    }
}
```

Registration in `WebMvcConfigurer`:
```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    private final SubscriptionPlanInterceptor subscriptionInterceptor;

    public WebMvcConfig(SubscriptionPlanInterceptor subscriptionInterceptor) {
        this.subscriptionInterceptor = subscriptionInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(subscriptionInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

---

### 3.3 Approach 3: Spring AOP Aspect (Method-Level Cross-Cutting)

#### Scenario
- Pure method-level concerns: capturing method execution metrics, method parameter auditing, or declarative security annotations without coupling to HTTP abstractions (`HttpServletRequest`).

#### Use-Case
Recording method execution latency into Micrometer metrics for specific high-value controller operations.

#### Benefits Over Others
- Works identically on any Spring bean (Controllers, Services, Repositories).
- Direct strongly-typed access to method arguments via `joinPoint.getArgs()`.

#### Well-Documented Code Example

```java
package com.enterprise.web.aspect;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class ControllerPerformanceAspect {

    private final MeterRegistry meterRegistry;

    public ControllerPerformanceAspect(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Around("within(@org.springframework.web.bind.annotation.RestController *)")
    public Object profileControllerMethods(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().toShortString();
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            return joinPoint.proceed(); // Invokes actual controller method
        } finally {
            sample.stop(meterRegistry.timer("api.controller.execution.time", "method", methodName));
        }
    }
}
```

---

### 3.4 Interception Execution Pipeline & Comparison Matrix

| Feature | Servlet Filter (`OncePerRequestFilter`) | Spring MVC `HandlerInterceptor` | Spring AOP Aspect |
|---|---|---|---|
| **Architectural Layer** | Servlet Container (Pre-DispatcherServlet) | Spring MVC (Post-DispatcherServlet) | Spring Bean Proxy Layer |
| **Aware of Spring MVC `HandlerMethod`?** | ❌ No | 🌟 **Yes (`HandlerMethod`)** | ⚠️ Via reflection joinpoint |
| **Can Modify Request/Response Streams?** | 🌟 **Yes (Stream Wrappers)** | ❌ Difficult (streams already open) | ❌ No |
| **Can Halt Request Execution?** | ✅ Yes (don't call `filterChain.doFilter`)| ✅ Yes (return `false` from `preHandle`) | ✅ Yes (throw exception) |
| **Exception Handling Integration** | Runs outside `@ControllerAdvice` | Runs inside MVC pipeline | Runs inside MVC pipeline |
| **Primary Use-Cases** | CORS, Security, GZIP, Correlation ID MDC | Auth validation, Rate limits, Locale | Business metrics, Parameter auditing |

---

## 4. Cross-Origin Resource Sharing (CORS) Configuration (3 Approaches)

---

### 4.1 Approach 1: Controller-Level `@CrossOrigin`
- **Scenario**: Fine-grained, per-controller or per-method cross-origin rules for specific partner integrations.

```java
@RestController
@RequestMapping("/api/v1/public-widgets")
@CrossOrigin(
    origins = { "https://partner.com", "https://analytics.io" },
    methods = { RequestMethod.GET, RequestMethod.POST },
    allowedHeaders = { "Content-Type", "Authorization" },
    maxAge = 3600
)
public class PublicWidgetController { ... }
```

---

### 4.2 Approach 2: Global `WebMvcConfigurer.addCorsMappings()`
- **Scenario**: Standard web applications without complex Spring Security filters.

```java
@Configuration
public class GlobalCorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://app.company.com", "http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .exposedHeaders("X-Correlation-ID")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

---

### 4.3 Approach 3: Security-Level `CorsFilter` Bean (Pre-Flight Protected)
- **Scenario**: Applications using **Spring Security**.
- **Critical Architectural Insight**: Spring Security filters execute *before* Spring MVC `HandlerMapping`. If CORS is configured only via `WebMvcConfigurer`, preflight `OPTIONS` requests will hit Spring Security first, get blocked with `401 Unauthorized` or `403 Forbidden`, and fail before ever reaching the MVC layer!

```java
package com.enterprise.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebSecurity
public class EnterpriseSecurityCorsConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 1. Integrates with CorsConfigurationSource bean below
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://dashboard.enterprise.com", "http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Correlation-ID"));
        config.setExposedHeaders(List.of("X-Correlation-ID"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

### 4.4 CORS Comparison Matrix & Production Gotchas

| Strategy | Security Filter Precedence | Maintenance Overhead | Best For |
|---|---|---|---|
| **`@CrossOrigin`** | ❌ Fails if Spring Security is present | High (scattered across controllers) | Temporary developer testing |
| **`WebMvcConfigurer`** | ❌ Fails pre-flight in Spring Security | Low (centralized MVC rules) | Plain Spring Boot apps without Spring Security |
| **`CorsConfigurationSource`**| 🌟 **Executes before Security Filters** | Low (single enterprise source of truth) | 🌟 **Mandatory for all apps with Spring Security** |

> [!WARNING]
> **The Wildcard Credential Trap**: Browsers explicitly reject responses where `Access-Control-Allow-Credentials: true` is combined with wildcard `Access-Control-Allow-Origin: *`. You must specify explicit origins (or use `allowedOriginPatterns`).

---

## 5. Modern HTTP Clients in Spring Boot 3.x (4 Approaches)

---

### 5.1 Approach 1: Synchronous Fluent `RestClient` (Boot 3.2+ Standard)

#### Scenario
- Modern synchronous REST client interactions in Spring Boot 3.2+. Replaces `RestTemplate` with a fluent, chainable API similar to `WebClient` but without reactive dependencies.

```java
@Configuration
public class ClientConfiguration {

    @Bean
    public RestClient paymentRestClient(RestClient.Builder builder) {
        return builder
                .baseUrl("https://api.stripe.com/v1")
                .defaultHeader("Authorization", "Bearer sk_test_secret")
                .requestFactory(new JdkClientHttpRequestFactory()) // Native HTTP/2 client
                .build();
    }
}

@Service
public class StripePaymentClient {
    private final RestClient restClient;

    public StripePaymentClient(RestClient restClient) {
        this.restClient = restClient;
    }

    public ChargeResponse createCharge(ChargeRequest charge) {
        return restClient.post()
                .uri("/charges")
                .contentType(MediaType.APPLICATION_JSON)
                .body(charge)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, (req, resp) -> {
                    throw new PaymentApiException("Client error: " + resp.getStatusCode());
                })
                .body(ChargeResponse.class);
    }
}
```

---

### 5.2 Approach 2: Declarative HTTP Interfaces (`@HttpExchange`)

#### Scenario
- Microservice inter-communication where you define an interface with annotations, and Spring generates the client proxy dynamically (similar to Spring Cloud OpenFeign, but built directly into Spring Framework 6).

```java
package com.enterprise.client;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

@HttpExchange("/api/v1/users")
public interface UserClient {

    @GetExchange("/{id}")
    UserDto getUserById(@PathVariable("id") Long id);

    @PostExchange
    UserDto createUser(UserCreateDto dto);
}
```

Adapter configuration:
```java
@Configuration
public class HttpInterfaceConfig {

    @Bean
    public UserClient userClient(RestClient.Builder builder) {
        RestClient restClient = builder.baseUrl("https://user-service.internal").build();
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        return factory.createClient(UserClient.class);
    }
}
```

---

### 5.3 Approach 3: Reactive `WebClient` (Project Reactor)

#### Scenario
- High-concurrency non-blocking client calls, event streaming, or reactive Spring WebFlux architectures.

```java
@Service
public class ReactiveWeatherClient {
    private final WebClient webClient;

    public ReactiveWeatherClient(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("https://api.weather.com").build();
    }

    public Mono<WeatherDto> getWeatherAsync(String city) {
        return webClient.get()
                .uri("/current?city={city}", city)
                .retrieve()
                .bodyToMono(WeatherDto.class);
    }
}
```

---

### 5.4 Approach 4: Legacy `RestTemplate` (Maintenance Mode)

#### Scenario
- Existing codebases prior to Spring Boot 3.2. As of Spring Framework 6, `RestTemplate` is in maintenance mode.

```java
@Service
public class LegacyClient {
    private final RestTemplate restTemplate;

    public LegacyClient(RestTemplateBuilder builder) {
        this.restTemplate = builder.build();
    }

    public UserDto fetchUser(Long id) {
        return restTemplate.getForObject("https://api.com/users/{id}", UserDto.class, id);
    }
}
```

---

### 5.5 HTTP Client Comparison Matrix

| Client | Paradigm | Spring Version | Fluent API? | Declarative Interface? | Best For |
|---|---|---|---|---|---|
| **`RestClient`** | Synchronous | 🌟 **Boot 3.2+ / Spring 6.1** | 🌟 **Yes** | Integrated with HTTP Interfaces | 🌟 **Default standard for Boot 3.x** |
| **`@HttpExchange`**| Declarative | 🌟 **Boot 3.0+ / Spring 6.0** | N/A | 🌟 **Yes (Native Feign replacement)** | Microservice-to-microservice RPC |
| **`WebClient`** | Reactive | Boot 2.0+ / Spring 5.0 | 🌟 **Yes** | Integrated with HTTP Interfaces | High-scale streaming & WebFlux |
| **`RestTemplate`**| Synchronous | Legacy (Spring 3.0+) | ❌ No | ❌ No | Maintenance legacy codebases |

---

## 6. Asynchronous REST Processing (4 Approaches)

Tomcat operates by default with a thread pool of 200 threads. When long-running external operations occur, threads become blocked. Asynchronous REST releases the container thread back to the pool immediately.

---

### 6.1 Approach 1: `CompletableFuture<ResponseEntity<T>>`
- **Scenario**: Offloading computationally expensive tasks or fan-out microservice aggregation.

```java
@GetMapping("/aggregate/{customerId}")
public CompletableFuture<ResponseEntity<CustomerDashboardDto>> getDashboard(@PathVariable Long customerId) {
    return dashboardService.loadDashboardAsync(customerId)
            .thenApply(ResponseEntity::ok)
            .exceptionally(ex -> ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build());
}
```

---

### 6.2 Approach 2: `DeferredResult<ResponseEntity<T>>` (Long Polling & Callbacks)
- **Scenario**: Asynchronous callbacks, third-party payment webhooks, or long polling where the result is completed by an external messaging event.

```java
@RestController
@RequestMapping("/api/orders")
public class AsyncOrderCallbackController {

    private final Map<String, DeferredResult<ResponseEntity<String>>> pendingOrders = new ConcurrentHashMap<>();

    @GetMapping("/await-fulfillment/{orderId}")
    public DeferredResult<ResponseEntity<String>> waitForFulfillment(@PathVariable String orderId) {
        DeferredResult<ResponseEntity<String>> result = new DeferredResult<>(30000L, 
                ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT).body("Timeout waiting for fulfillment"));

        pendingOrders.put(orderId, result);
        result.onCompletion(() -> pendingOrders.remove(orderId));
        return result; // Tomcat container thread is IMMEDIATELY released!
    }

    // Invoked by RabbitMQ/Kafka consumer on fulfillment
    public void onOrderFulfilledEvent(String orderId) {
        DeferredResult<ResponseEntity<String>> result = pendingOrders.get(orderId);
        if (result != null) {
            result.setResult(ResponseEntity.ok("Order Fulfilled successfully"));
        }
    }
}
```

---

### 6.3 Approach 3: Server-Sent Events (SSE) via `SseEmitter`
- **Scenario**: Unidirectional real-time updates from server to browser over standard HTTP (e.g. stock tickers, LLM streaming output).

```java
@GetMapping(value = "/stream/ai-chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamAiChat(@RequestParam String prompt) {
    SseEmitter emitter = new SseEmitter(60000L);

    Executors.newSingleThreadExecutor().submit(() -> {
        try {
            for (int i = 1; i <= 5; i++) {
                emitter.send(SseEmitter.event().data("Chunk #" + i + " response"));
                Thread.sleep(500);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });

    return emitter;
}
```

---

### 6.4 Approach 4: Virtual Threads (Java 21 Project Loom)
- **Scenario**: Modern high-throughput backend services in Spring Boot 3.2+ running on Java 21.
- **Why it changes everything**: Eliminates the need for complex reactive or `DeferredResult` code. You write standard imperative blocking code, but the JVM unmounts the virtual thread during blocking socket I/O, allowing millions of concurrent requests!

```properties
# application.properties
spring.threads.virtual.enabled=true
```

```java
// With Virtual Threads enabled, this imperative blocking method handles 10,000+ concurrent requests effortlessly!
@GetMapping("/quote/{id}")
public ResponseEntity<QuoteDto> fetchQuote(@PathVariable Long id) {
    // Normal blocking call: carrier thread unmounts automatically!
    QuoteDto quote = externalApiClient.getQuoteBlocking(id);
    return ResponseEntity.ok(quote);
}
```

---

### 6.5 Async Processing Comparison Matrix

| Approach | Thread Unblocking | Streaming Support | Programming Model | Best For |
|---|---|---|---|---|
| **`CompletableFuture`** | ✅ Releases Tomcat thread | ❌ Single response | Async / Functional | Microservice aggregation |
| **`DeferredResult`** | ✅ Releases Tomcat thread | ❌ Single response | Event Callback | Long polling, external webhooks |
| **`SseEmitter`** | ✅ Releases Tomcat thread | 🌟 **Yes (Continuous SSE)** | Event streaming | Real-time browser dashboards, LLM stream |
| **Virtual Threads (Java 21)**| 🌟 **JVM-level unmount** | ❌ Standard REST | 🌟 **Imperative / Clean** | 🌟 **Modern standard for high scale** |

---

## 7. Enterprise DTOs & High-Performance Mapping

Using entity classes directly in REST controllers causes security vulnerabilities (over-posting), tight database coupling, and Jackson serialization loops. Use Java records for immutable DTOs and MapStruct for compile-time mapping.

```java
// Compile-time generated mapper: zero reflection overhead!
@Mapper(componentModel = "spring")
public interface CustomerMapper {
    CustomerResponse toResponse(Customer customer);
    Customer toEntity(CreateCustomerRequest request);
}
```

---

## 8. Spring Boot Actuator & Observability Metrics

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true # Kubernetes liveness and readiness probes
```

Custom Health Indicator:
```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean reachable = pingGateway();
        if (reachable) {
            return Health.up().withDetail("gateway", "Available").build();
        }
        return Health.down().withDetail("gateway", "Unreachable").build();
    }
}
```

---

## 9. ⚠️ High-Frequency Interview Traps & Web Gotchas

### Trap 1: Pre-Flight CORS OPTIONS Failing in Spring Security
- **Problem**: CORS is defined in `WebMvcConfigurer`, but Spring Security filters intercept and block `OPTIONS` with `401 Unauthorized` before MVC evaluates.
- **Solution**: Always configure CORS via `http.cors(cors -> cors.configurationSource(...))` in Spring Security.

### Trap 2: Reading the Request Body Multiple Times
- **Problem**: `HttpServletRequest.getInputStream()` can only be read **once**. Attempting to read it in both a Filter and `@RequestBody` results in `IOException: Stream closed`.
- **Solution**: Wrap the request in `ContentCachingRequestWrapper`.

### Trap 3: PUT vs PATCH Idempotency
- **Problem**: Assuming PATCH is always non-idempotent.
- **Fact**: A PATCH that assigns an exact value (`{"name": "Alice"}`) is idempotent; an incremental operation (`{"increment": 1}`) is not.

### Trap 4: HandlerInterceptor `postHandle()` Not Called on Exceptions
- **Problem**: Relying on `postHandle()` to clean up resources. If the controller throws an exception, `postHandle()` is skipped!
- **Solution**: Always perform cleanup in `afterCompletion(request, response, handler, exception)`.

---

## 10. 🔍 Deep Dive — `HttpMessageConverter` Internals

When a controller returns an object, `RequestResponseBodyMethodProcessor` iterates over all registered `HttpMessageConverter`s in order:
1. `ByteArrayHttpMessageConverter`
2. `StringHttpMessageConverter`
3. `ResourceHttpMessageConverter`
4. `MappingJackson2HttpMessageConverter` (Handles JSON serialization)

The converter checks `canWrite(clazz, mediaType)`. Jackson's converter inspects the `Accept` header and matches `application/json`, streaming the JSON bytes directly into the servlet `OutputStream`.

---

## 11. 📝 Architecture Principle Checklist

- [ ] Use **constructor injection** in all `@RestController` classes.
- [ ] Represent request and response payloads as **Java records** (immutable DTOs).
- [ ] Configure CORS at the **Spring Security level** (`CorsConfigurationSource`) when security is active.
- [ ] Choose **URI Path Versioning (`/v1/`)** for pragmatic, gateway-friendly contracts.
- [ ] Use **`RestClient` or `@HttpExchange` interfaces** in Spring Boot 3.2+ for outbound HTTP calls.
- [ ] Enable **Virtual Threads** (`spring.threads.virtual.enabled=true`) on Java 21+ for effortless high concurrency.

---

## 12. ❓ Practice Questions & Answers

### Q1: What is the exact difference between a Servlet Filter and a Spring MVC HandlerInterceptor?
**Answer**: A Servlet Filter belongs to the Servlet Container (Tomcat) and executes before the request enters `DispatcherServlet`. It operates on raw `HttpServletRequest` and `HttpServletResponse` streams and can wrap streams (e.g. for caching bodies or GZIP compression). A `HandlerInterceptor` lives inside Spring MVC and executes after `DispatcherServlet` has resolved the handler. It has direct access to the target `HandlerMethod` and model, but cannot wrap underlying servlet streams.

### Q2: Why should you prefer `RestClient` over `RestTemplate` in modern Spring Boot 3.2+?
**Answer**: `RestTemplate` is in maintenance mode as of Spring Framework 6. `RestClient` offers a modern, fluent API with non-blocking chaining style while operating synchronously on standard thread pools (or virtual threads). It cleanly integrates with `@HttpExchange` interfaces, has better status code handling (`onStatus`), and native HTTP/2 client support.

### Q3: Why does preflight CORS fail when configured only in `WebMvcConfigurer` with Spring Security enabled?
**Answer**: The browser sends an unauthenticated HTTP `OPTIONS` preflight request before the actual request. The Spring Security filter chain runs before `DispatcherServlet`. Because the request lacks credentials, Spring Security rejects it with a `401` or `403` error before it ever reaches the MVC layer where `WebMvcConfigurer` CORS rules are evaluated.

---

[🏠 Course Index](./00-course-index.md) | [← 02 — Annotations](./02-annotations.md) | **Module 03** | [Next: 04 — Exception Handling →](./04-exception-handling.md)
