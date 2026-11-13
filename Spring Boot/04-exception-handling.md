# 04 — Exception Handling in Spring Boot
[🏠 Course Index](./00-course-index.md) | [← 03 — REST API](./03-rest-api.md) | **Module 04** | [Next: 05 — Database & JPA →](./05-database-jpa.md)

---

> **Series:** Spring Boot Mastery | **Level:** Intermediate → Advanced  
> **Spring Boot Version:** 3.x / Spring Framework 6.x | **Java Baseline:** Java 17/21+ (Jakarta EE 10)  
> **Focus:** RFC 7807 ProblemDetail, multi-approach error architecture, validation fault isolation, transaction rollback rules, and filter error handling.

---

## Table of Contents

1. [Exception Hierarchy & Spring Translation Infrastructure](#1-exception-hierarchy--spring-translation-infrastructure)
2. [Exception Handling Approaches in Spring Boot (6 Approaches)](#2-exception-handling-approaches-in-spring-boot-6-approaches)
   - [2.1 Approach 1: Controller-Level `@ExceptionHandler`](#21-approach-1-controller-level-exceptionhandler)
   - [2.2 Approach 2: Global `@RestControllerAdvice` Extending `ResponseEntityExceptionHandler`](#22-approach-2-global-restcontrolleradvice-extending-responseentityexceptionhandler)
   - [2.3 Approach 3: RFC 7807 `ProblemDetail` Specification (Spring Boot 3 Standard)](#23-approach-3-rfc-7807-problemdetail-specification-spring-boot-3-standard)
   - [2.4 Approach 4: Custom `ErrorController` (Servlet & Filter Layer Trapping)](#24-approach-4-custom-errorcontroller-servlet--filter-layer-trapping)
   - [2.5 Approach 5: Custom `DefaultErrorAttributes` Bean (Metadata Enrichment)](#25-approach-5-custom-defaulterrorattributes-bean-metadata-enrichment)
   - [2.6 Approach 6: Programmatic `ResponseStatusException` (Inline Fast Failure)](#26-approach-6-programmatic-responsestatusexception-inline-fast-failure)
   - [2.7 Exception Handling Approaches Comparison Matrix](#27-exception-handling-approaches-comparison-matrix)
3. [Validation Error Handling (3 Approaches)](#3-validation-error-handling-3-approaches)
   - [3.1 Approach 1: `@RequestBody` Validation Failures (`MethodArgumentNotValidException`)](#31-approach-1-requestbody-validation-failures-methodargumentnotvalidexception)
   - [3.2 Approach 2: Path & Query Parameter Constraint Violations (`ConstraintViolationException`)](#32-approach-2-path--query-parameter-constraint-violations-constraintviolationexception)
   - [3.3 Approach 3: Malformed JSON Stream (`HttpMessageNotReadableException`)](#33-approach-3-malformed-json-stream-httpmessagenotreadableexception)
   - [3.4 Validation Handling Comparison Matrix](#34-validation-handling-comparison-matrix)
4. [Transaction & Exception Rollback Interactions (3 Approaches)](#4-transaction--exception-rollback-interactions-3-approaches)
   - [4.1 Approach 1: Declarative `rollbackFor = Exception.class`](#41-approach-1-declarative-rollbackfor--exceptionclass)
   - [4.2 Approach 2: Programmatic Rollback via `TransactionAspectSupport`](#42-approach-2-programmatic-rollback-via-transactionaspectsupport)
   - [4.3 Approach 3: Selective Non-Rollback via `noRollbackFor`](#43-approach-3-selective-non-rollback-via-norollbackfor)
   - [4.4 Rollback Approaches Comparison Matrix](#44-rollback-approaches-comparison-matrix)
5. [Outbound HTTP Client Error Handling (4 Approaches)](#5-outbound-http-client-error-handling-4-approaches)
   - [5.1 Approach 1: Modern `RestClient` with `.onStatus()` (Boot 3.2+)](#51-approach-1-modern-restclient-with-onstatus-boot-32)
   - [5.2 Approach 2: Declarative HTTP Interfaces Exception Translators](#52-approach-2-declarative-http-interfaces-exception-translators)
   - [5.3 Approach 3: Reactive `WebClient` with `onStatus()` / `onErrorResume()`](#53-approach-3-reactive-webclient-with-onstatus--onerrorresume)
   - [5.4 Approach 4: Legacy `RestTemplate` with `ResponseErrorHandler`](#54-approach-4-legacy-resttemplate-with-responseerrorhandler)
   - [5.5 HTTP Client Error Handling Comparison Matrix](#55-http-client-error-handling-comparison-matrix)
6. [Asynchronous & Background Job Exception Handling](#6-asynchronous--background-job-exception-handling)
7. [Logging Best Practices & MDC Correlation Tracing](#7-logging-best-practices--mdc-correlation-tracing)
8. [⚠️ High-Frequency Interview Traps & Exception Gotchas](#8-️-high-frequency-interview-traps--exception-gotchas)
9. [🔍 Deep Dive — Internals of `HandlerExceptionResolverChain`](#9--deep-dive--internals-of-handlerexceptionresolverchain)
10. [📝 Architecture Principle Checklist](#10--architecture-principle-checklist)
11. [❓ Practice Questions & Answers](#11--practice-questions--answers)

---

## 1. Exception Hierarchy & Spring Translation Infrastructure

### 1.1 Java Exception Hierarchy & Spring's Philosophy

```
Throwable
├── Error (JVM fatal: OutOfMemoryError, StackOverflowError — NEVER catch)
└── Exception
    ├── RuntimeException (Unchecked — all Spring exceptions extend this)
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   └── DataAccessException (Spring persistence hierarchy)
    └── IOException, SQLException (Checked exceptions — require try/catch)
```

> [!IMPORTANT]
> **Spring's Core Philosophy**: Every single framework-level exception in Spring is **unchecked** (extends `RuntimeException`). Checked exceptions force boilerplate `try-catch` blocks and clutter domain logic. Because Spring's exceptions are unchecked, `@Transactional` defaults to rolling back **only** on unchecked exceptions (`RuntimeException` and `Error`).

---

### 1.2 Spring Persistence Exception Translation Infrastructure

Spring automatically translates low-level database-specific exceptions into a unified, technology-agnostic `DataAccessException` hierarchy.

```
DataAccessException (Abstract RuntimeException)
├── DataIntegrityViolationException       (Foreign key, unique index violation)
├── CannotAcquireLockException            (Database deadlock / lock timeout)
├── OptimisticLockingFailureException     (@Version mismatch)
├── PessimisticLockingFailureException    (SELECT FOR UPDATE timeout)
├── EmptyResultDataAccessException        (Expected 1 row, got 0)
└── QueryTimeoutException                 (SQL execution exceeded timeout)
```

```java
// Hibernate raw exception -> Spring unified exception translation:
// org.hibernate.exception.ConstraintViolationException ──► org.springframework.dao.DataIntegrityViolationException
// org.hibernate.StaleObjectStateException              ──► org.springframework.dao.OptimisticLockingFailureException
```

---

## 2. Exception Handling Approaches in Spring Boot (6 Approaches)

Spring Boot 3.x provides six distinct strategies for trapping, handling, and responding to application errors.

---

### 2.1 Approach 1: Controller-Level `@ExceptionHandler`

#### Scenario
- Controller-specific error logic that applies **only** to methods inside a single controller class.

#### Use-Case
A `FileUploadController` that specifically traps `MaxUploadSizeExceededException` to return a tailored multipart upload error message without altering error handling across other controllers.

#### Benefits Over Others
- Encapsulated within the controller; zero risk of cross-controller regression.
- Takes precedence over any global `@ControllerAdvice` handlers for that specific controller.

#### Limitations
- Code duplication if the same exception must be handled identically across multiple controllers.

#### Documented Code Example

```java
package com.enterprise.file.web;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MaxUploadSizeExceededException;

import java.util.Map;

@RestController
@RequestMapping("/api/v1/files")
public class FileUploadController {

    @PostMapping("/upload")
    public ResponseEntity<String> uploadFile() {
        // Business logic...
        return ResponseEntity.ok("Uploaded successfully");
    }

    /**
     * Controller-Level Exception Handler:
     * Intercepts MaxUploadSizeExceededException thrown by ANY method inside FileUploadController only!
     */
    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<Map<String, Object>> handleMaxSizeException(MaxUploadSizeExceededException ex) {
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE).body(Map.of(
                "error", "FILE_TOO_LARGE",
                "message", "Maximum allowable file size is 10MB.",
                "status", HttpStatus.PAYLOAD_TOO_LARGE.value()
        ));
    }
}
```

---

### 2.2 Approach 2: Global `@RestControllerAdvice` Extending `ResponseEntityExceptionHandler`

#### Scenario
- The enterprise standard for centralized, application-wide REST error handling.
- Overriding standard Spring MVC HTTP exceptions (`400`, `404`, `405`, `415`, `500`) with consistent corporate error structures.

#### Use-Case
A corporate microservice standardizing all error payloads into a unified format while inheriting pre-built handlers for 15+ standard Spring MVC exceptions.

#### Benefits Over Others
- **Centralized Single Source of Truth**: One global class catches exceptions from all `@RestController` beans.
- **Pre-Built Spring MVC Overrides**: Extending `ResponseEntityExceptionHandler` provides robust, pre-written override points for `handleMethodArgumentNotValid()`, `handleNoHandlerFoundException()`, and `handleHttpMessageNotReadable()`.

#### Documented Code Example

```java
package com.enterprise.web.advice;

import com.enterprise.common.exception.BusinessException;
import com.enterprise.common.exception.ResourceNotFoundException;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.HttpStatusCode;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalApiExceptionHandler extends ResponseEntityExceptionHandler {

    // 1. Handling custom domain exceptions
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Object> handleResourceNotFound(ResourceNotFoundException ex, WebRequest request) {
        Map<String, Object> body = Map.of(
                "timestamp", Instant.now(),
                "status", HttpStatus.NOT_FOUND.value(),
                "error", "RESOURCE_NOT_FOUND",
                "message", ex.getMessage()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }

    // 2. Overriding standard Spring MVC validation handling from ResponseEntityExceptionHandler
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {

        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
                fieldErrors.put(err.getField(), err.getDefaultMessage()));

        Map<String, Object> responseBody = Map.of(
                "timestamp", Instant.now(),
                "status", status.value(),
                "error", "VALIDATION_FAILED",
                "fieldErrors", fieldErrors
        );
        return new ResponseEntity<>(responseBody, headers, status);
    }
}
```

---

### 2.3 Approach 3: RFC 7807 `ProblemDetail` Specification (Spring Boot 3 Standard)

#### Scenario
- Modern REST APIs complying with the **IETF RFC 7807 (Problem Details for HTTP APIs)** international standard.
- First-class native support introduced in Spring Boot 3.x and Spring Framework 6.x.

#### Use-Case
An enterprise public API where frontend teams, third-party mobile developers, and API consumers require an industry-standard error format (`application/problem+json`).

#### Benefits Over Others
- **IETF Industry Standard**: Native interoperability with modern API gateways, client libraries, and SDK generators.
- **Zero Custom DTO Boilerplate**: `ProblemDetail` is a built-in Spring class (`org.springframework.http.ProblemDetail`) containing standard RFC fields (`type`, `title`, `status`, `detail`, `instance`) and extensible properties.

#### Documented Code Example

```java
package com.enterprise.web.advice;

import com.enterprise.common.exception.InsufficientBalanceException;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.time.Instant;

@RestControllerAdvice
public class ProblemDetailExceptionHandler {

    @ExceptionHandler(InsufficientBalanceException.class)
    public ProblemDetail handleInsufficientBalance(InsufficientBalanceException ex, HttpServletRequest request) {
        // 1. Create standard RFC 7807 ProblemDetail object
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.PAYMENT_REQUIRED, ex.getMessage());
        problem.setTitle("Insufficient Account Balance");
        problem.setType(URI.create("https://api.enterprise.com/errors/insufficient-balance"));
        problem.setInstance(URI.create(request.getRequestURI()));

        // 2. Add custom extended attributes
        problem.setProperty("accountId", ex.getAccountId());
        problem.setProperty("availableBalance", ex.getAvailableBalance());
        problem.setProperty("requiredAmount", ex.getRequiredAmount());
        problem.setProperty("timestamp", Instant.now());

        return problem; // Automatically serialized as Content-Type: application/problem+json
    }
}
```

Enabling standard Spring MVC ProblemDetail auto-generation in `application.yml`:
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true # Automatically converts all standard Spring MVC exceptions to RFC 7807 ProblemDetail
```

---

### 2.4 Approach 4: Custom `ErrorController` (Servlet & Filter Layer Trapping)

#### Scenario
- Handling exceptions thrown **outside of `DispatcherServlet`**: Filter chain exceptions (JWT parsing in Spring Security), 404 No Handler Found, or errors in container threads before reaching Spring MVC.

#### Use-Case
A security gateway that catches invalid JWT bearer tokens or malformed TLS certificates thrown inside a `OncePerRequestFilter`, rendering a clean JSON response instead of the Tomcat Whitelabel HTML page.

#### Benefits Over Others
- Intercepts errors that **never reached `@ControllerAdvice`** (e.g. security filter exceptions, 404s when `throwExceptionIfNoHandlerFound` is false).
- Fully overrides Spring Boot's internal `BasicErrorController` at path `/error`.

#### Documented Code Example

```java
package com.enterprise.web.controller;

import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.boot.web.servlet.error.ErrorController;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.Instant;
import java.util.Map;

@RestController
public class CustomErrorFallbackController implements ErrorController {

    @RequestMapping("/error")
    public ResponseEntity<Map<String, Object>> handleError(HttpServletRequest request) {
        Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        Object message = request.getAttribute(RequestDispatcher.ERROR_MESSAGE);
        Object exception = request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

        int statusCode = (status != null) ? Integer.parseInt(status.toString()) : 500;
        HttpStatus httpStatus = HttpStatus.resolve(statusCode);
        if (httpStatus == null) httpStatus = HttpStatus.INTERNAL_SERVER_ERROR;

        Map<String, Object> errorPayload = Map.of(
                "timestamp", Instant.now(),
                "status", statusCode,
                "error", httpStatus.getReasonPhrase(),
                "message", (message != null && !message.toString().isBlank()) ? message : "Unexpected servlet failure",
                "path", request.getAttribute(RequestDispatcher.ERROR_REQUEST_URI)
        );

        return new ResponseEntity<>(errorPayload, httpStatus);
    }
}
```

---

### 2.5 Approach 5: Custom `DefaultErrorAttributes` Bean (Metadata Enrichment)

#### Scenario
- Retaining Spring Boot's default `BasicErrorController` routing while injecting enterprise metadata (correlation ID, git commit hash, environment name) or stripping sensitive internal stack traces.

#### Use-Case
Injecting the logging MDC correlation ID into every default `/error` response.

#### Benefits Over Others
- Minimally invasive: Does not alter controller routing; simply customizes the error attributes map.

#### Documented Code Example

```java
package com.enterprise.web.attributes;

import org.slf4j.MDC;
import org.springframework.boot.web.error.ErrorAttributeOptions;
import org.springframework.boot.web.servlet.error.DefaultErrorAttributes;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.WebRequest;

import java.util.Map;

@Component
public class EnterpriseErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(WebRequest webRequest, ErrorAttributeOptions options) {
        // Retrieve base attributes (timestamp, status, error, path)
        Map<String, Object> errorAttributes = super.getErrorAttributes(webRequest, options);

        // 1. Inject enterprise metadata
        errorAttributes.put("correlationId", MDC.get("correlationId"));
        errorAttributes.put("serviceName", "billing-service");

        // 2. Security hardening: Strip sensitive stack trace and exception class names in prod
        errorAttributes.remove("trace");
        errorAttributes.remove("exception");

        return errorAttributes;
    }
}
```

---

### 2.6 Approach 6: Programmatic `ResponseStatusException` (Inline Fast Failure)

#### Scenario
- Small services, internal tools, or controller methods where creating dedicated custom exception classes is disproportionately heavyweight.

#### Use-Case
A simple lookup method failing fast with a 404 or 400 without defining a `UserNotFoundException` class.

#### Benefits Over Others
- Zero custom class creation: `throw new ResponseStatusException(HttpStatus.NOT_FOUND, "User not found")`.
- Directly recognized by Spring's `ResponseStatusExceptionResolver`.

#### Limitations
- Can lead to inconsistent error response schemas if used indiscriminately across large teams.

#### Documented Code Example

```java
package com.enterprise.catalog.web;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api/v1/products")
public class ProductQuickController {

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(
                        HttpStatus.NOT_FOUND, 
                        "Product with id " + id + " does not exist"
                ));
    }
}
```

---

### 2.7 Exception Handling Approaches Comparison Matrix

| Approach | Scope | Layer | RFC 7807 Compliant? | Handles Filter Exceptions? | Best For |
|---|---|---|---|---|---|
| **Controller `@ExceptionHandler`**| Single Controller | Spring MVC | Custom | ❌ No | Controller-specific errors (Multipart size) |
| **`@RestControllerAdvice`** | Global (All Controllers)| Spring MVC | Custom / Native | ❌ No | 🌟 **Enterprise standard REST errors** |
| **RFC 7807 `ProblemDetail`** | Global / Method | Spring MVC | 🌟 **Native Spec** | ❌ No | 🌟 **Public APIs & Microservices standard** |
| **Custom `ErrorController`** | Global | Servlet Container | Custom | 🌟 **Yes** | 🌟 **Filter / Security / 404 errors** |
| **Custom `ErrorAttributes`** | Default `/error` | Spring Boot | Partial | 🌟 **Yes** | Injecting MDC trace IDs into default errors |
| **`ResponseStatusException`** | Single Method | Spring MVC | Supported in Boot 3 | ❌ No | Quick prototyping / Internal tools |

---

## 3. Validation Error Handling (3 Approaches)

---

### 3.1 Approach 1: `@RequestBody` Validation Failures (`MethodArgumentNotValidException`)
- **Origin**: Triggered when `@Valid` or `@Validated` on a `@RequestBody` parameter fails constraint checks.
- **Handling**: Extract field-level errors from `BindingResult` and format into a field-error map.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ProblemDetail handleMethodArgumentNotValid(MethodArgumentNotValidException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Payload validation failed");
    problem.setTitle("Invalid Request Content");

    Map<String, String> fieldErrors = new HashMap<>();
    for (FieldError error : ex.getBindingResult().getFieldErrors()) {
        fieldErrors.put(error.getField(), error.getDefaultMessage());
    }
    problem.setProperty("invalidFields", fieldErrors);
    return problem;
}
```

---

### 3.2 Approach 2: Path & Query Parameter Constraint Violations (`ConstraintViolationException`)
- **Origin**: Triggered when a controller class is annotated with `@Validated`, and a `@PathVariable` or `@RequestParam` fails constraints (e.g. `@Min(1)`).
- **Handling**: Intercept `jakarta.validation.ConstraintViolationException` and unpack property paths.

```java
@ExceptionHandler(ConstraintViolationException.class)
public ProblemDetail handleConstraintViolation(ConstraintViolationException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Parameter constraint violation");
    problem.setTitle("Validation Constraint Violation");

    Map<String, String> violations = new HashMap<>();
    for (ConstraintViolation<?> violation : ex.getConstraintViolations()) {
        violations.put(violation.getPropertyPath().toString(), violation.getMessage());
    }
    problem.setProperty("violations", violations);
    return problem;
}
```

---

### 3.3 Approach 3: Malformed JSON Stream (`HttpMessageNotReadableException`)
- **Origin**: Occurs when Jackson fails to deserialize incoming bytes (e.g. unparseable JSON, string passed for numeric integer, invalid enum string).
- **Handling**: Prevent internal Jackson stack traces from leaking; provide clean client feedback.

```java
@ExceptionHandler(HttpMessageNotReadableException.class)
public ProblemDetail handleMalformedJson(HttpMessageNotReadableException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Malformed JSON request body");
    problem.setTitle("Malformed Payload");
    problem.setProperty("hint", "Verify field data types and JSON syntax");
    return problem;
}
```

---

### 3.4 Validation Handling Comparison Matrix

| Exception Type | Source Annotation | Target Data Source | Default HTTP Status |
|---|---|---|---|
| **`MethodArgumentNotValidException`** | `@Valid @RequestBody` | JSON / XML Request Body | `400 Bad Request` |
| **`ConstraintViolationException`** | `@Validated` (class-level) | `@PathVariable`, `@RequestParam` | `400 Bad Request` |
| **`HttpMessageNotReadableException`**| N/A (Jackson parser) | Unparseable JSON syntax | `400 Bad Request` |
| **`MissingServletRequestParameterException`**| `@RequestParam(required=true)`| Missing Query Parameter | `400 Bad Request` |

---

## 4. Transaction & Exception Rollback Interactions (3 Approaches)

A major source of production data corruption is misunderstanding how exceptions interact with Spring's `@Transactional` boundary.

---

### 4.1 Approach 1: Declarative `rollbackFor = Exception.class` (Recommended Standard)

#### Scenario
- Any method that throws checked exceptions (e.g. `IOException`, `PaymentDeclinedException`, `NoSuchFileException`) where the database transaction **must roll back**.

#### The Gotcha
By default, Spring `@Transactional` **only rolls back for unchecked exceptions** (`RuntimeException` and `Error`). If a method throws a checked `Exception`, Spring commits the database transaction!

#### Documented Code Example

```java
@Service
public class OrderFulfillmentService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;

    public OrderFulfillmentService(OrderRepository orderRepository, InventoryClient inventoryClient) {
        this.orderRepository = orderRepository;
        this.inventoryClient = inventoryClient;
    }

    /**
     * CRITICAL: rollbackFor = Exception.class ensures the transaction rolls back 
     * even if checked exceptions (like InventoryAllocationException) are thrown!
     */
    @Transactional(rollbackFor = Exception.class)
    public void fulfillOrder(Long orderId) throws InventoryAllocationException {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus("PROCESSING");
        orderRepository.save(order);

        // Checked exception thrown here: WITHOUT rollbackFor, the "PROCESSING" status COMMITS to DB!
        inventoryClient.allocateStock(order.getItems());
    }
}
```

---

### 4.2 Approach 2: Programmatic Rollback via `TransactionAspectSupport`

#### Scenario
- Complex business workflows where an error condition is detected, but you want to gracefully return a response object to the caller without throwing an exception, yet **must ensure the database transaction is aborted**.

```java
@Service
public class PaymentProcessingWorkflow {

    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        saveInitialAudit(request);

        boolean thirdPartyApproved = bankGateway.call(request);
        if (!thirdPartyApproved) {
            // Programmatically mark the transaction as ROLLBACK ONLY
            // DB changes will not commit, but no exception is thrown to caller!
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
            return new PaymentResult(false, "Declined by issuing bank");
        }

        finalizeLedger(request);
        return new PaymentResult(true, "Approved");
    }
}
```

---

### 4.3 Approach 3: Selective Non-Rollback via `noRollbackFor`

#### Scenario
- Non-critical exceptions (such as secondary audit failures or non-essential metric reporting errors) where the primary business transaction **should still commit**.

```java
@Transactional(
    rollbackFor = Exception.class,
    noRollbackFor = AuditTelemetryException.class // Audit failure will NOT roll back primary order save!
)
public Order placeOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    try {
        telemetryService.record(order);
    } catch (AuditTelemetryException e) {
        // Logged, but transaction commits normally
        log.warn("Telemetry failed: {}", e.getMessage());
    }
    return order;
}
```

---

### 4.4 Rollback Approaches Comparison Matrix

| Strategy | Trigger Mechanism | Throws Exception to Caller? | Best For |
|---|---|---|---|
| **`rollbackFor = Exception.class`**| Checked or Unchecked Exception | ✅ Yes | 🌟 **Mandatory standard for business methods** |
| **`setRollbackOnly()`** | Programmatic status flag | ❌ No (returns clean response) | Complex multi-branch workflow exits |
| **`noRollbackFor`** | Specific Exception class | ✅ Yes (unless caught) | Non-essential side-effects (audit logs, metrics) |

---

## 5. Outbound HTTP Client Error Handling (4 Approaches)

---

### 5.1 Approach 1: Modern `RestClient` with `.onStatus()` (Boot 3.2+)
- **Scenario**: Spring Boot 3.2+ synchronous client error handling.

```java
@Service
public class PartnerApiClient {
    private final RestClient restClient;

    public PartnerApiClient(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("https://api.partner.com").build();
    }

    public PartnerResponse fetchPartnerData(String id) {
        return restClient.get()
                .uri("/data/{id}", id)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
                    throw new PartnerClientException("Partner rejected request: " + response.getStatusCode());
                })
                .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                    throw new PartnerServerException("Partner server error: " + response.getStatusCode());
                })
                .body(PartnerResponse.class);
    }
}
```

---

### 5.2 Approach 2: Declarative HTTP Interfaces Exception Translators
- **Scenario**: Spring 6 `@HttpExchange` declarative clients.

```java
@Configuration
public class DeclarativeClientConfig {

    @Bean
    public PartnerClient partnerClient(RestClient.Builder builder) {
        RestClient restClient = builder
                .baseUrl("https://api.partner.com")
                .defaultStatusHandler(HttpStatusCode::isError, (req, resp) -> {
                    throw new PartnerApiException("Remote call failed: " + resp.getStatusCode());
                })
                .build();

        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        return factory.createClient(PartnerClient.class);
    }
}
```

---

### 5.3 Approach 3: Reactive `WebClient` with `onStatus()` / `onErrorResume()`
- **Scenario**: WebFlux reactive pipelines.

```java
public Mono<UserData> fetchUserDataReactive(String userId) {
    return webClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, resp -> 
                resp.bodyToMono(String.class).flatMap(msg -> Mono.error(new UserNotFoundException(msg))))
            .bodyToMono(UserData.class)
            .onErrorResume(UserNotFoundException.class, ex -> Mono.just(UserData.empty()));
}
```

---

### 5.4 Approach 4: Legacy `RestTemplate` with `ResponseErrorHandler`
- **Scenario**: Maintenance-mode Spring Boot applications using `RestTemplate`.

```java
public class CustomRestTemplateErrorHandler implements ResponseErrorHandler {
    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().isError();
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        if (response.getStatusCode().is4xxClientError()) {
            throw new HttpClientErrorException(response.getStatusCode());
        }
        throw new HttpServerErrorException(response.getStatusCode());
    }
}
```

---

### 5.5 HTTP Client Error Handling Comparison Matrix

| Client | Mechanism | Error Callback Style | Spring Boot Version |
|---|---|---|---|
| **`RestClient`** | Synchronous | 🌟 **Inline `.onStatus()` chaining** | 🌟 **Boot 3.2+** |
| **`@HttpExchange`** | Declarative Proxy | Global `defaultStatusHandler` on adapter | 🌟 **Boot 3.0+** |
| **`WebClient`** | Reactive Streams | `.onStatus()` + `.onErrorResume()` | Boot 2.0+ / 3.x |
| **`RestTemplate`** | Synchronous | `ResponseErrorHandler` interface | Legacy (Spring 3.0+) |

---

## 6. Asynchronous & Background Job Exception Handling

### 6.1 `@Async` `void` Methods & `AsyncUncaughtExceptionHandler`
Because asynchronous `void` methods execute on a background thread pool, unhandled exceptions are **silently swallowed** unless a global `AsyncUncaughtExceptionHandler` is registered!

```java
@Configuration
@EnableAsync
public class AsyncExceptionConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> {
            System.err.printf("[ASYNC ERROR] Method: %s | Message: %s%n", 
                    method.getName(), throwable.getMessage());
        };
    }
}
```

### 6.2 Scheduled Tasks ErrorHandler
If an unhandled exception escapes a `@Scheduled` task, the Spring scheduler **terminates future executions of that task**! Always configure an `ErrorHandler`:

```java
@Configuration
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setErrorHandler(t -> System.err.println("Scheduled job failure captured: " + t.getMessage()));
        scheduler.initialize();
        taskRegistrar.setTaskScheduler(scheduler);
    }
}
```

---

## 7. Logging Best Practices & MDC Correlation Tracing

### 7.1 What to Log in Exception Handlers
1. **Client Errors (4xx)**: Log at `WARN` or `INFO` level. Do **NOT** log full stack traces for invalid client input!
2. **Server Errors (5xx)**: Log at `ERROR` level **with the full stack trace** (`log.error("Internal failure", ex)`).
3. **Correlation ID**: Always attach the MDC correlation ID to connect log lines to APM traces.

```java
@ExceptionHandler(Exception.class)
public ProblemDetail handleGeneralError(Exception ex) {
    String correlationId = MDC.get("correlationId");
    // Full stack trace logged for 500 errors
    log.error("[CorrelationId: {}] Unhandled server failure: {}", correlationId, ex.getMessage(), ex);

    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "An internal server error occurred.");
    problem.setProperty("correlationId", correlationId);
    return problem;
}
```

---

## 8. ⚠️ High-Frequency Interview Traps & Exception Gotchas

### Trap 1: `@ControllerAdvice` Does NOT Catch Servlet Filter Exceptions
- **Problem**: Exceptions thrown inside a security filter (`JwtAuthFilter`) never reach `@ControllerAdvice` because filters execute *before* `DispatcherServlet`.
- **Solution**: Inject `HandlerExceptionResolver` into the filter and invoke `resolver.resolveException(request, response, null, ex)`.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final HandlerExceptionResolver handlerExceptionResolver;

    public JwtAuthenticationFilter(@Qualifier("handlerExceptionResolver") HandlerExceptionResolver resolver) {
        this.handlerExceptionResolver = resolver;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) 
            throws ServletException, IOException {
        try {
            validateToken(request);
            chain.doFilter(request, response);
        } catch (JwtTokenExpiredException ex) {
            // Forward exception into Spring MVC's @ControllerAdvice!
            handlerExceptionResolver.resolveException(request, response, null, ex);
        }
    }
}
```

### Trap 2: `@Transactional` Commits on Checked Exceptions
- **Problem**: Declaring `throws InsufficientFundsException` (checked) causes `@Transactional` to commit changes by default.
- **Solution**: Always use `@Transactional(rollbackFor = Exception.class)`.

### Trap 3: Catching Exceptions Inside `@Transactional` Without Re-Throwing
- **Problem**: Wrapping DB code in a `try-catch` inside a `@Transactional` method swallows the exception. Spring sees no exception and commits the partial transaction!
- **Solution**: Re-throw the exception or invoke `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`.

---

## 9. 🔍 Deep Dive — Internals of `HandlerExceptionResolverChain`

When an unhandled exception escapes a controller method in `DispatcherServlet.doDispatch()`, Spring invokes `processDispatchResult()`, which delegates to the `HandlerExceptionResolver` chain:

```
DispatcherServlet.processDispatchResult()
                 │
                 ▼
1. ExceptionHandlerExceptionResolver
   └── Matches @ExceptionHandler methods in @ControllerAdvice and Controller
                 │ (if unmatched)
                 ▼
2. ResponseStatusExceptionResolver
   └── Checks for @ResponseStatus on exception class or ResponseStatusException
                 │ (if unmatched)
                 ▼
3. DefaultHandlerExceptionResolver
   └── Standard Spring MVC status codes (e.g. 405 Method Not Allowed, 400 Bad Request)
                 │ (if unmatched)
                 ▼
4. BasicErrorController (/error endpoint)
   └── Final container fallback; renders Whitelabel HTML or default JSON
```

---

## 10. 📝 Architecture Principle Checklist

- [ ] Use **RFC 7807 `ProblemDetail`** for all REST API error responses.
- [ ] Extend **`ResponseEntityExceptionHandler`** in your global `@RestControllerAdvice`.
- [ ] Declare **`rollbackFor = Exception.class`** on all `@Transactional` services.
- [ ] Forward Servlet Filter exceptions to **`HandlerExceptionResolver`** so they reach `@ControllerAdvice`.
- [ ] Configure **`AsyncUncaughtExceptionHandler`** to prevent swallowed exceptions in `@Async void` methods.
- [ ] Log 4xx errors at `WARN` (without stack traces) and 5xx errors at `ERROR` (with full stack traces and MDC correlation IDs).

---

## 11. ❓ Practice Questions & Answers

### Q1: Why do exceptions thrown in a Servlet Filter bypass `@RestControllerAdvice`?
**Answer**: `@RestControllerAdvice` is registered inside Spring MVC's `DispatcherServlet`. Servlet Filters run inside the servlet container *before* the request reaches `DispatcherServlet`. If an exception is thrown in a Filter, it bubbles up directly to Tomcat rather than entering `DispatcherServlet`. To handle it in `@RestControllerAdvice`, the filter must catch the exception and delegate to `HandlerExceptionResolver.resolveException(request, response, null, ex)`.

### Q2: How does Spring Boot 3.x implement RFC 7807 Problem Details?
**Answer**: Spring Boot 3.x and Spring 6 introduce the `ProblemDetail` class (`org.springframework.http.ProblemDetail`), representing RFC 7807 fields (`type`, `title`, `status`, `detail`, `instance`). Enabling `spring.mvc.problemdetails.enabled=true` instructs `ResponseEntityExceptionHandler` to serialize all standard Spring MVC exceptions as `application/problem+json`. Custom handlers can also return `ProblemDetail` directly.

### Q3: What happens if an `@Async` method with a `void` return type throws an exception?
**Answer**: The exception cannot propagate to the caller because the caller thread has already returned. Without explicit configuration, the exception is logged and silently swallowed by Spring's simple asynchronous task executor. To trap it, implement `AsyncConfigurer` and override `getAsyncUncaughtExceptionHandler()`.

---

[🏠 Course Index](./00-course-index.md) | [← 03 — REST API](./03-rest-api.md) | **Module 04** | [Next: 05 — Database & JPA →](./05-database-jpa.md)
