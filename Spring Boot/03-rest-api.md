# 03 — REST API & Web Layer in Spring Boot

> A production-grade, interview-focused deep dive into how Spring Boot processes HTTP requests — from the moment a packet arrives at the JVM to the moment a JSON response leaves.

---

## Table of Contents
1. [REST Fundamentals in Spring Boot](#1-rest-fundamentals-in-spring-boot)
2. [Building REST APIs](#2-building-rest-apis)
3. [Filters vs Interceptors vs AOP](#3-filters-vs-interceptors-vs-aop)
4. [Filters In Depth](#4-filters-in-depth)
5. [HandlerInterceptor In Depth](#5-handlerinterceptor-in-depth)
6. [Request/Response DTOs](#6-requestresponse-dtos)
7. [Error Handling at the Web Layer](#7-error-handling-at-the-web-layer)
8. [CORS](#8-cors)
9. [Spring Boot Actuator](#9-spring-boot-actuator)
10. [Async REST](#10-async-rest)
11. [⚠️ Interview Traps & Gotchas](#11-️-interview-traps--gotchas)
12. [📝 Summary & Key Takeaways](#12--summary--key-takeaways)
13. [❓ Practice Questions](#13--practice-questions)

---

## 1. REST Fundamentals in Spring Boot

### 1.1 DispatcherServlet — The Front Controller

Spring MVC is built on the **Front Controller** pattern. A single servlet — `DispatcherServlet` — receives **every** HTTP request and delegates it to the appropriate handler.

```
Client HTTP Request
        │
        ▼
  [Servlet Container]  (Tomcat / Jetty / Undertow)
        │
        ▼
  DispatcherServlet.service()
        │
        ├─► HandlerMapping.getHandler()  → HandlerExecutionChain
        │
        ├─► HandlerAdapter.handle()      → calls your @Controller
        │
        ├─► ViewResolver (for MVC) OR HttpMessageConverter (for REST)
        │
        ▼
  HTTP Response written to OutputStream
```

In Spring Boot, `DispatcherServlet` is **auto-configured** by `DispatcherServletAutoConfiguration`. It is registered at path `"/"` by default, meaning it captures all requests.

```java
// What Spring Boot auto-configures for you (simplified)
@Bean
public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
    DispatcherServlet dispatcherServlet = new DispatcherServlet();
    dispatcherServlet.setThrowExceptionIfNoHandlerFound(
        webMvcProperties.isThrowExceptionIfNoHandlerFound()); // default: false
    return dispatcherServlet;
}

@Bean
public ServletRegistrationBean<DispatcherServlet> dispatcherServletRegistration(
        DispatcherServlet dispatcherServlet,
        WebMvcProperties webMvcProperties) {
    ServletRegistrationBean<DispatcherServlet> registration =
        new ServletRegistrationBean<>(dispatcherServlet,
            webMvcProperties.getServlet().getPath()); // default: "/"
    registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
    return registration;
}
```

#### The `DispatcherServlet.doDispatch()` Processing Pipeline

The core logic lives in `doDispatch()`. Here is the annotated sequence:

```
doDispatch(request, response):

 1. checkMultipart(request)
    └─ Wraps multipart requests in MultipartHttpServletRequest

 2. getHandler(processedRequest)
    └─ Iterates HandlerMappings to find HandlerExecutionChain
       (contains the handler + list of HandlerInterceptors)

 3. getHandlerAdapter(handler)
    └─ Finds an adapter that can invoke this handler type
       (RequestMappingHandlerAdapter for @Controller methods)

 4. applyPreHandle(interceptors)
    └─ Calls interceptor.preHandle() for each interceptor
       ← Returns false → request processing STOPS

 5. ha.handle(request, response, handler)
    └─ The actual controller method is invoked
       - Argument resolvers convert HTTP params → method args
       - Return value handlers convert return value → response

 6. applyPostHandle(interceptors)
    └─ Calls interceptor.postHandle() in REVERSE order

 7. processDispatchResult(exception or mv)
    └─ Render ModelAndView OR propagate exception to HandlerExceptionResolvers

 8. triggerAfterCompletion(interceptors)
    └─ Always called (even on exception)
```

> [!IMPORTANT]
> `DispatcherServlet` is a **Spring-managed bean** itself — it lives in the **WebApplicationContext** (child context). It is initialized lazily by default (on first request) but can be eagerly initialized via `spring.mvc.servlet.load-on-startup=1`.

---

### 1.2 Handler Mapping — How Spring Finds Your Controller

`HandlerMapping` is an interface that maps an incoming request to a handler object (usually a `HandlerMethod`).

Spring Boot registers several `HandlerMapping` implementations, ordered by priority:

| Order | Class | Handles |
|-------|-------|---------|
| 0 | `RequestMappingHandlerMapping` | `@RequestMapping` methods |
| 1 | `BeanNameUrlHandlerMapping` | Beans named with `/` prefix |
| 2 | `RouterFunctionMapping` | WebFlux `RouterFunction` (only in reactive) |
| `Integer.MAX_VALUE - 1` | `SimpleUrlHandlerMapping` | Static resources via `/**` |
| `Integer.MAX_VALUE` | `WelcomePageHandlerMapping` | `index.html` fallback |

`RequestMappingHandlerMapping` builds an internal registry during startup:

```java
// Simplified internal registration
// For every @Controller bean, it scans methods annotated with @RequestMapping
Map<RequestMappingInfo, HandlerMethod> handlerMethods;

// Lookup at request time:
// Finds ALL matching mappings, then picks the BEST match using specificity rules:
// 1. Exact path wins over pattern
// 2. More specific HTTP method wins
// 3. More specific media type wins (consumes/produces)
```

#### Path Matching: AntPathMatcher vs PathPatternParser

Since Spring Boot 2.6, the default path matcher is **`PathPatternParser`** (not the old `AntPathMatcher`). It is significantly faster because patterns are pre-compiled.

```yaml
# application.yml
spring:
  mvc:
    pathmatch:
      matching-strategy: path-pattern-parser  # default in Boot 2.6+
      # use-suffix-pattern: false              # suffix patterns (.json) deprecated
```

---

### 1.3 Request/Response Lifecycle — Step by Step

Let's trace a `POST /api/v1/users` request end-to-end:

```
1. TCP packet arrives at OS socket
2. Embedded Tomcat accepts connection, reads HTTP request bytes
3. Tomcat creates HttpServletRequest / HttpServletResponse objects
4. Tomcat hands request to the Servlet Filter Chain (your filters first)
5. After filters, request reaches DispatcherServlet
6. DispatcherServlet finds HandlerExecutionChain (controller method + interceptors)
7. HandlerInterceptor.preHandle() called
8. RequestMappingHandlerAdapter resolves method arguments:
   - @RequestBody → HttpMessageConverter reads JSON → Jackson deserializes → User DTO
   - @PathVariable → extracted from URI
   - @RequestParam → from query string
   - @RequestHeader → from HTTP headers
   - Principal → from SecurityContext
9. Bean Validation runs (@Valid triggers MethodValidationInterceptor)
10. Your controller method executes
11. Return value processed:
    - ResponseEntity<T> → status + headers + body
    - Body serialized by HttpMessageConverter (Jackson → JSON)
12. HandlerInterceptor.postHandle() called
13. Response committed to client
14. HandlerInterceptor.afterCompletion() called
15. Servlet Filter afterCompletion chain unwinds
16. Tomcat sends HTTP response bytes over TCP
```

---

## 2. Building REST APIs

### 2.1 @RestController Deep Dive

`@RestController` is a **composed annotation**:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller         // registers bean as a Spring component
@ResponseBody       // ALL methods implicitly have @ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```

The `@ResponseBody` annotation tells `RequestResponseBodyMethodProcessor` to use an `HttpMessageConverter` to write the return value directly to the response body, bypassing the `ViewResolver`.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) { // constructor injection — preferred
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

> [!TIP]
> Always use **constructor injection** in controllers. It makes the controller easier to unit test without Spring context.

---

### 2.2 HTTP Methods and Idempotency

| Method | Idempotent | Safe | Typical Use |
|--------|-----------|------|-------------|
| GET | ✅ Yes | ✅ Yes | Fetch resource |
| HEAD | ✅ Yes | ✅ Yes | Fetch headers only |
| OPTIONS | ✅ Yes | ✅ Yes | Discover allowed methods |
| PUT | ✅ Yes | ❌ No | Full replace of resource |
| DELETE | ✅ Yes | ❌ No | Delete resource |
| POST | ❌ No | ❌ No | Create resource |
| PATCH | ❌ No* | ❌ No | Partial update |

> [!WARNING]
> PATCH idempotency depends on the implementation. A PATCH that says "set field X to value Y" is idempotent. A PATCH that says "increment field X by 1" is NOT idempotent. This nuance is a common interview question.

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @PostMapping                           // Create — not idempotent
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@Valid @RequestBody CreateProductRequest req) { ... }

    @GetMapping("/{id}")                   // Read — safe + idempotent
    public ProductResponse get(@PathVariable Long id) { ... }

    @PutMapping("/{id}")                   // Full replace — idempotent
    public ProductResponse replace(@PathVariable Long id,
                                   @Valid @RequestBody ReplaceProductRequest req) { ... }

    @PatchMapping("/{id}")                 // Partial update — conditionally idempotent
    public ProductResponse update(@PathVariable Long id,
                                  @RequestBody JsonPatch patch) { ... }

    @DeleteMapping("/{id}")                // Delete — idempotent
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { ... }
}
```

---

### 2.3 @RequestMapping Patterns, Path Variables & Regex

```java
// Basic patterns
@GetMapping("/users")               // exact match
@GetMapping("/users/{id}")          // path variable
@GetMapping("/users/{id}/orders")   // nested resource

// Multiple paths
@GetMapping({"/users", "/accounts"})

// Wildcards (AntPathMatcher / PathPatternParser)
@GetMapping("/files/**")            // matches /files/a/b/c
@GetMapping("/images/*.png")        // matches /images/logo.png

// Regex constraints on path variables
@GetMapping("/users/{id:[0-9]+}")   // id must be numeric
@GetMapping("/items/{category:[a-z-]+}/{id:\\d+}") // compound regex

// Optional trailing slash (Spring Boot 6.x REMOVED trailing slash matching)
// Must now explicitly register both paths or configure matcher
```

```java
// Accessing path variables
@GetMapping("/users/{userId}/orders/{orderId}")
public ResponseEntity<OrderResponse> getOrder(
        @PathVariable Long userId,
        @PathVariable("orderId") Long orderId) { ... }

// Multiple path variables into a map
@GetMapping("/resources/{*path}")
public ResponseEntity<String> getResource(@PathVariable Map<String, String> vars) {
    return ResponseEntity.ok(vars.get("path"));
}

// Matrix variables (;key=value in URL — disabled by default)
// Enable via: WebMvcConfigurer.configurePathMatch()
@GetMapping("/cars/{brand}")
public ResponseEntity<List<CarResponse>> getCars(
        @PathVariable String brand,
        @MatrixVariable(pathVar = "brand") int year) { ... }
```

```java
// Request parameters
@GetMapping("/search")
public Page<ProductResponse> search(
        @RequestParam String keyword,
        @RequestParam(required = false, defaultValue = "0") int page,
        @RequestParam(required = false, defaultValue = "20") int size,
        @RequestParam(required = false) List<String> tags) { ... }
// GET /search?keyword=shoes&page=1&tags=sport&tags=casual
```

---

### 2.4 Request Body Validation with @Valid

Spring Boot integrates with **Jakarta Bean Validation** (Hibernate Validator as the default implementation).

```java
// DTO with validation annotations
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    String name,

    @NotBlank
    @Email(message = "Invalid email format")
    String email,

    @NotNull
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120)
    Integer age,

    @NotNull
    @Valid                              // cascade validation into nested objects
    AddressRequest address
) {}

public record AddressRequest(
    @NotBlank String street,
    @NotBlank @Size(min = 5, max = 10) String zipCode
) {}
```

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
    // If validation fails → MethodArgumentNotValidException is thrown
    // Default handler returns 400 Bad Request with field errors
    return userService.create(request);
}
```

```java
// Custom validator
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneNumberValidator.class)
public @interface ValidPhone {
    String message() default "Invalid phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneNumberValidator implements ConstraintValidator<ValidPhone, String> {
    private static final Pattern PHONE_PATTERN =
        Pattern.compile("^\\+?[1-9]\\d{1,14}$"); // E.164 format

    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true; // use @NotNull for null check
        return PHONE_PATTERN.matcher(value).matches();
    }
}
```

```java
// Validating @PathVariable and @RequestParam
// Must add @Validated at class level — uses Spring's MethodValidationPostProcessor
@RestController
@RequestMapping("/api/v1/users")
@Validated
public class UserController {

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable @Positive Long id) {
        // if id <= 0 → ConstraintViolationException (not MethodArgumentNotValidException!)
        return userService.findById(id);
    }
}
```

> [!IMPORTANT]
> `@Valid` on `@RequestBody` throws `MethodArgumentNotValidException`.
> `@Validated` + constraint on `@PathVariable`/`@RequestParam` throws `ConstraintViolationException`.
> These are **different exceptions** and need different handler methods.

---

### 2.5 ResponseEntity Patterns

`ResponseEntity<T>` gives you full control over status code, headers, and body:

```java
// Convenience factory methods
ResponseEntity.ok(body)                              // 200 OK
ResponseEntity.created(URI.create("/users/1")).body(dto) // 201 Created + Location header
ResponseEntity.accepted().build()                    // 202 Accepted, no body
ResponseEntity.noContent().build()                   // 204 No Content
ResponseEntity.badRequest().body(errorDto)           // 400
ResponseEntity.notFound().build()                    // 404, no body
ResponseEntity.status(HttpStatus.CONFLICT).body(dto) // 409

// Builder style — most flexible
return ResponseEntity
    .status(HttpStatus.CREATED)
    .header("X-Resource-Id", String.valueOf(user.getId()))
    .header(HttpHeaders.LOCATION, "/api/v1/users/" + user.getId())
    .contentType(MediaType.APPLICATION_JSON)
    .body(userResponse);

// Returning Optional
return userService.findById(id)
    .map(ResponseEntity::ok)
    .orElseGet(() -> ResponseEntity.notFound().build());

// Returning void with 204
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
}
```

---

### 2.6 HTTP Status Codes — When to Use Which

| Code | Meaning | When to Return |
|------|---------|----------------|
| 200 OK | Success | GET, PUT, PATCH that returns body |
| 201 Created | Resource created | POST that creates a resource |
| 202 Accepted | Async processing started | Long-running tasks |
| 204 No Content | Success, no body | DELETE, PUT/PATCH with no response body |
| 400 Bad Request | Client error in request | Validation failures, malformed JSON |
| 401 Unauthorized | Not authenticated | Missing/invalid auth token |
| 403 Forbidden | Authenticated but not authorized | Insufficient permissions |
| 404 Not Found | Resource doesn't exist | ID not found in DB |
| 405 Method Not Allowed | HTTP method not supported | POST on a GET-only endpoint |
| 409 Conflict | State conflict | Duplicate email, optimistic lock conflict |
| 410 Gone | Permanently deleted | Resource was deleted (vs 404 which means never existed) |
| 422 Unprocessable Entity | Semantic validation failed | Business rule violations |
| 429 Too Many Requests | Rate limit exceeded | Throttling |
| 500 Internal Server Error | Server bug | Unexpected exception |
| 503 Service Unavailable | Overloaded or maintenance | Circuit breaker open |

---

### 2.7 Content Negotiation

Spring MVC can serve different representations of the same resource based on the client's `Accept` header:

```java
// application/json AND application/xml from the same endpoint
@GetMapping(value = "/users/{id}",
    produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
public UserResponse getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

```xml
<!-- pom.xml — add Jackson XML support -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

```java
// Content negotiation configuration (prefer header-based, not URL-based)
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(false)          // no ?format=json
            .favorPathExtension(false)       // no /users.json (deprecated)
            .ignoreAcceptHeader(false)       // use Accept header (default)
            .defaultContentType(MediaType.APPLICATION_JSON);
    }
}
```

---

### 2.8 API Versioning Strategies

There are four common versioning approaches — each with tradeoffs:

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URI Path | `/api/v1/users` | Simple, cacheable, visible | URI changes, violates REST (URI = resource) |
| Request Parameter | `/api/users?version=1` | No URI change | Messy, easy to miss |
| Custom Header | `X-API-Version: 1` | Clean URIs | Not visible in browser, harder to test |
| Accept Header (Media Type) | `Accept: application/vnd.myapp.v1+json` | Proper REST (content negotiation) | Complex, verbose |

#### URI Versioning (Most Common in Practice)

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserResponseV1 getUser(@PathVariable Long id) { ... }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable Long id) { ... }
}
```

#### Header-Based Versioning

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(value = "/{id}", headers = "X-API-Version=1")
    public UserResponseV1 getUserV1(@PathVariable Long id) { ... }

    @GetMapping(value = "/{id}", headers = "X-API-Version=2")
    public UserResponseV2 getUserV2(@PathVariable Long id) { ... }
}
```

#### Media Type Versioning

```java
@GetMapping(value = "/{id}", produces = "application/vnd.myapp.users.v1+json")
public UserResponseV1 getUserV1(@PathVariable Long id) { ... }

@GetMapping(value = "/{id}", produces = "application/vnd.myapp.users.v2+json")
public UserResponseV2 getUserV2(@PathVariable Long id) { ... }
```

> [!TIP]
> For most teams, **URI versioning** is the pragmatic winner. It's immediately visible, easy to document with Swagger/OpenAPI, and straightforward to route via API gateways.

---

## 3. Filters vs Interceptors vs AOP

This is one of the most frequently asked interview topics. Understanding the execution layer of each is critical.

### 3.1 Conceptual Overview

```
Incoming Request
      │
      ▼
[Servlet Filter Chain]          ← javax/jakarta.servlet.Filter
      │   (before Servlet)
      ▼
[DispatcherServlet]
      │
      ▼
[HandlerInterceptor.preHandle]  ← Spring MVC layer (has access to HandlerMethod)
      │
      ▼
[@Around AOP Advice]            ← Spring AOP (wraps the bean method call)
      │
      ▼
[@Before AOP Advice]
      │
      ▼
[Controller Method]
      │
      ▼
[@After / @AfterReturning AOP]
      │
      ▼
[HandlerInterceptor.postHandle]
      │
      ▼
[HandlerInterceptor.afterCompletion]
      │
      ▼
[Servlet Filter afterCompletion] ← doFilter() continues
      │
      ▼
Outgoing Response
```

### 3.2 Detailed Comparison Table

| Feature | Servlet Filter | HandlerInterceptor | Spring AOP |
|---------|---------------|-------------------|------------|
| **Layer** | Servlet container | Spring MVC | Spring proxy |
| **Spec** | Servlet API (Jakarta EE) | Spring Framework | Spring Framework |
| **Access to Handler** | ❌ No | ✅ Yes (HandlerMethod) | ✅ Yes (ProceedingJoinPoint) |
| **Access to Model** | ❌ No | ✅ Yes (postHandle) | ❌ No |
| **Works with non-Spring requests** | ✅ Yes | ❌ No | ❌ No |
| **Can stop request processing** | ✅ Yes | ✅ Yes (preHandle) | ✅ Yes (don't call proceed()) |
| **Works with @ExceptionHandler** | ❌ No | ✅ Yes | ✅ Yes |
| **Can modify request body** | ✅ Yes (wrap stream) | ⚠️ Difficult (stream consumed) | ❌ No |
| **Can modify response** | ✅ Yes (wrap stream) | ✅ postHandle | ✅ AfterReturning |
| **Best for** | Auth, logging, CORS, encoding | Request metrics, locale, auth checks | Business logic cross-cutting |
| **Config location** | FilterRegistrationBean / @WebFilter | WebMvcConfigurer | @Aspect bean |

> [!NOTE]
> Filters run **outside** the Spring context — they cannot access Spring beans directly unless retrieved from `ApplicationContext`. However, `OncePerRequestFilter` (which extends `GenericFilterBean`) **can** have Spring beans injected because it's a Spring-managed bean itself.

---

## 4. Filters In Depth

### 4.1 Implementing a Filter

```java
// Option 1: Implement jakarta.servlet.Filter
@Component
@Order(1)  // lower number = earlier in chain
public class RequestLoggingFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        long startTime = System.currentTimeMillis();

        log.info("Incoming request: {} {}", httpRequest.getMethod(), httpRequest.getRequestURI());

        chain.doFilter(request, response);  // MUST call this to continue

        long duration = System.currentTimeMillis() - startTime;
        log.info("Completed in {}ms with status {}",
            duration, ((HttpServletResponse) response).getStatus());
    }
}
```

### 4.2 OncePerRequestFilter — The Spring Way

`OncePerRequestFilter` guarantees the filter executes exactly once per request, even if the request is dispatched internally (e.g., forward/include):

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String CORRELATION_ID_MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String correlationId = request.getHeader(CORRELATION_ID_HEADER);

        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        // Put in MDC so all log lines in this request include the ID
        MDC.put(CORRELATION_ID_MDC_KEY, correlationId);

        // Echo back in response header
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_MDC_KEY); // CRITICAL: clean up MDC (thread pool reuse)
        }
    }
}
```

> [!WARNING]
> Always clean up `MDC` in a `finally` block. Tomcat uses a thread pool — if you don't clean up, the next request on the same thread will inherit the previous request's correlation ID.

### 4.3 Registering Filters with FilterRegistrationBean

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<RequestLoggingFilter> loggingFilter() {
        FilterRegistrationBean<RequestLoggingFilter> registrationBean =
            new FilterRegistrationBean<>();
        registrationBean.setFilter(new RequestLoggingFilter());
        registrationBean.addUrlPatterns("/api/*");       // only for API paths
        registrationBean.setOrder(2);
        registrationBean.setName("requestLoggingFilter");
        return registrationBean;
    }
}
```

> [!TIP]
> Use `FilterRegistrationBean` when you need URL pattern filtering or ordering control. Use `@Component` + `@Order` for simple global filters.

### 4.4 Production-Grade Logging Filter

```java
@Component
@Order(2)
public class RequestResponseLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);
    private static final int MAX_PAYLOAD_LENGTH = 10_000;

    // These paths are too noisy or contain sensitive data
    private static final Set<String> EXCLUDED_PATHS = Set.of(
        "/actuator/health", "/actuator/prometheus"
    );

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return EXCLUDED_PATHS.contains(request.getServletPath());
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        // Wrap request to allow body re-reading (InputStream can only be read once!)
        ContentCachingRequestWrapper wrappedRequest =
            new ContentCachingRequestWrapper(request, MAX_PAYLOAD_LENGTH);
        ContentCachingResponseWrapper wrappedResponse =
            new ContentCachingResponseWrapper(response);

        long start = System.currentTimeMillis();
        try {
            chain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            logRequest(wrappedRequest);
            logResponse(wrappedResponse, System.currentTimeMillis() - start);
            // CRITICAL: copy cached response body back to actual response
            wrappedResponse.copyBodyToResponse();
        }
    }

    private void logRequest(ContentCachingRequestWrapper request) {
        String body = new String(request.getContentAsByteArray(),
            StandardCharsets.UTF_8);
        log.info("[REQUEST] {} {} | Body: {}",
            request.getMethod(), request.getRequestURI(),
            body.isBlank() ? "<empty>" : body);
    }

    private void logResponse(ContentCachingResponseWrapper response, long durationMs) {
        log.info("[RESPONSE] Status: {} | Duration: {}ms", response.getStatus(), durationMs);
    }
}
```

> [!WARNING]
> `ContentCachingRequestWrapper` buffers the entire request body in memory. For large file uploads or streaming APIs, this can cause `OutOfMemoryError`. Always set a size limit or skip this filter for multipart endpoints.

### 4.5 Filter Ordering in Spring Security Context

When Spring Security is on the classpath, it adds a `FilterChainProxy` (at `SecurityProperties.DEFAULT_FILTER_ORDER = -100`). This means security runs **before** most application filters:

```
Order -100: SecurityContextPersistenceFilter  (Spring Security's FilterChainProxy)
Order -100: UsernamePasswordAuthenticationFilter
Order -100: BearerTokenAuthenticationFilter
... other Spring Security filters ...
Order 1:    CorrelationIdFilter (your filter)
Order 2:    RequestLoggingFilter (your filter)
              ↓
         DispatcherServlet
```

```java
// To run BEFORE Spring Security:
@Bean
public FilterRegistrationBean<MyFilter> myFilter() {
    FilterRegistrationBean<MyFilter> reg = new FilterRegistrationBean<>(new MyFilter());
    reg.setOrder(SecurityProperties.DEFAULT_FILTER_ORDER - 1); // before security
    return reg;
}
```

---

## 5. HandlerInterceptor In Depth

### 5.1 The Three Phases

```java
public interface HandlerInterceptor {

    // Called BEFORE the handler method
    // Return true to continue, false to stop (you must handle response yourself)
    default boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws Exception {
        return true;
    }

    // Called AFTER handler method, BEFORE view rendering
    // NOT called if preHandle returned false OR if an exception was thrown
    default void postHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler,
                            @Nullable ModelAndView modelAndView) throws Exception {}

    // Called AFTER request completion (view rendered or exception handled)
    // ALWAYS called if preHandle returned true (even if exception occurred)
    default void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler,
                                 @Nullable Exception ex) throws Exception {}
}
```

> [!IMPORTANT]
> `postHandle` is **NOT called** if:
> 1. An exception is thrown from the handler
> 2. `preHandle` returned `false`
> 
> `afterCompletion` IS called as long as `preHandle` returned `true`, regardless of exceptions.

### 5.2 Authentication Check Interceptor

```java
@Component
public class ApiKeyInterceptor implements HandlerInterceptor {

    private static final String API_KEY_HEADER = "X-API-Key";

    @Autowired
    private ApiKeyService apiKeyService;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {

        // Skip for non-controller handlers (e.g., static resources)
        if (!(handler instanceof HandlerMethod handlerMethod)) {
            return true;
        }

        // Check if the method/class has a @Public annotation (no auth required)
        if (handlerMethod.hasMethodAnnotation(Public.class) ||
            handlerMethod.getBeanType().isAnnotationPresent(Public.class)) {
            return true;
        }

        String apiKey = request.getHeader(API_KEY_HEADER);
        if (apiKey == null || !apiKeyService.isValid(apiKey)) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {"error": "Invalid or missing API key"}
                """);
            return false;  // Stop processing
        }

        // Store validated key for use in controller
        request.setAttribute("apiKey", apiKeyService.load(apiKey));
        return true;
    }
}
```

### 5.3 Performance Metrics Interceptor

```java
@Component
public class MetricsInterceptor implements HandlerInterceptor {

    private static final String START_TIME_ATTR = "requestStartTime";
    private final MeterRegistry meterRegistry;

    public MetricsInterceptor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        request.setAttribute(START_TIME_ATTR, System.nanoTime());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) {
        Long startTime = (Long) request.getAttribute(START_TIME_ATTR);
        if (startTime == null) return;

        long durationNs = System.nanoTime() - startTime;

        String handlerName = (handler instanceof HandlerMethod hm)
            ? hm.getBeanType().getSimpleName() + "." + hm.getMethod().getName()
            : "unknown";

        meterRegistry.timer("http.server.requests.custom",
                "handler", handlerName,
                "status", String.valueOf(response.getStatus()),
                "method", request.getMethod())
            .record(durationNs, TimeUnit.NANOSECONDS);
    }
}
```

### 5.4 Registering Interceptors

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired private ApiKeyInterceptor apiKeyInterceptor;
    @Autowired private MetricsInterceptor metricsInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(apiKeyInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/public/**", "/actuator/**");

        registry.addInterceptor(metricsInterceptor)
            .addPathPatterns("/**");
        // Interceptors execute in registration order
    }
}
```

### 5.5 Can an Interceptor Read the Request Body?

This is a classic interview trap. The answer is: **usually not, without wrapping**.

The `HttpServletRequest` body is an `InputStream` that can be read **only once**. By the time `preHandle` is called, the body has NOT yet been read (Jackson reads it during argument resolution, which happens AFTER preHandle). So `preHandle` CAN theoretically read the body.

However, if you read it in `preHandle`, Jackson can't read it again in the controller. You must wrap the request:

```java
// In a Filter (not interceptor!) — wrap BEFORE the interceptor
ContentCachingRequestWrapper wrappedRequest =
    new ContentCachingRequestWrapper(request);

// In the interceptor's preHandle, after doFilter, body is cached:
byte[] body = ((ContentCachingRequestWrapper) request).getContentAsByteArray();
```

The correct production pattern is to use `ContentCachingRequestWrapper` in a **Filter** and then access the cached bytes in the interceptor.

---

## 6. Request/Response DTOs

### 6.1 The DTO Pattern

Never expose your JPA entity directly in your API. Use dedicated DTO classes:

```java
// Entity — internal domain model
@Entity
@Table(name = "users")
public class UserEntity {
    @Id @GeneratedValue private Long id;
    private String name;
    private String email;
    @Column(name = "password_hash") private String passwordHash;  // NEVER expose this!
    private LocalDateTime createdAt;
    private boolean deleted;
}

// Request DTO — what the client sends
public record CreateUserRequest(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8) String password
) {}

// Response DTO — what we send back
public record UserResponse(
    Long id,
    String name,
    String email,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'")
    LocalDateTime createdAt
) {}
```

### 6.2 MapStruct — Zero-Overhead Mapping

MapStruct generates mapping code at compile time (unlike ModelMapper which uses reflection at runtime):

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.5.5.Final</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

```java
@Mapper(componentModel = "spring")  // makes it a Spring @Component
public interface UserMapper {

    // Simple field mapping (same name = automatic)
    UserResponse toResponse(UserEntity entity);

    // Custom field mapping
    @Mapping(source = "passwordHash", target = "password", ignore = true) // ignore sensitive field
    @Mapping(source = "name", target = "displayName")
    UserSummary toSummary(UserEntity entity);

    // Collection mapping (automatically derived from single mapping)
    List<UserResponse> toResponseList(List<UserEntity> entities);

    // Reverse mapping
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    @Mapping(source = "password", target = "passwordHash",
             qualifiedByName = "hashPassword")
    UserEntity toEntity(CreateUserRequest request);

    @Named("hashPassword")
    default String hashPassword(String rawPassword) {
        return BCrypt.hashpw(rawPassword, BCrypt.gensalt());
    }
}
```

| Feature | ModelMapper | MapStruct |
|---------|-------------|-----------|
| Performance | Reflection at runtime | Generated code at compile time |
| Type safety | ❌ Runtime errors | ✅ Compile-time errors |
| Configuration | Programmatic | Annotation-based |
| Learning curve | Lower | Slightly higher |
| Spring Boot integration | Manual | `componentModel = "spring"` |
| Recommended | Legacy codebases | ✅ New projects |

### 6.3 API Response Wrapper Pattern

Wrapping all responses in a consistent envelope improves API usability:

```java
// Generic API response wrapper
public record ApiResponse<T>(
    boolean success,
    T data,
    String message,
    List<ApiError> errors,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'")
    LocalDateTime timestamp
) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, null, null, LocalDateTime.now());
    }

    public static <T> ApiResponse<T> success(T data, String message) {
        return new ApiResponse<>(true, data, message, null, LocalDateTime.now());
    }

    public static <T> ApiResponse<T> error(String message, List<ApiError> errors) {
        return new ApiResponse<>(false, null, message, errors, LocalDateTime.now());
    }
}

public record ApiError(String field, String message, Object rejectedValue) {}
```

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @PostMapping
    public ResponseEntity<ApiResponse<UserResponse>> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse user = userService.create(request);
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(ApiResponse.success(user, "User created successfully"));
    }
}

// Response:
// {
//   "success": true,
//   "data": { "id": 1, "name": "Alice", ... },
//   "message": "User created successfully",
//   "errors": null,
//   "timestamp": "2024-01-15T10:30:00Z"
// }
```

> [!NOTE]
> Some API design philosophies (e.g., JSON:API, HAL) prefer no wrapper and use HTTP status codes + `Link` headers. The wrapper pattern is more common in internal/B2B APIs where clients need consistent parsing logic.

---

## 7. Error Handling at the Web Layer

### 7.1 Spring Boot's Default Error Handling

Spring Boot auto-configures `BasicErrorController` which handles requests to `/error`. When an unhandled exception propagates:

1. `DispatcherServlet` → `HandlerExceptionResolverComposite` → no resolver handles it
2. Exception is re-thrown as a servlet error
3. Tomcat forwards the request to `/error`
4. `BasicErrorController.error()` returns a default JSON error response

```json
{
  "timestamp": "2024-01-15T10:30:00.000+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/api/v1/users"
}
```

### 7.2 Custom /error Endpoint

```java
@RestController
public class CustomErrorController implements ErrorController {

    @Autowired
    private ErrorAttributes errorAttributes;

    @RequestMapping("/error")
    public ResponseEntity<ApiResponse<Void>> handleError(HttpServletRequest request) {
        WebRequest webRequest = new ServletWebRequest(request);
        Map<String, Object> attrs = errorAttributes.getErrorAttributes(webRequest,
            ErrorAttributeOptions.of(ErrorAttributeOptions.Include.MESSAGE,
                                     ErrorAttributeOptions.Include.BINDING_ERRORS));

        int status = (Integer) attrs.getOrDefault("status", 500);
        String message = (String) attrs.getOrDefault("message", "An error occurred");

        return ResponseEntity
            .status(status)
            .body(ApiResponse.error(message, null));
    }
}
```

> [!TIP]
> For comprehensive error handling, see **Module 04 — Exception Handling**, which covers `@ControllerAdvice`, `@ExceptionHandler`, `ProblemDetail` (RFC 7807), and global exception strategies in depth.

---

## 8. CORS

### 8.1 What is CORS?

**Cross-Origin Resource Sharing** is a browser security mechanism that restricts web pages from making requests to a **different origin** (protocol + host + port) than the one that served the page.

```
Frontend: http://localhost:3000        (Origin A)
Backend:  http://localhost:8080        (Origin B — different port!)

Browser BLOCKS the API call unless the backend responds with:
Access-Control-Allow-Origin: http://localhost:3000
```

The browser sends a **preflight `OPTIONS` request** before the actual request for:
- Non-simple methods (not GET/POST/HEAD)
- Custom headers
- Non-standard content types

```
OPTIONS /api/v1/users HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization

HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600        ← browser caches preflight for 1 hour
```

### 8.2 @CrossOrigin Annotation

```java
// On a single controller
@RestController
@RequestMapping("/api/v1/users")
@CrossOrigin(
    origins = {"http://localhost:3000", "https://myapp.com"},
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowedHeaders = {"Content-Type", "Authorization"},
    exposedHeaders = {"X-Correlation-ID"},  // headers client can access
    allowCredentials = "true",
    maxAge = 3600
)
public class UserController { ... }

// On a single method
@GetMapping("/{id}")
@CrossOrigin(origins = "https://trusted-partner.com")
public UserResponse getUser(@PathVariable Long id) { ... }
```

### 8.3 Global CORS Configuration

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",
                "https://staging.myapp.com",
                "https://myapp.com"
            )
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("Content-Type", "Authorization", "X-Requested-With",
                            "X-Correlation-ID")
            .exposedHeaders("X-Correlation-ID", "X-Total-Count")
            .allowCredentials(true)
            .maxAge(3600);

        // Different rules for public endpoints
        registry.addMapping("/public/**")
            .allowedOriginPatterns("*")
            .allowedMethods("GET")
            .allowCredentials(false);
    }
}
```

### 8.4 CORS with Spring Security

When Spring Security is present, it has its own CORS processing. You must configure CORS at the **Spring Security level**, not just MVC level:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // CRITICAL: configure CORS before csrf and authorization
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:3000", "https://myapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setExposedHeaders(List.of("X-Correlation-ID"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

> [!IMPORTANT]
> When Spring Security is active, the `CorsFilter` in Spring Security runs **before** the `@CrossOrigin` / `WebMvcConfigurer` CORS handling. If you configure CORS only in `WebMvcConfigurer` but have Spring Security, the preflight `OPTIONS` request will be **rejected by security** before it reaches the MVC CORS handler. Always configure CORS at the `HttpSecurity` level when using Spring Security.

### 8.5 Common CORS Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `allowedOrigins("*")` + `allowCredentials(true)` | Browser rejects this — invalid combo | Use `allowedOriginPatterns("*")` or specific origins |
| Only configuring MVC CORS with Spring Security active | Preflight blocked by security filter | Configure CORS in `HttpSecurity` |
| Forgetting to expose custom response headers | Client JS can't read the header | Add to `exposedHeaders` |
| Not allowing `OPTIONS` method | Preflight fails | Include `OPTIONS` in allowed methods |
| Wildcard `allowedHeaders("*")` with credentials | May cause issues in some browsers | List specific required headers |

---

## 9. Spring Boot Actuator

### 9.1 Overview and Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      base-path: /actuator          # default
      exposure:
        include: "health,info,metrics,prometheus,loggers,env,beans"
        # In production, be selective! Never expose: env, beans, heapdump in production
  endpoint:
    health:
      show-details: when-authorized  # or always / never
      show-components: always
    prometheus:
      enabled: true
  server:
    port: 9090  # separate management port — best practice for production
```

### 9.2 Key Actuator Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/actuator/health` | GET | Application health status |
| `/actuator/info` | GET | Application info (version, build) |
| `/actuator/metrics` | GET | Available metrics |
| `/actuator/metrics/{name}` | GET | Specific metric value |
| `/actuator/prometheus` | GET | Prometheus-format metrics |
| `/actuator/loggers` | GET | Logger levels |
| `/actuator/loggers/{name}` | POST | Change log level at runtime |
| `/actuator/env` | GET | Environment properties (⚠️ sensitive) |
| `/actuator/beans` | GET | All Spring beans (⚠️ sensitive) |
| `/actuator/heapdump` | GET | JVM heap dump (⚠️ very sensitive) |
| `/actuator/threaddump` | GET | Thread dump |
| `/actuator/shutdown` | POST | Graceful shutdown (disabled by default!) |

### 9.3 Securing Actuator Endpoints

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {

    @Bean
    @Order(1) // Apply before the main security config
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                .requestMatchers(EndpointRequest.toAnyEndpoint())
                    .hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

### 9.4 Custom Health Indicators

```java
@Component
public class DatabaseConnectionHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    public DatabaseConnectionHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            boolean valid = conn.isValid(2); // 2-second timeout
            if (valid) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("maxPoolSize",
                        ((HikariDataSource) dataSource).getMaximumPoolSize())
                    .build();
            }
            return Health.down().withDetail("reason", "Connection invalid").build();
        } catch (SQLException e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Composite health indicator
@Component("externalServices")
public class ExternalServicesHealthIndicator extends AbstractHealthIndicator {

    private final PaymentGatewayClient paymentClient;
    private final NotificationServiceClient notificationClient;

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        Map<String, Object> details = new LinkedHashMap<>();

        boolean paymentUp = paymentClient.ping();
        boolean notificationUp = notificationClient.ping();

        details.put("payment-gateway", paymentUp ? "UP" : "DOWN");
        details.put("notification-service", notificationUp ? "UP" : "DOWN");

        if (paymentUp && notificationUp) {
            builder.up().withDetails(details);
        } else {
            builder.down().withDetails(details);
        }
    }
}
```

### 9.5 Custom Metrics with Micrometer

```java
@Service
public class OrderService {

    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderAmountSummary;

    public OrderService(MeterRegistry registry) {
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Total number of orders created")
            .tag("environment", "production")
            .register(registry);

        this.orderFailedCounter = Counter.builder("orders.failed")
            .description("Total number of failed orders")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time taken to process an order")
            .publishPercentiles(0.5, 0.75, 0.95, 0.99)
            .register(registry);

        this.orderAmountSummary = DistributionSummary.builder("orders.amount")
            .description("Order value distribution")
            .baseUnit("USD")
            .register(registry);
    }

    public OrderResponse createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                OrderResponse response = doCreateOrder(request);
                orderCreatedCounter.increment();
                orderAmountSummary.record(request.amount().doubleValue());
                return response;
            } catch (Exception e) {
                orderFailedCounter.increment();
                throw e;
            }
        });
    }
}
```

```java
// Custom Actuator endpoint
@Component
@Endpoint(id = "feature-flags")  // accessible at /actuator/feature-flags
public class FeatureFlagsEndpoint {

    private final FeatureFlagService featureFlagService;

    @ReadOperation   // maps to HTTP GET
    public Map<String, Boolean> getFlags() {
        return featureFlagService.getAllFlags();
    }

    @ReadOperation
    public Boolean getFlag(@Selector String flagName) {
        return featureFlagService.isEnabled(flagName);
    }

    @WriteOperation  // maps to HTTP POST
    public void toggleFlag(@Selector String flagName, boolean enabled) {
        featureFlagService.setFlag(flagName, enabled);
    }
}
```

---

## 10. Async REST

### 10.1 Why Async?

Tomcat's default thread pool has 200 threads. For requests that do blocking I/O (external HTTP calls, slow DB queries), each thread sits idle while waiting. Async processing releases the thread back to the pool so it can handle other requests.

### 10.2 DeferredResult

`DeferredResult` allows you to complete the response from a different thread:

```java
@RestController
@RequestMapping("/api/v1/reports")
public class ReportController {

    private final ReportGenerationService reportService;
    private final ExecutorService executorService =
        Executors.newFixedThreadPool(10); // your async executor

    @GetMapping("/{id}/generate")
    public DeferredResult<ResponseEntity<ReportResponse>> generateReport(
            @PathVariable Long id) {

        DeferredResult<ResponseEntity<ReportResponse>> deferredResult =
            new DeferredResult<>(30_000L, // 30 second timeout
                ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT)
                    .body(null)); // timeout result

        executorService.submit(() -> {
            try {
                ReportResponse report = reportService.generate(id); // blocking operation
                deferredResult.setResult(ResponseEntity.ok(report));
            } catch (Exception e) {
                deferredResult.setErrorResult(e);
            }
        });

        return deferredResult; // Tomcat thread is released here!
    }
}
```

### 10.3 CompletableFuture in Controllers

Spring MVC natively supports `CompletableFuture` return types:

```java
@GetMapping("/{id}")
public CompletableFuture<ResponseEntity<UserResponse>> getUser(@PathVariable Long id) {
    return userService.findByIdAsync(id)  // returns CompletableFuture<User>
        .thenApply(user -> ResponseEntity.ok(userMapper.toResponse(user)))
        .exceptionally(ex -> {
            if (ex.getCause() instanceof NotFoundException) {
                return ResponseEntity.notFound().build();
            }
            throw new CompletionException(ex);
        });
}

// In service:
@Service
public class UserService {

    @Async("asyncExecutor")  // custom thread pool
    public CompletableFuture<User> findByIdAsync(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("User not found: " + id));
        return CompletableFuture.completedFuture(user);
    }
}

// Custom async executor configuration
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("asyncExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### 10.4 Callable in Controllers

```java
@GetMapping("/slow-operation")
public Callable<ResponseEntity<String>> slowOperation() {
    // Callable is executed in a separate thread pool managed by Spring MVC
    return () -> {
        Thread.sleep(5000); // simulate slow operation
        return ResponseEntity.ok("Done!");
    };
}
```

### 10.5 WebFlux vs Spring MVC — When to Choose

| Aspect | Spring MVC (Servlet) | Spring WebFlux (Reactive) |
|--------|---------------------|--------------------------|
| Programming model | Imperative (blocking) | Reactive (non-blocking) |
| Thread model | Thread-per-request | Event loop (Netty) |
| Best for | Traditional CRUD, simple APIs | High concurrency, streaming, microservices |
| Learning curve | Low | High (reactive programming concepts) |
| JDBC support | Full (blocking) | R2DBC required for reactive |
| Spring Security | Full support | Full support (different API) |
| Backpressure | ❌ No | ✅ Yes (Reactor) |
| Server | Tomcat/Jetty/Undertow | Netty (default), Tomcat (limited) |

> [!TIP]
> Don't switch to WebFlux just for performance. Spring MVC with async processing + a properly sized thread pool handles millions of requests per day in production. Choose WebFlux when you genuinely need reactive streams, backpressure, or SSE (Server-Sent Events).

```java
// Spring WebFlux example for comparison
@RestController
@RequestMapping("/api/v1/users")
public class UserWebFluxController {

    @GetMapping("/{id}")
    public Mono<UserResponse> getUser(@PathVariable Long id) {
        return userService.findById(id)  // returns Mono<User>
            .map(userMapper::toResponse)
            .switchIfEmpty(Mono.error(new NotFoundException("User not found")));
    }

    @GetMapping
    public Flux<UserResponse> getAllUsers() {
        return userService.findAll()  // returns Flux<User>
            .map(userMapper::toResponse);
    }

    // Server-Sent Events streaming
    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<UserEvent> streamUserEvents() {
        return userEventService.getEventStream();
    }
}
```

---

## 11. ⚠️ Interview Traps & Gotchas

### Trap 1: Filter vs Interceptor — Precise Execution Layer

**Q: What is the difference between a Servlet Filter and a Spring HandlerInterceptor?**

**Wrong answer:** "Filters are for cross-cutting concerns and interceptors are too."

**Correct answer:**
- **Filter** runs at the **Servlet container level** (before `DispatcherServlet`). It has no Spring context awareness — cannot access `HandlerMethod`, `ModelAndView`, or Spring beans (unless explicitly retrieved). It CAN modify the raw request/response bytes.
- **Interceptor** runs **inside** `DispatcherServlet`, after handler resolution. It has full access to the `HandlerMethod` (controller method metadata — annotations, parameter types) and to the `ModelAndView`. It participates in Spring's exception handling.
- Use **Filter** for: authentication, encoding, rate limiting, request logging (raw HTTP layer).
- Use **Interceptor** for: authorization based on method annotations, metrics per controller, locale resolution.

### Trap 2: Can an Interceptor Read the Request Body?

**Q: Can you read the request body in `preHandle()`?**

**Answer:** The request body `InputStream` is single-read. In `preHandle()`, the body has NOT yet been read by Spring (Jackson reads it later during argument resolution). So technically yes, you CAN read it, but doing so BREAKS the controller — Jackson gets an empty stream.

To read it in an interceptor, wrap the request in a `ContentCachingRequestWrapper` in a **Filter** first. Then the body is cached and can be re-read.

### Trap 3: @PathVariable vs @RequestParam

| | @PathVariable | @RequestParam |
|--|--------------|---------------|
| Source | URI path segment `/users/{id}` | Query string `?id=1` or form data |
| Required by default | ✅ Yes | ✅ Yes (can set `required=false`) |
| Default value | ❌ Not supported | ✅ `defaultValue` attribute |
| Type | Any (with type conversion) | Any (with type conversion) |
| Encoding | URL path encoded | URL query encoded |

### Trap 4: What Happens if @RequestBody is Null / Missing?

- If `Content-Type: application/json` but body is empty → `HttpMessageNotReadableException` (400)
- If Content-Type header is missing → `UnsupportedMediaTypeStatusException` (415)
- To make body optional: `@RequestBody(required = false)` — the parameter will be `null`
- If the JSON body has `null` for a non-null annotated field → validation passes at deserialization but fails at `@Valid` (400 with field errors)

### Trap 5: PUT vs PATCH Idempotency

**Q: Is PATCH idempotent?**

It depends:
- `PATCH /users/1 { "status": "active" }` → **idempotent** (applying the same state twice = same result)
- `PATCH /users/1 { "loginCount": { "$increment": 1 } }` → **NOT idempotent** (each call changes state)

PUT MUST send the full resource representation. Sending partial data via PUT is a client error (you'd be overwriting omitted fields with null). PATCH sends only the changed fields.

### Trap 6: OPTIONS Request Handling

Spring MVC automatically handles `OPTIONS` requests to mapped paths, returning the `Allow` header with supported methods. This is used for CORS preflight.

```java
// You do NOT need to write this — Spring does it automatically
@RequestMapping(value = "/users", method = RequestMethod.OPTIONS)
// Spring auto-generates: Allow: GET, POST, HEAD, OPTIONS
```

However, if you want custom `OPTIONS` handling:
```java
@RequestMapping(value = "/{id}", method = RequestMethod.OPTIONS)
public ResponseEntity<Void> options(@PathVariable Long id) {
    return ResponseEntity.ok()
        .allow(HttpMethod.GET, HttpMethod.PUT, HttpMethod.DELETE)
        .build();
}
```

### Trap 7: DispatcherServlet Not Throwing 404

By default, if no handler is found, `DispatcherServlet` does NOT throw a `NoHandlerFoundException`. Instead, it forwards to `/error`. This means your `@ExceptionHandler(NoHandlerFoundException.class)` never fires!

Fix:
```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
  web:
    resources:
      add-mappings: false  # disable ResourceHandlerMapping for /** which catches everything
```

### Trap 8: @ResponseStatus vs ResponseEntity Return Type

```java
// This sets 201 for ALL responses from this method
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserResponse createUser(...) { ... }

// This overrides @ResponseStatus if used
@PostMapping
@ResponseStatus(HttpStatus.CREATED)  // ignored when ResponseEntity is returned
public ResponseEntity<UserResponse> createUser(...) {
    return ResponseEntity.status(HttpStatus.CREATED).body(response); // this wins
}
```

### Trap 9: postHandle Not Called When Exception Thrown

```java
// If your controller throws an exception:
@GetMapping
public UserResponse getUser(@PathVariable Long id) {
    throw new RuntimeException("DB error");
}

// Execution order:
// 1. preHandle() ← called
// 2. controller method ← throws exception
// 3. postHandle() ← NOT called
// 4. HandlerExceptionResolver handles it
// 5. afterCompletion() ← called (with the exception as parameter)
```

### Trap 10: @CrossOrigin with allowCredentials and Wildcard Origin

```java
// This will cause a runtime error at startup (Spring 5.3+):
@CrossOrigin(origins = "*", allowCredentials = "true")
// → IllegalArgumentException: When allowCredentials is true, allowedOrigins cannot contain "*"

// Fix: use allowedOriginPatterns
@CrossOrigin(originPatterns = "*", allowCredentials = "true")
// OR specify exact origins
@CrossOrigin(origins = {"https://myapp.com"}, allowCredentials = "true")
```

### Trap 11: Filter Order with @Component vs FilterRegistrationBean

Using `@Component` on a filter AND registering it via `FilterRegistrationBean` creates **two registrations** — the filter runs **twice**! Choose one approach.

```java
// Wrong — filter runs twice
@Component
public class MyFilter extends OncePerRequestFilter { ... }

@Bean
public FilterRegistrationBean<MyFilter> myFilter(MyFilter filter) {
    FilterRegistrationBean<MyFilter> reg = new FilterRegistrationBean<>(filter);
    // ...
    return reg;
}

// Fix: Remove @Component and only use FilterRegistrationBean
// OR keep @Component and don't create FilterRegistrationBean
```

### Trap 12: Content Negotiation and Jackson View

```java
// Jackson @JsonView lets you expose different fields for different views
public class Views {
    public interface Summary {}
    public interface Detail extends Summary {}
}

public class UserResponse {
    @JsonView(Views.Summary.class)
    private Long id;
    @JsonView(Views.Summary.class)
    private String name;
    @JsonView(Views.Detail.class)  // only in detail view
    private String email;
    @JsonView(Views.Detail.class)
    private LocalDateTime createdAt;
}

@GetMapping("/summary")
@JsonView(Views.Summary.class)  // serializes only Summary-annotated fields
public List<UserResponse> getSummary() { ... }

@GetMapping("/{id}")
@JsonView(Views.Detail.class)   // serializes Summary + Detail fields
public UserResponse getDetail(@PathVariable Long id) { ... }
```

---

## 🔍 Deep Dive — HttpMessageConverter Internals

When `@ResponseBody` is used (or `@RestController`), Spring uses `HttpMessageConverter` to serialize the return value. Spring Boot auto-configures several converters via `HttpMessageConvertersAutoConfiguration`:

```
Registered converters (in order of priority):
1. ByteArrayHttpMessageConverter        → byte[]
2. StringHttpMessageConverter           → String
3. ResourceHttpMessageConverter         → Resource
4. FormHttpMessageConverter             → MultiValueMap (form data)
5. MappingJackson2HttpMessageConverter  → application/json (most common)
6. MappingJackson2XmlHttpMessageConverter → application/xml (if Jackson XML on classpath)
7. Jaxb2RootElementHttpMessageConverter → application/xml (if JAXB on classpath)
```

Content negotiation selects the converter:
1. Spring checks the controller's `produces` attribute
2. Then the client's `Accept` header
3. Then the default (usually `application/json`)

```java
// Customizing Jackson ObjectMapper globally
@Bean
public Jackson2ObjectMapperBuilderCustomizer jsonCustomizer() {
    return builder -> builder
        .featuresToDisable(
            SerializationFeature.WRITE_DATES_AS_TIMESTAMPS,
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES
        )
        .featuresToEnable(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY)
        .modules(new JavaTimeModule())
        .serializationInclusion(JsonInclude.Include.NON_NULL)
        .timeZone(TimeZone.getTimeZone("UTC"));
}
```

---

## 📝 Summary & Key Takeaways

| Concept | Key Point |
|---------|-----------|
| DispatcherServlet | Single front controller; finds handler via HandlerMapping, invokes via HandlerAdapter |
| Request Pipeline | Filter → Interceptor (pre) → AOP → Controller → AOP → Interceptor (post/after) → Filter |
| @RestController | = @Controller + @ResponseBody; uses HttpMessageConverter, not ViewResolver |
| Idempotency | GET/PUT/DELETE/HEAD = idempotent; POST = not; PATCH = depends on semantics |
| Validation | @Valid on @RequestBody → MethodArgumentNotValidException (400); @Validated on path/query params → ConstraintViolationException |
| Filter | Servlet level, no Spring awareness, CAN modify request/response bytes |
| Interceptor | Spring MVC level, accesses HandlerMethod, postHandle skipped on exception |
| OncePerRequestFilter | Preferred filter base class; runs exactly once; Spring-managed (can inject beans) |
| CORS + Security | Must configure CORS at HttpSecurity level when Spring Security is present |
| CORS Credentials | allowedOrigins("*") + allowCredentials=true is INVALID; use allowedOriginPatterns |
| Actuator | Expose minimal endpoints in production; use separate management port |
| Async | DeferredResult / CompletableFuture release Tomcat threads; choose WebFlux for reactive |
| MapStruct | Compile-time, type-safe; preferred over ModelMapper for new projects |
| MDC Cleanup | Always clean up MDC in finally block due to thread pool reuse |

---

## ❓ Practice Questions

**Conceptual:**
1. Explain the complete request processing pipeline from TCP packet to JSON response in Spring Boot. Name every major component in order.
2. Why is `DispatcherServlet` called a "Front Controller"? What problem does this pattern solve?
3. A colleague says "I'll use a HandlerInterceptor to add CORS headers." What problems will they encounter?
4. You need to extract the JWT token from the `Authorization` header for every request. Should you use a Filter or Interceptor, and why?
5. What is the difference between `@Validated` and `@Valid`? When would you use each?

**Debugging:**
6. Your `@ExceptionHandler(NoHandlerFoundException.class)` never fires when users hit undefined endpoints. Why? How do you fix it?
7. A filter is running twice on every request. What likely caused this? How do you diagnose it?
8. Your CORS is configured in `WebMvcConfigurer`, but preflight requests are getting 403. Spring Security is on the classpath. What went wrong?
9. Your correlation ID filter stores data in `MDC` but you notice log lines leaking correlation IDs from previous requests. What did you forget?

**Design:**
10. Design a rate-limiting system for your REST API. Which layer would you implement it in (Filter/Interceptor/AOP), and why? What data would you need to store per client?
11. You need to add request/response body logging for debugging, but some endpoints handle large file uploads. How do you implement this safely?
12. How would you implement API versioning for a public API that needs to support v1 clients for 2 more years while actively developing v3?
13. Design a custom `HealthIndicator` that checks whether a critical Redis cache is reachable. What should it report if Redis is down but data can still be served from the DB?

**Code:**
14. Write a filter that adds a `X-Request-ID` header to every response. If the request already contains the header, use that value; otherwise generate a UUID.
15. Write a `HandlerInterceptor` that reads a `@RequiresRole("ADMIN")` custom annotation from the controller method and returns 403 if the current user doesn't have that role.
16. Write a `MapStruct` mapper that converts a `UserEntity` to `UserResponse`, ignoring the `passwordHash` field and formatting the `createdAt` as an ISO-8601 string.
