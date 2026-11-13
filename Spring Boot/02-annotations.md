# 02 — Spring Boot Annotations: The Complete Reference
[🏠 Course Index](./00-course-index.md) | [← 01 — Fundamentals](./01-fundamentals.md) | **Module 02** | [Next: 03 — REST API →](./03-rest-api.md)

---

> **Series:** Spring Boot Mastery | **Level:** Intermediate → Advanced  
> **Spring Boot Version:** 3.x / Spring Framework 6.x | **Java Baseline:** Java 17/21+ (Jakarta EE 10)  
> **Focus:** Architectural semantics, multi-approach bean definitions, AOP proxy mechanics, method security, and validation strategies.

---

## Table of Contents

1. [Introduction — Why Annotations Matter](#1-introduction)
2. [Stereotype Annotations: Architectural Selection (5 Approaches)](#2-stereotype-annotations-architectural-selection-5-approaches)
   - [2.1 Approach 1: `@Component` — Generic Infrastructure Bean](#21-approach-1-component--generic-infrastructure-bean)
   - [2.2 Approach 2: `@Service` — Domain Business Logic](#22-approach-2-service--domain-business-logic)
   - [2.3 Approach 3: `@Repository` — Data Access & Exception Translation](#23-approach-3-repository--data-access--exception-translation)
   - [2.4 Approach 4: `@Controller` — Presentation & View Resolution](#24-approach-4-controller--presentation--view-resolution)
   - [2.5 Approach 5: `@RestController` — Modern JSON/XML APIs](#25-approach-5-restcontroller--modern-jsonxml-apis)
   - [2.6 Stereotype Comparison Matrix](#26-stereotype-comparison-matrix)
3. [Dependency Injection Annotations](#3-dependency-injection-annotations)
   - [`@Autowired`, `@Qualifier`, `@Primary`, `@Resource`, `@Inject`]
4. [Bean Configuration & Registration (5 Approaches)](#4-bean-configuration--registration-5-approaches)
   - [4.1 Approach 1: Stereotype Component Scanning](#41-approach-1-stereotype-component-scanning)
   - [4.2 Approach 2: Full `@Configuration` with `@Bean` (CGLIB Mode)](#42-approach-2-full-configuration-with-bean-cglib-mode)
   - [4.3 Approach 3: Lite Mode Configuration (`proxyBeanMethods = false`)](#43-approach-3-lite-mode-configuration-proxybeanmethods--false)
   - [4.4 Approach 4: Conditional Auto-Configuration (`@ConditionalOn*`)](#44-approach-4-conditional-auto-configuration-conditionalon)
   - [4.5 Approach 5: Dynamic Programmatic Registration (`ImportBeanDefinitionRegistrar`)](#45-approach-5-dynamic-programmatic-registration-importbeandefinitionregistrar)
   - [4.6 Configuration Approaches Comparison](#46-configuration-approaches-comparison)
5. [Web & REST Data Binding (6 Approaches)](#5-web--rest-data-binding-6-approaches)
   - [5.1 Approach 1: `@PathVariable` (Resource Identity)](#51-approach-1-pathvariable-resource-identity)
   - [5.2 Approach 2: `@RequestParam` (Query Filtering & Pagination)](#52-approach-2-requestparam-query-filtering--pagination)
   - [5.3 Approach 3: `@RequestBody` with `@Valid` (Structured JSON Payload)](#53-approach-3-requestbody-with-valid-structured-json-payload)
   - [5.4 Approach 4: `@ModelAttribute` (Form Encoded & Multipart Data)](#54-approach-4-modelattribute-form-encoded--multipart-data)
   - [5.5 Approach 5: `@RequestHeader` (HTTP Headers & Context Propagation)](#55-approach-5-requestheader-http-headers--context-propagation)
   - [5.6 Approach 6: `@CookieValue` (Session Markers & Tracking)](#56-approach-6-cookievalue-session-markers--tracking)
   - [5.7 Web Binding Approaches Comparison](#57-web-binding-approaches-comparison)
6. [Data & JPA Persistence Annotations](#6-data--jpa-persistence-annotations)
   - [Entity Mapping, Generation Strategies, Relationships, and `@Transactional`]
7. [Exception Handling Annotations](#7-exception-handling-annotations)
   - [`@ExceptionHandler`, `@RestControllerAdvice`, `@ResponseStatus`]
8. [Scheduling & Asynchronous Execution (7 Approaches)](#8-scheduling--asynchronous-execution-7-approaches)
   - [8.1 Scheduling Approaches: Fixed Rate vs Fixed Delay vs Cron vs ShedLock](#81-scheduling-approaches-fixed-rate-vs-fixed-delay-vs-cron-vs-shedlock)
   - [8.2 Async Execution Approaches: Void vs CompletableFuture vs Virtual Threads](#82-async-execution-approaches-void-vs-completablefuture-vs-virtual-threads)
9. [Caching Annotations](#9-caching-annotations)
   - [`@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`]
10. [Method Security Approaches (Spring Boot 3.x / Security 6)](#10-method-security-approaches-spring-boot-3x--security-6)
    - [10.1 Approach 1: `@PreAuthorize` with SpEL](#101-approach-1-preauthorize-with-spel)
    - [10.2 Approach 2: `@PostAuthorize` (Post-Execution Inspection)](#102-approach-2-postauthorize-post-execution-inspection)
    - [10.3 Approach 3: `@Secured` (Simple Role Checking)](#103-approach-3-secured-simple-role-checking)
    - [10.4 Approach 4: `@RolesAllowed` (JSR-250 Portability)](#104-approach-4-rolesallowed-jsr-250-portability)
    - [10.5 Method Security Comparison](#105-method-security-comparison)
11. [Testing Slice Annotations](#11-testing-slice-annotations)
    - [`@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean`]
12. [Lifecycle & Conditional Annotations](#12-lifecycle--conditional-annotations)
13. [Validation Annotations (4 Approaches)](#13-validation-annotations-4-approaches)
    - [13.1 Approach 1: Payload DTO Validation (`@Valid @RequestBody`)](#131-approach-1-payload-dto-validation-valid-requestbody)
    - [13.2 Approach 2: Parameter-Level Validation (`@Validated`)](#132-approach-2-parameter-level-validation-validated)
    - [13.3 Approach 3: Validation Groups (Create vs Update Rules)](#133-approach-3-validation-groups-create-vs-update-rules)
    - [13.4 Approach 4: Custom Constraint Annotation (`ConstraintValidator`)](#134-approach-4-custom-constraint-annotation-constraintvalidator)
14. [⚠️ Interview Traps & Gotchas — The AOP Proxy Dilemma](#14-️-interview-traps--gotchas--the-aop-proxy-dilemma)
    - [Solving Self-Invocation Proxy Bypass (5 Architectural Approaches)](#solving-self-invocation-proxy-bypass-5-architectural-approaches)
15. [🔍 Deep Dive — Spring AOP CGLIB vs JDK Dynamic Proxies](#15--deep-dive--spring-aop-cglib-vs-jdk-dynamic-proxies)
16. [📝 Summary & Best Practice Rules](#16--summary--best-practice-rules)
17. [❓ Practice Questions & Answers](#17--practice-questions--answers)

---

## 1. Introduction

In Spring Boot 3.x, annotations are not passive docstrings; they are **declarative bytecode and runtime metadata instructions**. At bootstrap, the container scans and processes annotations through three primary mechanisms:

1. **Classpath Scanning**: Discovering components and building `BeanDefinition` metadata.
2. **Proxy-based AOP Weaving**: Intercepting method execution for transactions, caching, security, and async processing via CGLIB subclasses.
3. **Reflective Pipeline Binding**: Resolving HTTP parameters, validating DTO constraints, and binding configuration YAML files.

---

## 2. Stereotype Annotations: Architectural Selection (5 Approaches)

All stereotype annotations derive meta-annotationally from `@Component`. However, each stereotype communicates architectural layer intent and, in some cases, activates technical runtime behavior.

```
                    @Component (Root Stereotype)
                         ▲
         ┌───────────────┼───────────────┐
         │               │               │
      @Service      @Repository     @Controller
                                         ▲
                                         │
                                  @RestController
```

---

### 2.1 Approach 1: `@Component` — Generic Infrastructure Bean

#### Scenario
- Utilities, helper classes, custom validators, interceptors, or cross-cutting tools that do not naturally belong to the presentation, service, or persistence layers.

#### Use-Case
An `AesEncryptionHelper` that provides cryptographic encryption and decryption across disparate microservice modules.

#### Benefits Over Others
- Root annotation with minimal assumptions.
- Automatically discovered by `@ComponentScan`.

#### Documented Code Example

```java
package com.enterprise.common.security;

import org.springframework.stereotype.Component;
import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

@Component
public class AesEncryptionHelper {

    private static final String ALGORITHM = "AES";

    public String encrypt(String plainText, String secretKey) {
        try {
            SecretKeySpec key = new SecretKeySpec(secretKey.getBytes(StandardCharsets.UTF_8), ALGORITHM);
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, key);
            byte[] encrypted = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(encrypted);
        } catch (Exception e) {
            throw new IllegalStateException("Encryption failure", e);
        }
    }
}
```

---

### 2.2 Approach 2: `@Service` — Domain Business Logic

#### Scenario
- Core domain business logic, orchestration of multiple repositories, transaction boundaries, and external API coordination.

#### Use-Case
A `CheckoutWorkflowService` that validates shopping cart inventory, executes credit card charges, records orders in PostgreSQL, and enqueues fulfillment events.

#### Benefits Over Others
- Communicates pure business domain intent to developers and architectural linters.
- Standard target for enterprise AOP pointcuts (e.g. `@Pointcut("within(@org.springframework.stereotype.Service *)")` to measure business service latency).

#### Documented Code Example

```java
package com.enterprise.billing.service;

import com.enterprise.billing.domain.Order;
import com.enterprise.billing.repository.OrderRepository;
import com.enterprise.payment.service.PaymentGateway;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CheckoutWorkflowService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public CheckoutWorkflowService(OrderRepository orderRepository, PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }

    @Transactional // Business transactional boundary
    public Order completeCheckout(Long customerId, double totalAmount) {
        paymentGateway.charge(customerId, totalAmount);
        Order order = new Order(customerId, totalAmount, "PAID");
        return orderRepository.save(order);
    }
}
```

---

### 2.3 Approach 3: `@Repository` — Data Access & Exception Translation

#### Scenario
- Custom Data Access Objects (DAOs) executing native SQL, JDBC queries, or direct Hibernate `EntityManager` calls.

#### Use-Case
A custom JDBC DAO executing complex analytical queries using Spring's `JdbcTemplate`.

#### Benefits Over Others
- **Automatic Exception Translation**: Unlike `@Component` or `@Service`, `@Repository` enables `PersistenceExceptionTranslationPostProcessor`. It intercepts technology-specific exceptions (e.g. `java.sql.SQLException`, `org.hibernate.HibernateException`) and translates them into Spring's unified, unchecked `DataAccessException` hierarchy (e.g. `DataIntegrityViolationException`, `CannotAcquireLockException`).

#### Documented Code Example

```java
package com.enterprise.analytics.repository;

import org.springframework.dao.DataAccessException;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.time.LocalDate;
import java.util.Map;

@Repository
public class DailyRevenueDao {

    private final JdbcTemplate jdbcTemplate;

    public DailyRevenueDao(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Map<String, Object> calculateDailyMetrics(LocalDate date) {
        // Any raw SQLException thrown by the DB driver is automatically intercepted
        // and translated to Spring's portable DataAccessException hierarchy!
        String sql = "SELECT SUM(amount) as total_revenue, COUNT(*) as tx_count FROM orders WHERE order_date = ?";
        return jdbcTemplate.queryForMap(sql, date);
    }
}
```

---

### 2.4 Approach 4: `@Controller` — Presentation & View Resolution

#### Scenario
- Traditional server-side rendered web applications using template engines (Thymeleaf, FreeMarker, JSP).

#### Use-Case
An internal administrative back-office portal that renders HTML pages.

#### Benefits Over Others
- Return strings are interpreted as **View Template Names** resolved by `ViewResolver` rather than serialized as raw JSON/XML to the HTTP response stream.

#### Documented Code Example

```java
package com.enterprise.admin.web;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("/admin")
public class AdminPortalController {

    @GetMapping("/dashboard")
    public String showDashboard(Model model) {
        model.addAttribute("serverStatus", "ONLINE");
        model.addAttribute("activeUsers", 1420);
        return "admin/dashboard"; // Resolves to /templates/admin/dashboard.html
    }
}
```

---

### 2.5 Approach 5: `@RestController` — Modern JSON/XML APIs

#### Scenario
- Stateless RESTful microservices, single-page application (SPA) backends (React/Vue/Angular), and mobile API gateways.

#### Use-Case
An e-commerce API exposing endpoints returning JSON entities with standard HTTP status codes.

#### Benefits Over Others
- Convenience meta-annotation combining `@Controller` and `@ResponseBody`.
- Eliminates the need to annotate every controller method with `@ResponseBody`. All method return values are automatically serialized via `HttpMessageConverter` (Jackson) to JSON.

#### Documented Code Example

```java
package com.enterprise.catalog.api;

import com.enterprise.catalog.domain.Product;
import com.enterprise.catalog.service.ProductService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products")
public class ProductApiController {

    private final ProductService productService;

    public ProductApiController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        // Return object is automatically serialized to JSON in response body
        return productService.findProduct(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

---

### 2.6 Stereotype Comparison Matrix

| Stereotype | Architectural Layer | Default Return Semantics | Exception Translation Enabled? | Standard Pointcut Target |
|---|---|---|---|---|
| **`@Component`** | Generic Utility / Infrastructure | N/A | ❌ No | Low |
| **`@Service`** | Business Logic / Domain Service | N/A | ❌ No | 🌟 High (Transaction & Metric logging) |
| **`@Repository`** | Data Access / Custom DAO | N/A | 🌟 **Yes (`DataAccessException`)** | 🌟 High (Query performance auditing) |
| **`@Controller`** | Presentation (Thymeleaf/MVC) | View Template Name (HTML) | ❌ No | Moderate |
| **`@RestController`**| REST API / Mobile Backend | Serialized Response Body (JSON) | ❌ No | 🌟 High (API metrics & error advice) |

---

## 3. Dependency Injection Annotations

| Annotation | Standard / Package | Resolution Strategy | Optional Injection | Best Use-Case |
|---|---|---|---|---|
| **`@Autowired`** | Spring (`org.springframework...`) | Type first -> Bean Name fallback | Supported via `required=false` | Primary standard for Spring Boot constructors |
| **`@Qualifier`** | Spring (`org.springframework...`) | Disambiguates by explicit bean name | N/A | Disambiguating multiple bean implementations |
| **`@Primary`** | Spring (`org.springframework...`) | Marks default candidate for type | N/A | Providing fallback bean when multiple exist |
| **`@Resource`** | JSR-250 (`jakarta.annotation...`) | Name first -> Type fallback | Not supported | Legacy migration / Name-based injection |
| **`@Inject`** | JSR-330 (`jakarta.inject...`) | Type first -> Bean Name fallback | Not supported | Cross-framework portability (Quarkus/Micronaut) |

---

## 4. Bean Configuration & Registration (5 Approaches)

Spring Boot provides five distinct strategies for declaring and registering beans in the `ApplicationContext`.

---

### 4.1 Approach 1: Stereotype Component Scanning
- **Scenario**: Application code that you write and own directly.
- **Mechanism**: Classpath scanning detects `@Component`, `@Service`, `@Repository`, and `@RestController`.

---

### 4.2 Approach 2: Full `@Configuration` with `@Bean` (CGLIB Mode)
- **Scenario**: Third-party classes where source code cannot be annotated, or internal `@Bean` methods that call each other to share singletons.
- **Mechanism**: CGLIB proxies the configuration class. Direct calls between `@Bean` methods intercept and return the managed singleton from the container.

```java
@Configuration(proxyBeanMethods = true) // Default "Full" mode
public class DatabaseSecurityConfig {

    @Bean
    public DataSourceConfig dataSourceConfig() {
        return new DataSourceConfig("jdbc:postgresql://localhost:5432/db");
    }

    @Bean
    public ConnectionPool connectionPool() {
        // CGLIB intercepts this call! It returns the existing singleton bean rather than creating a second instance!
        return new ConnectionPool(dataSourceConfig());
    }
}
```

---

### 4.3 Approach 3: Lite Mode Configuration (`proxyBeanMethods = false`)
- **Scenario**: High-performance configurations, AOT native compilation (GraalVM), or independent `@Bean` declarations that do not call each other.
- **Benefits**: Eliminates CGLIB bytecode subclassing overhead at startup.

```java
@Configuration(proxyBeanMethods = false) // High-performance "Lite" mode
public class CacheConfiguration {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        // Collaborators are injected via method parameters, eliminating inter-method calls!
        return RedisCacheManager.builder(connectionFactory).build();
    }
}
```

---

### 4.4 Approach 4: Conditional Auto-Configuration (`@ConditionalOn*`)
- **Scenario**: Reusable enterprise starter libraries activating beans conditionally based on classpath, environment properties, or existing beans.

```java
@AutoConfiguration
@ConditionalOnClass(RedisOperations.class)
@ConditionalOnProperty(prefix = "app.cache", name = "type", havingValue = "redis")
public class RedisCacheAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CustomCacheService customCacheService(RedisTemplate<String, Object> template) {
        return new RedisCustomCacheService(template);
    }
}
```

---

### 4.5 Approach 5: Dynamic Programmatic Registration (`ImportBeanDefinitionRegistrar`)
- **Scenario**: Advanced framework extensions, dynamic multi-tenant datasource factories, or custom annotation scanners.
- **Benefits**: Direct, programmatic manipulation of the `BeanDefinitionRegistry`.

```java
package com.enterprise.dynamic;

import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.context.annotation.ImportBeanDefinitionRegistrar;
import org.springframework.core.type.AnnotationMetadata;

public class DynamicClientRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // Dynamically register a bean definition at container bootstrap
        RootBeanDefinition beanDef = new RootBeanDefinition(DynamicApiClient.class);
        beanDef.getPropertyValues().add("endpoint", "https://api.partner.com/v2");
        registry.registerBeanDefinition("partnerApiClient", beanDef);
    }
}
```

---

### 4.6 Configuration Approaches Comparison

| Approach | Configuration Source | CGLIB Overhead | Dynamic At Runtime | Best For |
|---|---|---|---|---|
| **Stereotype Scanning** | Application Source | None | ❌ Static | First-party business logic and controllers |
| **`@Configuration` (Full)**| Java Factory Method | High (CGLIB subclass) | ❌ Static | Inter-dependent `@Bean` method calls |
| **`@Configuration` (Lite)**| Java Factory Method | None | ❌ Static | Independent beans, GraalVM native image |
| **Conditional Auto-Config**| Library Starter | None | ⚠️ Environment conditional | Enterprise shared libraries & starters |
| **`ImportBeanDefinitionRegistrar`**| Bytecode Registry | None | 🌟 **Fully Dynamic** | Multi-tenant dynamic factories & plugins |

---

## 5. Web & REST Data Binding (6 Approaches)

Spring MVC controllers extract incoming HTTP request data using six specialized binding annotations.

---

### 5.1 Approach 1: `@PathVariable` (Resource Identity)
- **Scenario**: Hierarchical RESTful resource identification in the URI path.
- **Use-Case**: `GET /api/v1/tenants/42/orders/1080`

```java
@GetMapping("/tenants/{tenantId}/orders/{orderId}")
public ResponseEntity<Order> getTenantOrder(
        @PathVariable("tenantId") Long tenantId,
        @PathVariable("orderId") Long orderId) {
    return ResponseEntity.ok(orderService.find(tenantId, orderId));
}
```

---

### 5.2 Approach 2: `@RequestParam` (Query Filtering & Pagination)
- **Scenario**: Optional search parameters, sorting keys, and pagination boundaries.
- **Use-Case**: `GET /api/v1/orders?status=PENDING&page=0&size=20`

```java
@GetMapping("/orders")
public ResponseEntity<Page<Order>> listOrders(
        @RequestParam(name = "status", required = false, defaultValue = "ALL") String status,
        @RequestParam(name = "page", defaultValue = "0") int page,
        @RequestParam(name = "size", defaultValue = "25") int size) {
    return ResponseEntity.ok(orderService.search(status, PageRequest.of(page, size)));
}
```

---

### 5.3 Approach 3: `@RequestBody` with `@Valid` (Structured JSON Payload)
- **Scenario**: Inbound JSON/XML request bodies for `POST`, `PUT`, `PATCH` endpoints.
- **Use-Case**: Creating or updating complex domain entities with schema validation.

```java
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.create(request));
}
```

---

### 5.4 Approach 4: `@ModelAttribute` (Form Encoded & Multipart Data)
- **Scenario**: HTML form submissions (`application/x-www-form-urlencoded`) and binary file uploads (`multipart/form-data`).

```java
@PostMapping(value = "/profile/avatar", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<Void> uploadAvatar(@ModelAttribute AvatarUploadForm form) {
    storageService.store(form.getUserId(), form.getFile());
    return ResponseEntity.ok().build();
}
```

---

### 5.5 Approach 5: `@RequestHeader` (HTTP Headers & Context Propagation)
- **Scenario**: Correlation IDs, JWT bearer tokens, client version metadata, and API keys.

```java
@GetMapping("/account")
public ResponseEntity<AccountDetails> getAccount(
        @RequestHeader("X-Correlation-ID") String correlationId,
        @RequestHeader(value = "X-Client-Version", defaultValue = "unknown") String clientVersion) {
    return ResponseEntity.ok(accountService.getDetails(correlationId));
}
```

---

### 5.6 Approach 6: `@CookieValue` (Session Markers & Tracking)
- **Scenario**: Reading HTTP cookies (e.g. session tracking, refresh token storage).

```java
@PostMapping("/auth/refresh")
public ResponseEntity<TokenResponse> refreshToken(
        @CookieValue(name = "refreshToken", required = true) String refreshToken) {
    return ResponseEntity.ok(authService.renewToken(refreshToken));
}
```

---

### 5.7 Web Binding Approaches Comparison

| Annotation | Data Source | Media Type | Validation Support | Typical HTTP Methods |
|---|---|---|---|---|
| **`@PathVariable`** | URI Path Segments | Any | Via `@Validated` on class | `GET`, `DELETE`, `PUT` |
| **`@RequestParam`** | Query String / Form URL | Any | Via `@Validated` on class | `GET`, `DELETE` |
| **`@RequestBody`** | HTTP Body Stream | `application/json`, `application/xml` | 🌟 **`@Valid` Bean Validation** | `POST`, `PUT`, `PATCH` |
| **`@ModelAttribute`**| Form / Multipart Body | `multipart/form-data`, form-urlencoded| 🌟 **`@Valid` Bean Validation** | `POST`, `PUT` |
| **`@RequestHeader`** | HTTP Headers | Plaintext Strings | Via manual/custom checks | All |
| **`@CookieValue`** | `Cookie` Header | Plaintext Strings | Via manual/custom checks | `GET`, `POST` |

---

## 6. Data & JPA Persistence Annotations

```java
@Entity
@Table(name = "customers", indexes = @Index(name = "idx_cust_email", columnList = "email"))
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "cust_seq")
    @SequenceGenerator(name = "cust_seq", sequenceName = "customer_sequence", allocationSize = 50)
    private Long id;

    @Column(nullable = false, unique = true, length = 120)
    private String email;

    @Enumerated(EnumType.STRING) // Store enum text, NEVER ordinal number!
    private AccountTier tier;

    @Version // Optimistic locking mechanism
    private Long version;

    // ALWAYS use LAZY fetching for ManyToOne to avoid N+1 query disasters!
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "organization_id", nullable = false)
    private Organization organization;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}
```

### Transaction Boundaries with `@Transactional`
```java
@Service
@Transactional(readOnly = true) // Class default: read-only optimizations
public class AccountManagementService {

    private final AccountRepository accountRepo;

    public AccountManagementService(AccountRepository accountRepo) {
        this.accountRepo = accountRepo;
    }

    public Account getAccount(Long id) {
        return accountRepo.findById(id).orElseThrow();
    }

    @Transactional(
        isolation = Isolation.READ_COMMITTED,
        propagation = Propagation.REQUIRED,
        rollbackFor = Exception.class // Roll back for checked exceptions as well!
    )
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) throws InsufficientFundsException {
        // Atomic transaction execution...
    }
}
```

---

## 7. Exception Handling Annotations

```java
@RestControllerAdvice // Global advice applied across all @RestControllers
public class GlobalApiExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ProblemDetail handleNotFound(EntityNotFoundException ex, HttpServletRequest request) {
        // RFC 7807 standard error representation in Spring Boot 3.x
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationErrors(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        Map<String, String> errors = new HashMap<>();
        for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {
            errors.put(fieldError.getField(), fieldError.getDefaultMessage());
        }
        problem.setProperty("fieldErrors", errors);
        return problem;
    }
}
```

---

## 8. Scheduling & Asynchronous Execution (7 Approaches)

### 8.1 Scheduling Approaches: Fixed Rate vs Fixed Delay vs Cron vs ShedLock

```
1. fixedRate = 5000ms:
Task Start: 00:00 ──(takes 3s)──► End: 00:03
Task Start: 00:05 ──(starts strictly 5s after PREVIOUS START)

2. fixedDelay = 5000ms:
Task Start: 00:00 ──(takes 3s)──► End: 00:03
                                   └──── 5s pause ────► Task Start: 00:08 (starts 5s after PREVIOUS FINISH)
```

#### Approach 1: `@Scheduled(fixedRate = 10000)`
- **Scenario**: Tasks measuring regular periodic ticks where start-to-start timing must be preserved.

#### Approach 2: `@Scheduled(fixedDelay = 10000)`
- **Scenario**: Tasks where consecutive executions must NEVER overlap and require a mandatory cooldown period after completion.

#### Approach 3: `@Scheduled(cron = "0 0 2 * * ?")`
- **Scenario**: Wall-clock schedules (e.g. daily at 2:00 AM).

#### Approach 4: Distributed ShedLock Integration
- **Scenario**: Multi-instance microservices in Kubernetes where scheduled tasks must execute on **exactly one node**.

```java
@Component
public class DistributedClusterJob {

    @Scheduled(cron = "0 0/15 * * * ?")
    @SchedulerLock(name = "billingReconciliationJob", lockAtLeastFor = "30s", lockAtMostFor = "10m")
    public void executeDistributedTask() {
        // Only ONE Kubernetes pod executes this job at a time!
        System.out.println("Executing distributed cluster task with ShedLock protection.");
    }
}
```

---

### 8.2 Async Execution Approaches: Void vs CompletableFuture vs Virtual Threads

#### Approach 1: Fire-and-Forget `void` with `AsyncUncaughtExceptionHandler`
```java
@Async("notificationExecutor")
public void sendAuditWebhook(String payload) {
    // Runs in background; exceptions captured by AsyncUncaughtExceptionHandler
    webhookClient.post(payload);
}
```

#### Approach 2: Asynchronous Result with `CompletableFuture<T>`
```java
@Async("analyticalExecutor")
public CompletableFuture<CreditScore> calculateScoreAsync(Long customerId) {
    CreditScore score = creditBureauClient.fetchScore(customerId);
    return CompletableFuture.completedFuture(score);
}
```

#### Approach 3: Spring Boot 3.x Virtual Threads (Java 21 Project Loom)
```properties
# application.properties:
spring.threads.virtual.enabled=true
```
When enabled, `@Async` and Tomcat request threads automatically run on **lightweight virtual threads**, handling high-concurrency blocking I/O with zero complex reactive coding!

---

## 9. Caching Annotations

```java
@Service
public class CatalogCacheService {

    // 1. @Cacheable: Look in Redis/Caffeine cache first; execute method only on cache miss
    @Cacheable(cacheNames = "products", key = "#sku", unless = "#result == null")
    public Product findProductBySku(String sku) {
        return productRepository.findBySku(sku).orElse(null);
    }

    // 2. @CachePut: ALWAYS executes method and updates the cache with the new return value
    @CachePut(cacheNames = "products", key = "#product.sku")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // 3. @CacheEvict: Evicts specific cache entries or all entries (allEntries = true)
    @CacheEvict(cacheNames = "products", key = "#sku")
    public void deleteProduct(String sku) {
        productRepository.deleteBySku(sku);
    }
}
```

---

## 10. Method Security Approaches (Spring Boot 3.x / Security 6)

In Spring Boot 3.x, `@EnableGlobalMethodSecurity` is deprecated and replaced by `@EnableMethodSecurity`.

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true, // Enables @PreAuthorize and @PostAuthorize
    securedEnabled = true, // Enables @Secured
    jsr250Enabled = true   // Enables @RolesAllowed
)
public class MethodSecurityConfiguration {}
```

---

### 10.1 Approach 1: `@PreAuthorize` with SpEL (Recommended Enterprise Standard)
- **Scenario**: Fine-grained access control evaluating method arguments, user roles, permissions, or domain object ownership *before* method entry.

```java
@Service
public class DocumentManagementService {

    // Role check + Argument inspection via SpEL
    @PreAuthorize("hasRole('ADMIN') or #authorId == authentication.principal.id")
    public void deleteDocument(Long documentId, Long authorId) {
        documentRepository.deleteById(documentId);
    }
}
```

---

### 10.2 Approach 2: `@PostAuthorize` (Post-Execution Inspection)
- **Scenario**: Checking security against the **return object** after method execution.

```java
@PostAuthorize("returnObject.ownerUsername == authentication.name")
public MedicalRecord fetchRecord(Long recordId) {
    return recordRepository.findById(recordId).orElseThrow();
}
```

---

### 10.3 Approach 3: `@Secured` (Simple Role Checking)
- **Scenario**: Simple role-based checks without SpEL complexity. Requires `"ROLE_"` prefix.

```java
@Secured("ROLE_FINANCE_MANAGER")
public void approvePayroll(Long payrollId) { ... }
```

---

### 10.4 Approach 4: `@RolesAllowed` (JSR-250 Portability)
- **Scenario**: Standard Java EE applications maintaining compatibility across different framework containers.

```java
@RolesAllowed({"ADMIN", "OPERATOR"})
public void restartService() { ... }
```

---

### 10.5 Method Security Comparison

| Annotation | Expression Language (SpEL) | Inspects Arguments? | Inspects Return Value? | Java EE Standard? |
|---|---|---|---|---|
| **`@PreAuthorize`** | 🌟 **Full SpEL support** | ✅ Yes (`#arg`) | ❌ No | ❌ Spring-specific |
| **`@PostAuthorize`** | 🌟 **Full SpEL support** | ✅ Yes | 🌟 **Yes (`returnObject`)** | ❌ Spring-specific |
| **`@Secured`** | ❌ Strings only | ❌ No | ❌ No | ❌ Spring-specific |
| **`@RolesAllowed`** | ❌ Strings only | ❌ No | ❌ No | 🌟 **Yes (JSR-250)** |

---

## 11. Testing Slice Annotations

```
┌────────────────────────────────────────────────────────┐
│             Spring Boot Test Slice Hierarchy           │
├────────────────────────────────────────────────────────┤
│ @SpringBootTest                                        │
│ └── Full Context: Boots embedded server, all beans     │
├────────────────────────────────────────────────────────┤
│ @WebMvcTest                                            │
│ └── Web Layer Slice: DispatcherServlet, Filters, MockMvc│
├────────────────────────────────────────────────────────┤
│ @DataJpaTest                                           │
│ └── Data Slice: Entities, Repositories, In-Memory DB    │
├────────────────────────────────────────────────────────┤
│ @JsonTest                                              │
│ └── JSON Serialization Slice: JacksonTester, Mappers   │
└────────────────────────────────────────────────────────┘
```

---

## 12. Lifecycle & Conditional Annotations

### Conditional Annotations Quick Reference
- `@ConditionalOnClass({ DataSource.class, HikariDataSource.class })`: Class must be present on classpath.
- `@ConditionalOnMissingBean(ObjectMapper.class)`: Applies only if no user has declared this bean.
- `@ConditionalOnProperty(prefix = "feature.flag", name = "v2-checkout", havingValue = "true")`: Property must match value.

---

## 13. Validation Annotations (4 Approaches)

---

### 13.1 Approach 1: Payload DTO Validation (`@Valid @RequestBody`)
- **Scenario**: Validating JSON payloads received in `@PostMapping` or `@PutMapping`.

```java
public record UserRegistrationDto(
    @NotBlank(message = "Username cannot be empty")
    @Size(min = 3, max = 50)
    String username,

    @Email(message = "Must be a valid email format")
    String email,

    @Min(value = 18, message = "User must be at least 18 years old")
    int age
) {}
```

---

### 13.2 Approach 2: Parameter-Level Validation (`@Validated`)
- **Scenario**: Validating `@PathVariable` or `@RequestParam` directly on controller or service method signatures.

```java
@RestController
@RequestMapping("/api/v1/orders")
@Validated // Required at class level for method-level constraint evaluation
public class OrderQueryController {

    @GetMapping("/{orderId}")
    public Order getOrder(
            @PathVariable @Min(value = 1, message = "orderId must be positive") Long orderId) {
        return orderService.findById(orderId);
    }
}
```

---

### 13.3 Approach 3: Validation Groups (Create vs Update Rules)
- **Scenario**: Reusing a single DTO where some fields (like `id`) are `null` on Create, but mandatory on Update.

```java
public interface OnCreate {}
public interface OnUpdate {}

public class AccountDto {
    @Null(groups = OnCreate.class)
    @NotNull(groups = OnUpdate.class)
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String accountName;
}

// Controller usage:
@PostMapping
public void create(@Validated(OnCreate.class) @RequestBody AccountDto dto) { ... }

@PutMapping
public void update(@Validated(OnUpdate.class) @RequestBody AccountDto dto) { ... }
```

---

### 13.4 Approach 4: Custom Constraint Annotation (`ConstraintValidator`)
- **Scenario**: Enterprise domain validation that cannot be represented by simple regex or min/max (e.g. validating an IBAN or corporate tax ID).

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = TaxIdValidator.class)
public @interface ValidTaxId {
    String message() default "Invalid corporate tax identification number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class TaxIdValidator implements ConstraintValidator<ValidTaxId, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && value.matches("^[A-Z]{2}[0-9]{8}$");
    }
}
```

---

## 14. ⚠️ Interview Traps & Gotchas — The AOP Proxy Dilemma

### Solving Self-Invocation Proxy Bypass (5 Architectural Approaches)

When a method in a Spring-managed bean calls another annotated method in the **same class**, it bypasses the Spring CGLIB proxy (`this.method()`). The `@Transactional`, `@Async`, or `@Cacheable` annotation is **silently ignored**!

```
External Request ──► [Spring CGLIB Proxy] ──► RealBean.methodA()
                                                    │
                                                    └──► this.methodB() ❌ (Bypasses Proxy!)
```

---

#### Solution 1: Refactoring to a Separate Collaborator Service (Recommended Clean Architecture)
- **Scenario**: Standard production architecture adhering to the Single Responsibility Principle.
- **Benefits**: Eliminates proxy trickery completely; clean and testable.

```java
@Service
public class OrderService {
    private final OrderAuditService auditService; // Separate collaborator

    public OrderService(OrderAuditService auditService) {
        this.auditService = auditService;
    }

    public void processOrder(Order order) {
        // External call through the proxy: @Async executes properly!
        auditService.recordAuditAsync(order);
    }
}

@Service
public class OrderAuditService {
    @Async
    public void recordAuditAsync(Order order) {
        // Runs in separate background thread!
    }
}
```

---

#### Solution 2: Self-Injection via `ObjectProvider<Self>` or `@Lazy`
- **Scenario**: When extracting a separate class is not immediately practical.

```java
@Service
public class UserService {

    private final ObjectProvider<UserService> selfProvider;

    public UserService(ObjectProvider<UserService> selfProvider) {
        this.selfProvider = selfProvider;
    }

    public void registerUser(User user) {
        // Calls via the proxy instance retrieved from provider!
        selfProvider.getObject().sendActivationEmailAsync(user);
    }

    @Async
    public void sendActivationEmailAsync(User user) {
        // Executes asynchronously!
    }
}
```

---

#### Solution 3: Programmatic Execution via `TransactionTemplate` / `TaskExecutor`
- **Scenario**: When you need exact control over the execution boundary without annotation proxies.

```java
@Service
public class PaymentService {
    private final TransactionTemplate transactionTemplate;

    public PaymentService(TransactionTemplate transactionTemplate) {
        this.transactionTemplate = transactionTemplate;
    }

    public void executePayment() {
        // Programmatic transaction boundary: zero proxy required!
        transactionTemplate.executeWithoutResult(status -> {
            debitAccount();
            creditMerchant();
        });
    }
}
```

---

#### Solution 4: `AopContext.currentProxy()`
- **Scenario**: Legacy applications where `@EnableAspectJAutoProxy(exposeProxy = true)` is configured.
- **Trade-off**: Strongly couples code to Spring AOP internals; throws exception if proxy not exposed.

```java
@Service
public class LegacyReportService {

    public void generateAll() {
        ((LegacyReportService) AopContext.currentProxy()).generateSingleReport();
    }

    @Cacheable("reports")
    public Report generateSingleReport() { ... }
}
```

---

#### Solution 5: Compile-Time / Load-Time Bytecode Weaving with AspectJ
- **Scenario**: When private methods must be transactional or self-invocation proxy bypass must be permanently eliminated at the JVM bytecode level.
- **Benefits**: Modifies class bytecode directly (`.class` files), making transactions and aspects work even with `private` methods and `this.method()` calls.

---

## 15. 🔍 Deep Dive — Spring AOP CGLIB vs JDK Dynamic Proxies

| Feature | JDK Dynamic Proxy | CGLIB Subclass Proxy |
|---|---|---|
| **Mechanism** | Implements target interfaces using Java Reflection (`Proxy.newProxyInstance`) | Generates dynamic subclass bytecode extending target class |
| **Interface Required?** | 🌟 **Mandatory** (Target must implement an interface) | ❌ **No** (Proxies concrete classes directly) |
| **Final Methods / Classes**| Interfaces cannot have final methods | ❌ **Cannot override final classes or final methods** |
| **Spring Boot 3.x Default**| Fallback if `spring.aop.proxy-target-class=false` | 🌟 **Default for all Spring beans** |

---

## 16. 📝 Summary & Best Practice Rules

1. **AOP Proxies Only Intercept External Calls**: Always refactor same-class invocations to a collaborator or use self-injection.
2. **Never Annotate `private` Methods with `@Transactional` or `@Async`**: CGLIB cannot override private methods.
3. **Always Use `FetchType.LAZY` for `@ManyToOne`**: Eliminates unintended eager join storms and N+1 query traps.
4. **Prefer `@PreAuthorize` over `@Secured`**: Allows powerful SpEL expressions and parameter validation.
5. **Use `@RestController` for APIs**: Combines `@Controller` and `@ResponseBody` cleanly.

---

## 17. ❓ Practice Questions & Answers

### Q1: What happens if you mark a private method with `@Transactional`?
**Answer**: In standard Spring AOP (CGLIB or JDK Dynamic Proxies), the transaction will **silently fail to start**. CGLIB operates by generating a subclass of the bean and overriding public/protected methods to weave transaction interceptors. Because private methods cannot be overridden by subclasses, the proxy cannot intercept the call, and no transaction is created.

### Q2: Why does `@Repository` provide unique behavior compared to `@Service`?
**Answer**: While `@Service` is functionally identical to `@Component` and serves as an architectural semantic marker, `@Repository` triggers automatic persistence exception translation via Spring's `PersistenceExceptionTranslationPostProcessor`. It catches native database exceptions (`SQLException`, `HibernateException`) and re-throws them as unified Spring `DataAccessException`s.

### Q3: How do you solve the self-invocation proxy bypass when calling `@Async` within the same class?
**Answer**: The best approach is to refactor the asynchronous method into a separate collaborator service and inject it. Alternatively, inject `ObjectProvider<Self>` or a `@Lazy` reference to self, calling `selfProvider.getObject().asyncMethod()`, which routes the call through the proxy.

---

[🏠 Course Index](./00-course-index.md) | [← 01 — Fundamentals](./01-fundamentals.md) | **Module 02** | [Next: 03 — REST API →](./03-rest-api.md)
