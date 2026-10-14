# Exception Handling in Spring Boot

> A production-grade, interview-focused deep dive into every layer of exception handling in Spring Boot 3.x / Spring 6.x

---

## Table of Contents

1. [Exception Hierarchy & Fundamentals](#1-exception-hierarchy--fundamentals)
2. [Default Spring Boot Error Handling](#2-default-spring-boot-error-handling)
3. [@ExceptionHandler — Method-Level Handling](#3-exceptionhandler--method-level-handling)
4. [@ControllerAdvice & @RestControllerAdvice](#4-controlleradvice--restcontrolleradvice)
5. [Building a Production-Ready Error Response](#5-building-a-production-ready-error-response)
6. [Custom Exception Hierarchy for REST APIs](#6-custom-exception-hierarchy-for-rest-apis)
7. [Validation Exceptions](#7-validation-exceptions)
8. [Transaction + Exception Interaction](#8-transaction--exception-interaction)
9. [Spring Data / JPA Exceptions](#9-spring-data--jpa-exceptions)
10. [HTTP Client Exceptions](#10-http-client-exceptions)
11. [Async Exception Handling](#11-async-exception-handling)
12. [Scheduled Task Exception Handling](#12-scheduled-task-exception-handling)
13. [Logging Best Practices](#13-logging-best-practices)
14. [⚠️ Interview Traps & Gotchas](#14-️-interview-traps--gotchas)
15. [🔍 Deep Dive — Internals](#15--deep-dive--internals)
16. [📝 Summary & Key Takeaways](#16--summary--key-takeaways)
17. [❓ Practice Questions](#17--practice-questions)

---

## 1. Exception Hierarchy & Fundamentals

### 1.1 Java Exception Hierarchy

```
Throwable
├── Error                         ← JVM-level, do NOT catch
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── AssertionError
└── Exception
    ├── RuntimeException          ← Unchecked
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── UnsupportedOperationException
    │   └── ... (all Spring exceptions)
    └── IOException               ← Checked
        ├── FileNotFoundException
        └── ...
```

**Checked vs Unchecked — The core distinction:**

| Feature | Checked | Unchecked |
|---------|---------|-----------|
| Extends | `Exception` (not `RuntimeException`) | `RuntimeException` |
| Compile-time enforcement | Yes — must declare or catch | No |
| Use case | Recoverable conditions | Programming errors / unexpected state |
| Spring's preference | Avoided internally | All Spring exceptions are unchecked |
| `@Transactional` rollback | **NO** by default ⚠️ | Yes by default |

> [!IMPORTANT]
> Spring made a deliberate design decision: **all Spring framework exceptions extend `RuntimeException`**. This avoids forcing callers to write try-catch boilerplate. This is why `@Transactional` only rolls back on `RuntimeException` (and `Error`) by default.

### 1.2 Spring's Exception Translation Infrastructure

Spring's **persistence exception translation** converts low-level, technology-specific exceptions into a unified `DataAccessException` hierarchy. This is enabled by `@Repository` or the `PersistenceExceptionTranslationPostProcessor` bean.

```
DataAccessException (abstract, RuntimeException)
├── DataIntegrityViolationException     ← FK violation, unique constraint
├── EmptyResultDataAccessException      ← Expected 1 row, got 0
├── IncorrectResultSizeDataAccessException
├── OptimisticLockingFailureException   ← @Version conflict
├── PessimisticLockingFailureException
├── QueryTimeoutException
├── DataRetrievalFailureException
└── TransientDataAccessException
    ├── LockAcquisitionException
    └── QueryTimeoutException
```

```java
// Spring wraps Hibernate's HibernateException, JPA's PersistenceException,
// JDBC's SQLException — all into DataAccessException subclasses.
// This is done via PersistenceExceptionTranslator implementations.

// Hibernate → Spring translation example:
// ConstraintViolationException (Hibernate) → DataIntegrityViolationException (Spring)
// ObjectNotFoundException (Hibernate)      → JpaObjectRetrievalFailureException (Spring)
```

> [!NOTE]
> Exception translation is **only applied to beans annotated with `@Repository`** (or when `PersistenceExceptionTranslationPostProcessor` is in the context). A plain service class does NOT get translation.

---

## 2. Default Spring Boot Error Handling

### 2.1 The Auto-Configuration Chain

When an unhandled exception escapes a controller, Spring Boot's auto-configuration kicks in:

```
Request → DispatcherServlet → Controller → Exception thrown
                                              ↓
                              HandlerExceptionResolver chain
                              (ExceptionHandlerExceptionResolver,
                               ResponseStatusExceptionResolver,
                               DefaultHandlerExceptionResolver)
                                              ↓ (none handled it)
                              ErrorController → /error endpoint
                              BasicErrorController
```

### 2.2 BasicErrorController

Spring Boot auto-configures `BasicErrorController` which handles `/error` requests:

```java
// Simplified view of BasicErrorController internals
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {

    // Returns HTML for browser requests (Accept: text/html)
    @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
    public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
        HttpStatus status = getStatus(request);
        Map<String, Object> model = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML));
        response.setStatus(status.value());
        ModelAndView modelAndView = resolveErrorView(request, response, status, model);
        return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
    }

    // Returns JSON for API requests (Accept: application/json)
    @RequestMapping
    public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
        HttpStatus status = getStatus(request);
        Map<String, Object> body = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.ALL));
        return new ResponseEntity<>(body, status);
    }
}
```

### 2.3 Default Error Attributes

By default, `/error` returns:

```json
{
  "timestamp": "2024-01-15T10:30:00.000+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "",
  "path": "/api/users/999"
}
```

> [!WARNING]
> In Spring Boot 2.3+, `message` and `trace` are **hidden by default** for security reasons. Enable them via:
> ```properties
> server.error.include-message=always
> server.error.include-stacktrace=always
> server.error.include-binding-errors=always
> ```
> **Never enable stacktrace in production!**

### 2.4 Customizing ErrorAttributes

```java
@Component
public class CustomErrorAttributes extends DefaultErrorAttributes {

    @Override
    public Map<String, Object> getErrorAttributes(
            WebRequest webRequest, ErrorAttributeOptions options) {

        Map<String, Object> errorAttributes = super.getErrorAttributes(webRequest, options);

        // Add custom fields
        errorAttributes.put("application", "my-service");
        errorAttributes.put("correlationId", MDC.get("correlationId"));

        // Remove sensitive fields
        errorAttributes.remove("trace");

        return errorAttributes;
    }
}
```

### 2.5 Custom Error Controller

```java
@Controller
@RequestMapping("/error")
public class CustomErrorController implements ErrorController {

    private final ErrorAttributes errorAttributes;

    public CustomErrorController(ErrorAttributes errorAttributes) {
        this.errorAttributes = errorAttributes;
    }

    @RequestMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<ApiError> handleError(HttpServletRequest request) {
        WebRequest webRequest = new ServletWebRequest(request);
        Map<String, Object> attrs = errorAttributes.getErrorAttributes(
            webRequest, ErrorAttributeOptions.defaults());

        HttpStatus status = HttpStatus.valueOf((Integer) attrs.getOrDefault("status", 500));
        ApiError error = ApiError.builder()
            .status(status.value())
            .message((String) attrs.getOrDefault("message", "An error occurred"))
            .path((String) attrs.get("path"))
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(status).body(error);
    }
}
```

### 2.6 The Whitelabel Error Page

The Whitelabel error page is shown when no custom error view is found. Disable it:

```properties
# Option 1: Disable whitelabel, fall through to ErrorController
server.error.whitelabel.enabled=false

# Option 2: Provide custom Thymeleaf templates
# src/main/resources/templates/error/404.html
# src/main/resources/templates/error/500.html
# src/main/resources/templates/error/4xx.html
# src/main/resources/templates/error/5xx.html
```

```java
// Option 3: Exclude autoconfiguration entirely
@SpringBootApplication(exclude = ErrorMvcAutoConfiguration.class)
public class Application { ... }
```

---

## 3. @ExceptionHandler — Method-Level Handling

### 3.1 Basic Usage

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id); // may throw UserNotFoundException
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    // ---- Exception Handlers (local to this controller) ----

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handleUserNotFound(
            UserNotFoundException ex,
            HttpServletRequest request) {

        ApiError error = ApiError.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // Handles multiple exception types
    @ExceptionHandler({IllegalArgumentException.class, IllegalStateException.class})
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleBadRequest(RuntimeException ex, HttpServletRequest request) {
        return ApiError.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }
}
```

### 3.2 Supported Method Signatures

`@ExceptionHandler` methods are extremely flexible in the parameters they accept:

```java
@ExceptionHandler(SomeException.class)
public ResponseEntity<ApiError> handle(
    SomeException ex,                // The exception itself
    HttpServletRequest request,      // Servlet request
    HttpServletResponse response,    // Servlet response
    WebRequest webRequest,           // Spring's WebRequest abstraction
    Locale locale,                   // Current locale (for i18n)
    Principal principal,             // Authenticated principal
    Authentication authentication,   // Spring Security authentication
    Model model,                     // For view-based responses
    BindingResult bindingResult      // Binding result (if applicable)
) { ... }
```

### 3.3 Supported Return Types

```java
// 1. ResponseEntity<T> — Full control over status, headers, body
@ExceptionHandler(NotFoundException.class)
public ResponseEntity<ApiError> handle(NotFoundException ex) {
    return ResponseEntity.status(404).body(new ApiError(...));
}

// 2. @ResponseBody + @ResponseStatus — Shorthand
@ExceptionHandler(NotFoundException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
@ResponseBody
public ApiError handle(NotFoundException ex) {
    return new ApiError(...);
}

// 3. String (view name, for MVC)
@ExceptionHandler(SomeException.class)
public String handle(SomeException ex, Model model) {
    model.addAttribute("error", ex.getMessage());
    return "error-view";
}

// 4. ModelAndView (for MVC)
@ExceptionHandler(SomeException.class)
public ModelAndView handle(SomeException ex) {
    ModelAndView mav = new ModelAndView("error");
    mav.addObject("message", ex.getMessage());
    return mav;
}

// 5. void (manipulate response directly)
@ExceptionHandler(SomeException.class)
public void handle(SomeException ex, HttpServletResponse response) throws IOException {
    response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR, ex.getMessage());
}
```

### 3.4 Limitations of Method-Level @ExceptionHandler

> [!WARNING]
> **Critical limitation**: `@ExceptionHandler` methods in a controller class **only handle exceptions thrown by handler methods in that same controller**. They do NOT handle exceptions from other controllers.

```java
@RestController
public class OrderController {

    // This handler CANNOT catch exceptions from UserController
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handle(UserNotFoundException ex) { ... }
}
```

**Solution**: Use `@ControllerAdvice` for global handling (Section 4).

---

## 4. @ControllerAdvice & @RestControllerAdvice

### 4.1 @ControllerAdvice — Global Exception Handling

`@ControllerAdvice` is a specialization of `@Component` that acts as an interceptor for all controllers. It supports:
- `@ExceptionHandler` — exception handling
- `@ModelAttribute` — add model attributes to all controllers
- `@InitBinder` — customize data binding

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {

        log.warn("Resource not found: {} at {}", ex.getMessage(), request.getRequestURI());

        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ApiError.of(HttpStatus.NOT_FOUND, ex.getMessage(), request));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneric(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error at {}", request.getRequestURI(), ex);

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiError.of(HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred", request));
    }
}
```

### 4.2 @RestControllerAdvice = @ControllerAdvice + @ResponseBody

```java
// @RestControllerAdvice is shorthand:
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ControllerAdvice
@ResponseBody
public @interface RestControllerAdvice { ... }

// These two are equivalent:

@ControllerAdvice
@ResponseBody
public class GlobalHandler { ... }

// ↕ Same as:

@RestControllerAdvice
public class GlobalHandler { ... }
```

> [!TIP]
> Use `@RestControllerAdvice` for pure REST APIs (returns JSON). Use `@ControllerAdvice` when you also need to return view names (String) for MVC responses.

### 4.3 Scope Control — Targeting Specific Controllers

```java
// Target by base package
@RestControllerAdvice(basePackages = "com.example.api.v2")
public class V2ExceptionHandler { ... }

// Target by annotation
@RestControllerAdvice(annotations = RestController.class)
public class RestApiExceptionHandler { ... }

// Target by assignable type (specific controller classes or interfaces)
@RestControllerAdvice(assignableTypes = {UserController.class, OrderController.class})
public class UserOrderExceptionHandler { ... }

// Combine conditions (OR logic)
@RestControllerAdvice(
    basePackages = "com.example.api",
    annotations = RestController.class
)
public class ApiExceptionHandler { ... }
```

### 4.4 Priority When Multiple @ControllerAdvice Exist

> [!IMPORTANT]
> When multiple `@ControllerAdvice` beans handle the same exception type, **the most specific handler wins**. If equally specific, the ordering matters.

**Rules (in order of precedence):**

1. **Local `@ExceptionHandler`** (in the controller itself) always wins over `@ControllerAdvice`
2. Among `@ControllerAdvice` beans, **`@Order` / `Ordered` interface** determines priority
3. Lower order value = higher priority (`@Order(1)` beats `@Order(2)`)
4. Without `@Order`, ordering is **undefined** (depends on bean registration order)

```java
// This handler takes priority for validation errors (runs first)
@RestControllerAdvice
@Order(1)
public class ValidationExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationError> handleValidation(
            MethodArgumentNotValidException ex) {
        // Specialized handler with field-level details
        List<FieldError> errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .collect(Collectors.toList());

        return ResponseEntity.badRequest().body(new ValidationError(errors));
    }
}

// Fallback handler for everything else (runs last)
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)
public class FallbackExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleAll(Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.internalServerError()
            .body(ApiError.of(500, "Internal Server Error", request));
    }
}
```

### 4.5 Extending ResponseEntityExceptionHandler

Spring provides `ResponseEntityExceptionHandler` — a base class that already handles all standard Spring MVC exceptions. **Extend this instead of starting from scratch.**

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // Override specific Spring MVC exception handlers
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        List<FieldValidationError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> new FieldValidationError(
                fe.getField(),
                fe.getRejectedValue(),
                fe.getDefaultMessage()))
            .collect(Collectors.toList());

        ValidationApiError error = ValidationApiError.builder()
            .status(status.value())
            .message("Validation failed")
            .path(extractPath(request))
            .timestamp(Instant.now())
            .fieldErrors(fieldErrors)
            .build();

        return ResponseEntity.status(status).body(error);
    }

    @Override
    protected ResponseEntity<Object> handleHttpMessageNotReadable(
            HttpMessageNotReadableException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        String message = "Malformed JSON request";
        if (ex.getCause() instanceof InvalidFormatException ife) {
            message = "Invalid value '%s' for field '%s'. Expected type: %s"
                .formatted(ife.getValue(), extractFieldName(ife), ife.getTargetType().getSimpleName());
        }

        return ResponseEntity.badRequest().body(
            ApiError.of(400, message, extractPath(request)));
    }

    @Override
    protected ResponseEntity<Object> handleNoResourceFoundException(
            NoResourceFoundException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(
            ApiError.of(404, "No resource found at: " + ex.getResourcePath(), extractPath(request)));
    }

    // Add your own custom handlers on top
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiError> handleBusinessException(
            BusinessException ex, WebRequest request) {

        log.warn("Business rule violation: {}", ex.getMessage());
        return ResponseEntity
            .status(ex.getHttpStatus())
            .body(ApiError.of(ex.getHttpStatus().value(), ex.getMessage(), extractPath(request)));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(Exception ex, WebRequest request) {
        log.error("Unexpected error processing request to {}", extractPath(request), ex);
        return ResponseEntity.internalServerError()
            .body(ApiError.of(500, "An internal error occurred", extractPath(request)));
    }

    private String extractPath(WebRequest request) {
        return ((ServletWebRequest) request).getRequest().getRequestURI();
    }

    private String extractFieldName(InvalidFormatException ife) {
        return ife.getPath().stream()
            .map(ref -> ref.getFieldName())
            .collect(Collectors.joining("."));
    }
}
```

> [!NOTE]
> Spring MVC exceptions handled by `ResponseEntityExceptionHandler` include:
> `MethodArgumentNotValidException`, `BindException`, `TypeMismatchException`,
> `HttpMessageNotReadableException`, `HttpRequestMethodNotSupportedException`,
> `MissingServletRequestParameterException`, `NoHandlerFoundException`, and many more.

---

## 5. Building a Production-Ready Error Response

### 5.1 Standard Error Response Structure

```java
// Core error response DTO
@Getter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiError {

    private final int status;
    private final String error;            // HTTP status text
    private final String message;          // Human-readable message
    private final String path;             // Request URI
    private final Instant timestamp;
    private final String correlationId;    // For tracing
    private final String errorCode;        // Internal error code (e.g., "USER_NOT_FOUND")
    private final List<FieldError> fieldErrors;   // For validation errors
    private final Map<String, Object> details;    // Extra context

    public static ApiError of(HttpStatus status, String message, String path) {
        return ApiError.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .message(message)
            .path(path)
            .timestamp(Instant.now())
            .correlationId(MDC.get("correlationId"))
            .build();
    }

    public static ApiError of(int statusCode, String message, String path) {
        return of(HttpStatus.valueOf(statusCode), message, path);
    }
}

// Field-level validation error
@Getter
@AllArgsConstructor
public class FieldError {
    private final String field;
    private final Object rejectedValue;
    private final String message;
}
```

### 5.2 Validation Error Response

```java
// Extended error response for validation failures
@Getter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ValidationApiError {

    private final int status;
    private final String message;
    private final String path;
    private final Instant timestamp;
    private final List<FieldError> fieldErrors;

    // Example JSON output:
    // {
    //   "status": 400,
    //   "message": "Validation failed",
    //   "path": "/api/users",
    //   "timestamp": "2024-01-15T10:30:00Z",
    //   "fieldErrors": [
    //     { "field": "email", "rejectedValue": "not-an-email", "message": "must be a valid email" },
    //     { "field": "age",   "rejectedValue": -1,             "message": "must be greater than 0" }
    //   ]
    // }
}
```

### 5.3 RFC 7807 Problem Details (Standard)

RFC 7807 defines a standard JSON format for API errors. Spring 6 / Spring Boot 3 implements this natively.

**Standard fields:**
- `type` — URI reference identifying the error type
- `title` — Short human-readable summary
- `status` — HTTP status code
- `detail` — Detailed explanation
- `instance` — URI of the specific occurrence

```java
// Spring 6 native ProblemDetail support
@ExceptionHandler(ResourceNotFoundException.class)
public ProblemDetail handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {

    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.NOT_FOUND,
        ex.getMessage()
    );
    problem.setTitle("Resource Not Found");
    problem.setType(URI.create("https://api.myapp.com/errors/not-found"));
    problem.setInstance(URI.create(request.getRequestURI()));

    // Custom extensions
    problem.setProperty("errorCode", "RESOURCE_NOT_FOUND");
    problem.setProperty("timestamp", Instant.now());
    problem.setProperty("correlationId", MDC.get("correlationId"));

    return problem;
}
```

```json
// Response with Content-Type: application/problem+json
{
  "type": "https://api.myapp.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with id 999 was not found",
  "instance": "/api/users/999",
  "errorCode": "RESOURCE_NOT_FOUND",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlationId": "abc-123"
}
```

### 5.4 Enabling ProblemDetail in Spring Boot 3

```properties
# Enable RFC 7807 Problem Details for all Spring MVC exceptions
spring.mvc.problemdetails.enabled=true
```

```java
// For custom exceptions, implement ErrorResponseException
public class UserNotFoundException extends ErrorResponseException {

    public UserNotFoundException(Long userId) {
        super(HttpStatus.NOT_FOUND,
              createProblemDetail(userId),
              null);
    }

    private static ProblemDetail createProblemDetail(Long userId) {
        ProblemDetail detail = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            "User with id " + userId + " was not found"
        );
        detail.setTitle("User Not Found");
        detail.setType(URI.create("https://api.myapp.com/errors/user-not-found"));
        detail.setProperty("userId", userId);
        return detail;
    }
}
```

### 5.5 Internationalization of Error Messages

```java
// messages.properties (default)
error.user.not_found=User with id {0} was not found
error.validation.email=Email must be a valid email address
error.validation.required=Field {0} is required

// messages_es.properties
error.user.not_found=Usuario con id {0} no fue encontrado
```

```java
@RestControllerAdvice
public class I18nExceptionHandler {

    private final MessageSource messageSource;

    public I18nExceptionHandler(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request,
            Locale locale) {

        String message = messageSource.getMessage(
            ex.getMessageCode(),
            ex.getArgs(),
            ex.getMessage(),   // fallback
            locale
        );

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiError.of(HttpStatus.NOT_FOUND, message, request.getRequestURI()));
    }
}
```

```java
// Base exception with i18n support
@Getter
public abstract class LocalizableException extends RuntimeException {

    private final String messageCode;
    private final Object[] args;

    protected LocalizableException(String messageCode, Object[] args, String defaultMessage) {
        super(defaultMessage);
        this.messageCode = messageCode;
        this.args = args;
    }
}

public class ResourceNotFoundException extends LocalizableException {
    public ResourceNotFoundException(String resourceType, Long id) {
        super(
            "error." + resourceType.toLowerCase() + ".not_found",
            new Object[]{id},
            resourceType + " with id " + id + " was not found"
        );
    }
}
```

---

## 6. Custom Exception Hierarchy for REST APIs

### 6.1 Designing the Exception Hierarchy

```
RuntimeException
└── ApplicationException (base)
    ├── BusinessException (domain logic violations)
    │   ├── ResourceNotFoundException     → 404
    │   ├── ResourceAlreadyExistsException→ 409
    │   ├── InsufficientFundsException    → 422
    │   └── OrderAlreadyCancelledException→ 422
    ├── ValidationException               → 400
    │   ├── InvalidInputException
    │   └── ConstraintViolationException
    ├── AuthenticationException           → 401
    ├── AuthorizationException            → 403
    └── ExternalServiceException          → 502
        ├── PaymentGatewayException
        └── EmailServiceException
```

### 6.2 Base Exception Classes

```java
// Root of all application exceptions
@Getter
public abstract class ApplicationException extends RuntimeException {

    private final ErrorCode errorCode;
    private final HttpStatus httpStatus;
    private final Map<String, Object> details;

    protected ApplicationException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = errorCode.getHttpStatus();
        this.details = new LinkedHashMap<>();
    }

    protected ApplicationException(ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = errorCode.getHttpStatus();
        this.details = new LinkedHashMap<>();
    }

    public ApplicationException withDetail(String key, Object value) {
        this.details.put(key, value);
        return this;
    }
}

// Business rule violations
public abstract class BusinessException extends ApplicationException {
    protected BusinessException(ErrorCode errorCode, String message) {
        super(errorCode, message);
    }
}
```

### 6.3 ErrorCode Enum Pattern

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    // 4xx Client Errors
    RESOURCE_NOT_FOUND("ERR_001", HttpStatus.NOT_FOUND),
    RESOURCE_ALREADY_EXISTS("ERR_002", HttpStatus.CONFLICT),
    INVALID_INPUT("ERR_003", HttpStatus.BAD_REQUEST),
    VALIDATION_FAILED("ERR_004", HttpStatus.BAD_REQUEST),
    UNAUTHORIZED("ERR_005", HttpStatus.UNAUTHORIZED),
    FORBIDDEN("ERR_006", HttpStatus.FORBIDDEN),
    INSUFFICIENT_FUNDS("ERR_007", HttpStatus.UNPROCESSABLE_ENTITY),
    ORDER_ALREADY_CANCELLED("ERR_008", HttpStatus.UNPROCESSABLE_ENTITY),
    ACCOUNT_LOCKED("ERR_009", HttpStatus.LOCKED),

    // 5xx Server Errors
    INTERNAL_ERROR("ERR_500", HttpStatus.INTERNAL_SERVER_ERROR),
    EXTERNAL_SERVICE_ERROR("ERR_502", HttpStatus.BAD_GATEWAY),
    SERVICE_UNAVAILABLE("ERR_503", HttpStatus.SERVICE_UNAVAILABLE),
    DATABASE_ERROR("ERR_504", HttpStatus.INTERNAL_SERVER_ERROR);

    private final String code;
    private final HttpStatus httpStatus;
}
```

### 6.4 Concrete Exception Implementations

```java
public class ResourceNotFoundException extends BusinessException {

    public ResourceNotFoundException(String resourceType, Object id) {
        super(ErrorCode.RESOURCE_NOT_FOUND,
              String.format("%s with id '%s' was not found", resourceType, id));
        withDetail("resourceType", resourceType);
        withDetail("id", id);
    }

    // Convenience factory methods
    public static ResourceNotFoundException user(Long userId) {
        return new ResourceNotFoundException("User", userId);
    }

    public static ResourceNotFoundException order(Long orderId) {
        return new ResourceNotFoundException("Order", orderId);
    }

    public static ResourceNotFoundException product(String sku) {
        return new ResourceNotFoundException("Product", sku);
    }
}

public class ResourceAlreadyExistsException extends BusinessException {
    public ResourceAlreadyExistsException(String resourceType, String identifier) {
        super(ErrorCode.RESOURCE_ALREADY_EXISTS,
              String.format("%s with identifier '%s' already exists", resourceType, identifier));
        withDetail("resourceType", resourceType);
        withDetail("identifier", identifier);
    }
}

public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException(BigDecimal required, BigDecimal available) {
        super(ErrorCode.INSUFFICIENT_FUNDS,
              String.format("Insufficient funds. Required: %s, Available: %s", required, available));
        withDetail("required", required);
        withDetail("available", available);
    }
}

public class ExternalServiceException extends ApplicationException {
    public ExternalServiceException(String serviceName, String reason, Throwable cause) {
        super(ErrorCode.EXTERNAL_SERVICE_ERROR,
              String.format("External service '%s' failed: %s", serviceName, reason),
              cause);
        withDetail("serviceName", serviceName);
    }
}
```

### 6.5 Exception-to-Status Mapping in Handler

```java
@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
@Slf4j
public class ApplicationExceptionHandler {

    @ExceptionHandler(ApplicationException.class)
    public ResponseEntity<ApiError> handleApplicationException(
            ApplicationException ex,
            HttpServletRequest request) {

        // Log based on severity
        if (ex.getHttpStatus().is5xxServerError()) {
            log.error("Application error [{}]: {}", ex.getErrorCode().getCode(), ex.getMessage(), ex);
        } else {
            log.warn("Application error [{}]: {}", ex.getErrorCode().getCode(), ex.getMessage());
        }

        ApiError error = ApiError.builder()
            .status(ex.getHttpStatus().value())
            .error(ex.getHttpStatus().getReasonPhrase())
            .errorCode(ex.getErrorCode().getCode())
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .correlationId(MDC.get("correlationId"))
            .details(ex.getDetails().isEmpty() ? null : ex.getDetails())
            .build();

        return ResponseEntity.status(ex.getHttpStatus()).body(error);
    }
}
```

---

## 7. Validation Exceptions

### 7.1 Exception Types Comparison

| Exception | Trigger | Source | Status |
|-----------|---------|--------|--------|
| `MethodArgumentNotValidException` | `@Valid`/`@Validated` on `@RequestBody` | Spring MVC | 400 |
| `ConstraintViolationException` | `@Validated` on method params/class level | Bean Validation | 400 |
| `BindException` | `@Valid` on `@ModelAttribute` | Spring MVC | 400 |
| `TypeMismatchException` | Wrong type in path/query param | Spring MVC | 400 |
| `MissingServletRequestParameterException` | Missing required `@RequestParam` | Spring MVC | 400 |
| `HttpMessageNotReadableException` | Malformed JSON body | Jackson | 400 |

### 7.2 MethodArgumentNotValidException (from @Valid on @RequestBody)

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
        // If validation fails → MethodArgumentNotValidException is thrown
        ...
    }
}

// Handler
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ValidationApiError> handleValidation(
        MethodArgumentNotValidException ex,
        HttpServletRequest request) {

    BindingResult bindingResult = ex.getBindingResult();

    // Field-level errors (e.g., @NotBlank, @Email, @Size)
    List<FieldError> fieldErrors = bindingResult.getFieldErrors().stream()
        .map(fe -> new FieldError(
            fe.getField(),
            fe.getRejectedValue(),
            fe.getDefaultMessage()))
        .collect(Collectors.toList());

    // Object-level errors (e.g., @AssertTrue on class)
    List<String> globalErrors = bindingResult.getGlobalErrors().stream()
        .map(DefaultMessageSourceResolvable::getDefaultMessage)
        .collect(Collectors.toList());

    ValidationApiError error = ValidationApiError.builder()
        .status(400)
        .message("Validation failed for " + bindingResult.getObjectName())
        .path(request.getRequestURI())
        .timestamp(Instant.now())
        .fieldErrors(fieldErrors)
        .globalErrors(globalErrors)
        .build();

    return ResponseEntity.badRequest().body(error);
}
```

### 7.3 ConstraintViolationException (from @Validated at class/method level)

```java
@Service
@Validated  // Enables method-level constraint validation
public class UserService {

    public UserResponse findByEmail(@Email @NotBlank String email) {
        // If email is blank or invalid → ConstraintViolationException thrown
        ...
    }
}

// Also triggered for path/query params in controllers
@RestController
@Validated  // Required on controller for param validation
public class ProductController {

    @GetMapping("/products/{id}")
    public Product get(@PathVariable @Positive Long id) {
        // If id <= 0 → ConstraintViolationException
        ...
    }
}

// Handler — note: different from MethodArgumentNotValidException!
@ExceptionHandler(ConstraintViolationException.class)
public ResponseEntity<ValidationApiError> handleConstraintViolation(
        ConstraintViolationException ex,
        HttpServletRequest request) {

    List<FieldError> errors = ex.getConstraintViolations().stream()
        .map(cv -> new FieldError(
            extractFieldName(cv.getPropertyPath().toString()),
            cv.getInvalidValue(),
            cv.getMessage()))
        .collect(Collectors.toList());

    return ResponseEntity.badRequest().body(ValidationApiError.builder()
        .status(400)
        .message("Constraint violation")
        .path(request.getRequestURI())
        .fieldErrors(errors)
        .timestamp(Instant.now())
        .build());
}

private String extractFieldName(String propertyPath) {
    // propertyPath is like "findByEmail.email" → extract "email"
    int lastDot = propertyPath.lastIndexOf('.');
    return lastDot >= 0 ? propertyPath.substring(lastDot + 1) : propertyPath;
}
```

### 7.4 TypeMismatchException

```java
// Triggered when Spring cannot convert a path/query variable
// GET /api/products/abc → @PathVariable Long id → TypeMismatchException

@ExceptionHandler(TypeMismatchException.class)
public ResponseEntity<ApiError> handleTypeMismatch(
        TypeMismatchException ex,
        HttpServletRequest request) {

    String message = String.format(
        "Parameter '%s' should be of type '%s' but got value '%s'",
        ex.getPropertyName(),
        ex.getRequiredType() != null ? ex.getRequiredType().getSimpleName() : "unknown",
        ex.getValue()
    );

    return ResponseEntity.badRequest()
        .body(ApiError.of(HttpStatus.BAD_REQUEST, message, request.getRequestURI()));
}
```

### 7.5 MissingServletRequestParameterException

```java
// GET /api/search → but @RequestParam(required=true) String query missing
@ExceptionHandler(MissingServletRequestParameterException.class)
public ResponseEntity<ApiError> handleMissingParam(
        MissingServletRequestParameterException ex,
        HttpServletRequest request) {

    String message = String.format(
        "Required request parameter '%s' of type '%s' is missing",
        ex.getParameterName(),
        ex.getParameterType()
    );

    return ResponseEntity.badRequest()
        .body(ApiError.of(HttpStatus.BAD_REQUEST, message, request.getRequestURI()));
}
```

### 7.6 HttpMessageNotReadableException

```java
// Triggered when Jackson cannot parse the request body
// - Malformed JSON
// - Wrong types (e.g., string where number expected)
// - Missing required fields (if configured)

@ExceptionHandler(HttpMessageNotReadableException.class)
public ResponseEntity<ApiError> handleNotReadable(
        HttpMessageNotReadableException ex,
        HttpServletRequest request) {

    String message = "Malformed or unreadable request body";

    Throwable cause = ex.getCause();
    if (cause instanceof InvalidFormatException ife) {
        String field = ife.getPath().stream()
            .map(JsonMappingException.Reference::getFieldName)
            .collect(Collectors.joining("."));
        message = String.format(
            "Invalid value '%s' for field '%s'. Expected: %s",
            ife.getValue(), field, ife.getTargetType().getSimpleName()
        );
    } else if (cause instanceof MismatchedInputException mie) {
        message = "Required field is missing or has wrong type";
    }

    return ResponseEntity.badRequest()
        .body(ApiError.of(HttpStatus.BAD_REQUEST, message, request.getRequestURI()));
}
```

---

## 8. Transaction + Exception Interaction

### 8.1 Default Rollback Rules

> [!IMPORTANT]
> `@Transactional` **only rolls back on `RuntimeException` and `Error` by default**. Checked exceptions do NOT trigger rollback unless explicitly configured.

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Order order) throws OrderProcessingException {
        orderRepository.save(order);
        inventoryService.deduct(order);
        // If OrderProcessingException (checked) is thrown here:
        // → Transaction COMMITS ← This is the famous checked exception trap!
        paymentService.charge(order);
    }
}
```

### 8.2 Rollback Rules Configuration

```java
// Force rollback for checked exceptions
@Transactional(rollbackFor = Exception.class)
public void processOrder(Order order) throws OrderProcessingException {
    // Now ANY exception (checked or unchecked) triggers rollback
}

// Force rollback for specific checked exception
@Transactional(rollbackFor = {OrderProcessingException.class, PaymentException.class})
public void processOrder(Order order) throws OrderProcessingException {
    ...
}

// Suppress rollback for specific runtime exception
@Transactional(noRollbackFor = OptimisticLockingFailureException.class)
public void updateOrder(Order order) {
    // OptimisticLockingFailureException will NOT trigger rollback
    // (you'll handle the retry yourself)
}
```

### 8.3 Complete Rollback Rules Summary

| Exception Type | Default Behavior | Override Option |
|---------------|-----------------|-----------------|
| `RuntimeException` | **Rollback** | `noRollbackFor` |
| `Error` | **Rollback** | `noRollbackFor` |
| Checked `Exception` | **Commit** ⚠️ | `rollbackFor` |
| `@ApplicationException(rollback=true)` | **Rollback** (JEE) | N/A in Spring |

### 8.4 @Transactional and @ExceptionHandler Interaction

```java
@Service
public class UserService {

    @Transactional
    public User createUser(CreateUserRequest request) {
        User user = userRepository.save(new User(request));
        // DataIntegrityViolationException thrown here (unique constraint)
        // → Transaction is marked for rollback
        return user;
    }
}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ApiError> handleDataIntegrity(DataIntegrityViolationException ex) {
        // By the time this handler runs, the transaction is ALREADY rolled back
        // The handler simply translates the exception to an HTTP response
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ApiError.of(HttpStatus.CONFLICT, "Data integrity violation", ""));
    }
}
```

### 8.5 The Self-Invocation Trap with Transactions

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        // ...
        this.notifyUser(order); // SELF-INVOCATION! @Transactional is IGNORED
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void notifyUser(Order order) {
        // This runs in the SAME transaction as createOrder!
        // Because proxy is bypassed on self-invocation
    }
}

// Solution: Inject self or use ApplicationContext
@Service
public class OrderService {

    @Autowired
    private OrderService self; // Spring injects the proxy

    @Transactional
    public void createOrder(Order order) {
        self.notifyUser(order); // Now uses proxy, @Transactional respected
    }
}
```

### 8.6 Exception Handling in Nested Transactions

```java
@Service
public class PaymentService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordPayment(Payment payment) {
        // New independent transaction
        paymentRepository.save(payment);
        if (payment.isFraudulent()) {
            throw new FraudDetectedException("Fraudulent payment detected");
            // Rolls back ONLY this inner transaction
        }
    }
}

@Service
public class OrderService {

    private final PaymentService paymentService;

    @Transactional
    public void processOrder(Order order) {
        orderRepository.save(order);
        try {
            paymentService.recordPayment(order.getPayment());
        } catch (FraudDetectedException e) {
            // Inner transaction already rolled back
            // Outer transaction can still commit or rollback
            order.setStatus(OrderStatus.PAYMENT_FAILED);
            orderRepository.save(order);
            // No re-throw → outer transaction COMMITS
        }
    }
}
```

---

## 9. Spring Data / JPA Exceptions

### 9.1 Common JPA/Spring Data Exceptions

```java
@RestControllerAdvice
public class DataExceptionHandler {

    // Unique constraint, FK violation, NOT NULL violation
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ApiError> handleDataIntegrity(
            DataIntegrityViolationException ex,
            HttpServletRequest request) {

        String message = "Data integrity constraint violated";

        // Inspect the root cause for more context
        Throwable rootCause = ex.getMostSpecificCause();
        if (rootCause.getMessage() != null) {
            if (rootCause.getMessage().contains("unique")) {
                message = "A record with this identifier already exists";
            } else if (rootCause.getMessage().contains("foreign key")) {
                message = "Referenced resource does not exist";
            }
        }

        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ApiError.of(HttpStatus.CONFLICT, message, request.getRequestURI()));
    }

    // Expected 1 result, got 0
    @ExceptionHandler(EmptyResultDataAccessException.class)
    public ResponseEntity<ApiError> handleEmptyResult(
            EmptyResultDataAccessException ex,
            HttpServletRequest request) {

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiError.of(HttpStatus.NOT_FOUND, "Resource not found", request.getRequestURI()));
    }

    // @Version field conflict (optimistic locking)
    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<ApiError> handleOptimisticLock(
            OptimisticLockingFailureException ex,
            HttpServletRequest request) {

        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ApiError.of(HttpStatus.CONFLICT,
                "Resource was modified by another user. Please refresh and retry.",
                request.getRequestURI()));
    }

    // JPA EntityNotFoundException (lazy load of deleted entity)
    @ExceptionHandler(jakarta.persistence.EntityNotFoundException.class)
    public ResponseEntity<ApiError> handleEntityNotFound(
            jakarta.persistence.EntityNotFoundException ex,
            HttpServletRequest request) {

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiError.of(HttpStatus.NOT_FOUND, "Entity not found", request.getRequestURI()));
    }
}
```

### 9.2 How @Repository Exception Translation Works

```java
// Spring uses PersistenceExceptionTranslationPostProcessor to wrap
// @Repository beans with a proxy that translates exceptions.

// Without @Repository:
@Component
public class RawDao {
    public User find(Long id) {
        return entityManager.find(User.class, id);
        // Throws jakarta.persistence.PersistenceException — NOT translated
    }
}

// With @Repository:
@Repository
public class UserRepository {
    public User find(Long id) {
        return entityManager.find(User.class, id);
        // Throws DataAccessException (Spring) — translated and unified
    }
}

// Spring Data JPA repositories automatically get @Repository behavior
public interface UserJpaRepository extends JpaRepository<User, Long> {
    // All exceptions are translated automatically
}
```

### 9.3 Custom JPA Exception Handling

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public Product findBySku(String sku) {
        return productRepository.findBySku(sku)
            .orElseThrow(() -> ResourceNotFoundException.product(sku));
    }

    @Transactional
    public Product update(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> ResourceNotFoundException.product(id.toString()));

        // Optimistic lock check
        if (!product.getVersion().equals(request.getVersion())) {
            throw new ResourceConflictException(
                "Product was modified since you last fetched it. " +
                "Current version: " + product.getVersion() + ", your version: " + request.getVersion()
            );
        }

        product.update(request);
        return productRepository.save(product);
    }
}
```

---

## 10. HTTP Client Exceptions

### 10.1 RestTemplate Exception Handling

```java
// RestTemplate throws HttpClientErrorException (4xx) and HttpServerErrorException (5xx)
@Service
public class ExternalUserService {

    private final RestTemplate restTemplate;

    public ExternalUserDto fetchUser(Long userId) {
        try {
            return restTemplate.getForObject(
                "https://api.external.com/users/{id}",
                ExternalUserDto.class,
                userId
            );
        } catch (HttpClientErrorException.NotFound e) {
            throw ResourceNotFoundException.user(userId);
        } catch (HttpClientErrorException.Unauthorized e) {
            throw new AuthenticationException("External API authentication failed");
        } catch (HttpClientErrorException e) {
            throw new ExternalServiceException("UserAPI",
                "Client error: " + e.getStatusCode(), e);
        } catch (HttpServerErrorException e) {
            throw new ExternalServiceException("UserAPI",
                "Server error: " + e.getStatusCode(), e);
        } catch (ResourceAccessException e) {
            // Network error, timeout
            throw new ExternalServiceException("UserAPI", "Connection failed", e);
        }
    }
}

// Custom ResponseErrorHandler for RestTemplate
@Component
public class CustomResponseErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().isError();
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        HttpStatusCode status = response.getStatusCode();
        String body = StreamUtils.copyToString(response.getBody(), StandardCharsets.UTF_8);

        if (status.is4xxClientError()) {
            if (status == HttpStatus.NOT_FOUND) {
                throw new ResourceNotFoundException("External resource", "unknown");
            }
            throw new HttpClientErrorException(status, body);
        } else if (status.is5xxServerError()) {
            throw new ExternalServiceException("External", "Server error: " + body, null);
        }
    }
}

// Register the error handler
@Bean
public RestTemplate restTemplate() {
    RestTemplate rt = new RestTemplate();
    rt.setErrorHandler(new CustomResponseErrorHandler());
    return rt;
}
```

### 10.2 WebClient Error Handling

```java
@Service
public class ReactiveExternalService {

    private final WebClient webClient;

    public Mono<ExternalDto> fetchData(String id) {
        return webClient.get()
            .uri("/api/data/{id}", id)
            .retrieve()
            // Specific status handlers
            .onStatus(HttpStatusCode::is4xxClientError, response ->
                response.bodyToMono(String.class)
                    .map(body -> {
                        if (response.statusCode() == HttpStatus.NOT_FOUND) {
                            return new ResourceNotFoundException("Data", id);
                        }
                        return new ExternalServiceException("DataAPI", body, null);
                    })
            )
            .onStatus(HttpStatusCode::is5xxServerError, response ->
                response.bodyToMono(String.class)
                    .map(body -> new ExternalServiceException("DataAPI",
                        "Server error: " + body, null))
            )
            .bodyToMono(ExternalDto.class)
            // Handle network errors
            .onErrorMap(WebClientRequestException.class,
                e -> new ExternalServiceException("DataAPI", "Connection failed", e))
            // Retry on transient errors
            .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
                .filter(e -> e instanceof ExternalServiceException)
                .onRetryExhaustedThrow((spec, signal) ->
                    new ExternalServiceException("DataAPI", "Max retries exceeded", signal.failure())));
    }

    // Global WebClient error handling via ExchangeFilterFunction
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .baseUrl("https://api.external.com")
            .filter(errorHandlingFilter())
            .build();
    }

    private ExchangeFilterFunction errorHandlingFilter() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            if (response.statusCode().isError()) {
                return response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(
                        new ExternalServiceException("WebClient",
                            "Error " + response.statusCode() + ": " + body, null)));
            }
            return Mono.just(response);
        });
    }
}
```

### 10.3 Feign Client Exception Handling

```java
// Feign client declaration
@FeignClient(
    name = "user-service",
    url = "${services.user.url}",
    configuration = UserFeignConfig.class
)
public interface UserFeignClient {

    @GetMapping("/api/users/{id}")
    UserDto getUser(@PathVariable Long id);

    @PostMapping("/api/users")
    UserDto createUser(@RequestBody CreateUserRequest request);
}

// Custom error decoder
@Configuration
public class UserFeignConfig {

    @Bean
    public ErrorDecoder errorDecoder() {
        return new UserServiceErrorDecoder();
    }
}

public class UserServiceErrorDecoder implements ErrorDecoder {

    private final ErrorDecoder defaultDecoder = new ErrorDecoder.Default();
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Exception decode(String methodKey, Response response) {
        String body = "";
        try {
            if (response.body() != null) {
                body = new String(response.body().asInputStream().readAllBytes());
            }
        } catch (IOException ignored) {}

        return switch (response.status()) {
            case 400 -> new InvalidInputException("Bad request to user-service: " + body);
            case 401 -> new AuthenticationException("Unauthorized access to user-service");
            case 403 -> new AuthorizationException("Forbidden access to user-service");
            case 404 -> new ResourceNotFoundException("User", "unknown");
            case 409 -> new ResourceAlreadyExistsException("User", body);
            case 429 -> new RateLimitExceededException("user-service");
            default -> {
                if (response.status() >= 500) {
                    yield new ExternalServiceException("user-service",
                        "Server error: " + response.status() + " " + body, null);
                }
                yield defaultDecoder.decode(methodKey, response);
            }
        };
    }
}

// Feign retry configuration
@Bean
public Retryer retryer() {
    // Retry on RetryableException (5xx), up to 3 times, with 100ms interval
    return new Retryer.Default(100, 1000, 3);
}
```

---

## 11. Async Exception Handling

### 11.1 Exceptions in @Async Methods

> [!WARNING]
> Exceptions thrown in `@Async` methods are **NOT propagated to the caller thread**. The method returns immediately, and exceptions are silently swallowed unless you configure an `AsyncUncaughtExceptionHandler`.

```java
@Service
public class NotificationService {

    @Async
    public void sendEmail(String to, String subject, String body) {
        // If this throws, the caller NEVER knows about it
        emailSender.send(to, subject, body);
    }

    // Return CompletableFuture if you need exception propagation
    @Async
    public CompletableFuture<Void> sendEmailAsync(String to, String subject, String body) {
        try {
            emailSender.send(to, subject, body);
            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}

// Caller side for CompletableFuture
@Service
public class UserService {

    public void registerUser(User user) {
        userRepository.save(user);

        // Exception propagation via CompletableFuture
        notificationService.sendEmailAsync(user.getEmail(), "Welcome!", "...")
            .exceptionally(ex -> {
                log.error("Failed to send welcome email to {}: {}", user.getEmail(), ex.getMessage());
                return null; // Handle gracefully
            });
    }
}
```

### 11.2 AsyncUncaughtExceptionHandler

```java
// Configure a global handler for @Async void method exceptions
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomAsyncExceptionHandler.class);

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error(
            "Async exception in method '{}' with params {}: {}",
            method.getName(),
            Arrays.toString(params),
            ex.getMessage(),
            ex
        );

        // Optionally: publish an event, send to dead letter queue, alert monitoring
        if (ex instanceof ExternalServiceException) {
            // Retry logic, dead letter handling
        }
    }
}
```

### 11.3 CompletableFuture Exception Propagation

```java
@Service
public class DataProcessingService {

    @Async
    public CompletableFuture<ProcessResult> processData(DataPayload payload) {
        return CompletableFuture.supplyAsync(() -> {
            validatePayload(payload);          // May throw ValidationException
            ProcessResult result = process(payload); // May throw ProcessingException
            return result;
        })
        .exceptionally(ex -> {
            log.error("Processing failed for payload {}: {}", payload.getId(), ex.getMessage(), ex);
            throw new CompletionException(ex); // Wrap and re-throw
        });
    }
}

// Handling CompletableFuture results in controller
@RestController
public class DataController {

    @PostMapping("/process")
    public ResponseEntity<ProcessResult> process(@RequestBody DataPayload payload)
            throws ExecutionException, InterruptedException {
        try {
            ProcessResult result = dataProcessingService.processData(payload).get();
            return ResponseEntity.ok(result);
        } catch (ExecutionException e) {
            // Unwrap the cause
            Throwable cause = e.getCause();
            if (cause instanceof ValidationException ve) {
                throw ve; // Let @ExceptionHandler handle it
            }
            throw new ProcessingException("Processing failed", cause);
        }
    }
}
```

---

## 12. Scheduled Task Exception Handling

### 12.1 What Happens When @Scheduled Throws?

> [!WARNING]
> By default, if an exception escapes from a `@Scheduled` method, **the task IS NOT killed** — Spring catches the exception and logs it. However, if using a custom `TaskScheduler` that isn't configured properly, the thread pool could get depleted.

```java
@Component
@Slf4j
public class ReportGenerationJob {

    @Scheduled(cron = "0 0 2 * * *")  // Daily at 2 AM
    public void generateDailyReport() {
        log.info("Starting daily report generation");
        try {
            reportService.generate();
            log.info("Daily report generated successfully");
        } catch (Exception e) {
            // Catch ALL exceptions to prevent task from "dying"
            log.error("Failed to generate daily report", e);
            alertService.sendAlert("Report generation failed: " + e.getMessage());
        }
    }
}
```

### 12.2 Global ErrorHandler for Scheduled Tasks

```java
@Configuration
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setErrorHandler(new ScheduledTaskErrorHandler());
        scheduler.initialize();

        taskRegistrar.setTaskScheduler(scheduler);
    }
}

public class ScheduledTaskErrorHandler implements ErrorHandler {

    private static final Logger log = LoggerFactory.getLogger(ScheduledTaskErrorHandler.class);

    @Override
    public void handleError(Throwable t) {
        log.error("Scheduled task failed: {}", t.getMessage(), t);

        // Notify monitoring system
        // Send to alerting (PagerDuty, OpsGenie, etc.)
        // Push metrics to Micrometer
    }
}
```

### 12.3 Preventing Task Starvation

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);  // Multiple threads to avoid starvation
        scheduler.setThreadNamePrefix("sched-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(60);
        scheduler.setErrorHandler(t ->
            LoggerFactory.getLogger(SchedulingConfig.class)
                .error("Scheduled task error: {}", t.getMessage(), t));
        return scheduler;
    }
}

@Component
public class SafeScheduledTask {

    @Scheduled(fixedDelay = 5000, initialDelay = 1000)
    public void runSafely() {
        try {
            doWork();
        } catch (Exception e) {
            // Log and continue — do NOT re-throw
            log.error("Task failed, will retry in 5s", e);
        }
    }
}
```

---

## 13. Logging Best Practices

### 13.1 What to Log in Exception Handlers

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex,
                                                    HttpServletRequest request) {
        // WARN level for expected business errors — no stack trace needed
        log.warn("Resource not found | uri={} correlationId={} error={}",
            request.getRequestURI(),
            MDC.get("correlationId"),
            ex.getMessage());

        return buildResponse(HttpStatus.NOT_FOUND, ex, request);
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ApiError> handleDataIntegrity(DataIntegrityViolationException ex,
                                                         HttpServletRequest request) {
        // ERROR with exception for unexpected data errors
        log.error("Data integrity violation | uri={} correlationId={}",
            request.getRequestURI(),
            MDC.get("correlationId"),
            ex);  // Pass exception as last argument for stack trace

        return buildResponse(HttpStatus.CONFLICT, "Data constraint violated", request);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleAll(Exception ex, HttpServletRequest request) {
        // Always log full stack trace for unexpected errors
        log.error("Unexpected error | method={} uri={} correlationId={} user={}",
            request.getMethod(),
            request.getRequestURI(),
            MDC.get("correlationId"),
            getCurrentUsername(),
            ex);  // Stack trace included

        return buildResponse(HttpStatus.INTERNAL_SERVER_ERROR, "Internal error", request);
    }
}
```

### 13.2 Correlation ID with MDC

```java
// Filter to inject correlation ID into every request
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter implements Filter {

    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    public static final String CORRELATION_ID_MDC_KEY = "correlationId";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String correlationId = Optional
            .ofNullable(httpRequest.getHeader(CORRELATION_ID_HEADER))
            .filter(id -> !id.isBlank())
            .orElse(UUID.randomUUID().toString());

        MDC.put(CORRELATION_ID_MDC_KEY, correlationId);
        httpResponse.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_MDC_KEY);  // Critical: clean up MDC
        }
    }
}
```

### 13.3 Structured Logging

```java
// Use structured logging for easier log aggregation (ELK, Splunk, Datadog)
// With Logback + logstash-logback-encoder or similar

// logback-spring.xml
// <appender name="JSON" class="net.logstash.logback.appender.LogstashConsoleAppender">
//     <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
// </appender>

// Produces structured JSON logs:
// {
//   "@timestamp": "2024-01-15T10:30:00.000Z",
//   "level": "ERROR",
//   "logger": "GlobalExceptionHandler",
//   "message": "Unexpected error",
//   "correlationId": "abc-123",
//   "uri": "/api/users/999",
//   "stack_trace": "..."
// }

@ExceptionHandler(Exception.class)
public ResponseEntity<ApiError> handleAll(Exception ex, HttpServletRequest request) {
    // Structured key-value pairs are automatically included if using MDC
    // and logstash-logback-encoder
    log.error("Unexpected error | status=500 uri={}", request.getRequestURI(), ex);
    return buildResponse(HttpStatus.INTERNAL_SERVER_ERROR, "Internal error", request);
}
```

### 13.4 Avoiding Log Spam

```java
// Problem: A bug in a polling loop can generate millions of log lines
// Solution: Rate limiting, sampling, or circuit breaker for logging

@Component
public class RateLimitedLogger {

    private final Map<String, Instant> lastLoggedAt = new ConcurrentHashMap<>();
    private static final Duration LOG_INTERVAL = Duration.ofMinutes(1);

    public void logError(String key, Supplier<String> messageSupplier, Throwable ex) {
        Instant now = Instant.now();
        Instant lastLogged = lastLoggedAt.get(key);

        if (lastLogged == null || Duration.between(lastLogged, now).compareTo(LOG_INTERVAL) > 0) {
            lastLoggedAt.put(key, now);
            log.error(messageSupplier.get(), ex);
        }
    }
}

// Better: Use Logback's DuplicateMessageFilter
// In logback-spring.xml:
// <turboFilter class="ch.qos.logback.classic.turbo.DuplicateMessageFilter">
//     <AllowedRepetitions>5</AllowedRepetitions>
// </turboFilter>
```

---

## 14. ⚠️ Interview Traps & Gotchas

### Trap 1: @ControllerAdvice Does NOT Handle Filter Exceptions

```java
// WRONG assumption: @ControllerAdvice catches ALL exceptions
@Component
public class AuthFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        throw new AuthenticationException("Token expired");
        // ↑ This NEVER reaches @ControllerAdvice!
        // Filters execute BEFORE DispatcherServlet, so @ExceptionHandler
        // (which is registered with DispatcherServlet) never sees this.
    }
}
```

**Solution**: Handle exceptions in the filter itself, or delegate to `/error`:

```java
@Component
public class AuthFilter extends OncePerRequestFilter {

    private final HandlerExceptionResolver resolver;

    public AuthFilter(@Qualifier("handlerExceptionResolver") HandlerExceptionResolver resolver) {
        this.resolver = resolver;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        try {
            chain.doFilter(request, response);
        } catch (AuthenticationException ex) {
            // Delegate to Spring MVC exception resolution
            resolver.resolveException(request, response, null, ex);
        }
    }
}
```

### Trap 2: Checked Exceptions and @Transactional

```java
// BUG: Transaction commits even on exception!
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount)
        throws InsufficientFundsException {  // Checked exception!

    Account from = accountRepo.findById(fromId).orElseThrow();
    Account to = accountRepo.findById(toId).orElseThrow();

    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException("Insufficient funds");
        // TRANSACTION COMMITS! from is saved but transfer didn't complete.
    }

    from.debit(amount);
    to.credit(amount);
}

// FIX 1: rollbackFor
@Transactional(rollbackFor = InsufficientFundsException.class)
public void transfer(...) throws InsufficientFundsException { ... }

// FIX 2: Make exception unchecked
public class InsufficientFundsException extends RuntimeException { ... }
```

### Trap 3: @ExceptionHandler in @Async — Does NOT Work!

```java
// This is a COMMON interview question:
// Does @ControllerAdvice catch exceptions from @Async methods?

@Service
public class EmailService {
    @Async
    public void sendEmail(String to) {
        throw new RuntimeException("SMTP error");
        // This runs on a different thread!
        // @ControllerAdvice CANNOT catch this — it's not in the controller thread
    }
}

@RestControllerAdvice
public class GlobalHandler {
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<ApiError> handle(RuntimeException ex) {
        // This will NEVER be called for @Async exceptions
        return ResponseEntity.internalServerError().build();
    }
}

// SOLUTION: Use AsyncUncaughtExceptionHandler (see Section 11.2)
```

### Trap 4: Multiple @ControllerAdvice — Priority

```java
// Which handler runs first?

@RestControllerAdvice
@Order(1)
public class SpecificHandler {
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<ApiError> handle(RuntimeException ex) {
        return ResponseEntity.badRequest().build(); // This wins — Order(1) = highest priority
    }
}

@RestControllerAdvice
@Order(2)
public class GeneralHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handle(Exception ex) {
        return ResponseEntity.internalServerError().build(); // This loses
    }
}

// RULE: Lower @Order value = higher priority.
// Also: local @ExceptionHandler ALWAYS beats @ControllerAdvice, regardless of @Order.
```

### Trap 5: @ResponseStatus on Exception Class vs @ExceptionHandler

```java
// Method 1: @ResponseStatus on exception class
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User " + id + " not found");
    }
}

// Method 2: @ExceptionHandler in advice
@ExceptionHandler(UserNotFoundException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
public ApiError handle(UserNotFoundException ex) {
    return new ApiError(ex.getMessage());
}

// GOTCHA: If you have BOTH:
// - A @ResponseStatus on the exception class
// - AND an @ExceptionHandler for the same exception
// → The @ExceptionHandler WINS and completely overrides @ResponseStatus on the class.

// Another GOTCHA with @ResponseStatus alone:
// The Whitelabel error page / default error body is used, NOT a custom JSON body.
// For custom JSON: always use @ExceptionHandler.
```

### Trap 6: ResponseEntityExceptionHandler — Why Extend It?

```java
// If you DON'T extend ResponseEntityExceptionHandler:
// Spring's DefaultHandlerExceptionResolver handles standard MVC exceptions
// and returns ONLY the HTTP status with an empty body — no JSON!

// GET /api/users/abc → TypeMismatchException → 400 with NO body

// If you DO extend ResponseEntityExceptionHandler:
// You get overridable handlers for ALL standard MVC exceptions
// and can return a proper JSON body.

// Common mistake: Forgetting to extend and wondering why validation errors
// return 400 with no body.
```

### Trap 7: Global vs Local Exception Handler Precedence

```java
@RestControllerAdvice  // Global
public class GlobalHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handleGlobal(UserNotFoundException ex) {
        return ResponseEntity.status(404).body(new ApiError("Global handler"));
    }
}

@RestController
public class UserController {

    // LOCAL handler ALWAYS takes precedence over global
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handleLocal(UserNotFoundException ex) {
        return ResponseEntity.status(404).body(new ApiError("Local handler"));  // This wins!
    }
}
```

### Trap 8: Exception Thrown During View Rendering

```java
// @ExceptionHandler catches exceptions thrown in the controller method itself.
// But if the exception is thrown during VIEW RENDERING (Thymeleaf template error),
// it is NOT caught by @ExceptionHandler!

// Solution: Handle via BasicErrorController / error view templates,
// or use HandlerExceptionResolverComposite.
```

### Trap 9: @Transactional + @ExceptionHandler Order

```java
// Service throws exception → @Transactional proxy catches it → marks for rollback
// → re-throws → @ExceptionHandler in controller catches it for HTTP response

// By the time @ExceptionHandler runs, the transaction is ALREADY rolled back.
// You CANNOT un-rollback inside @ExceptionHandler.

// The exception handler is for HTTP response only, not for transaction control.
```

### Trap 10: AOP Proxy and Exception Wrapping

```java
// Spring AOP wraps exceptions from @Transactional in UndeclaredThrowableException
// when a checked exception is thrown but NOT declared in the method signature.

@Transactional
public void doSomething() {
    throw new SQLException("DB error"); // Checked, NOT declared
    // Spring wraps in UndeclaredThrowableException!
}

// To unwrap:
@ExceptionHandler(UndeclaredThrowableException.class)
public ResponseEntity<ApiError> handleUndeclared(UndeclaredThrowableException ex) {
    Throwable cause = ex.getCause();
    if (cause instanceof SQLException sqle) {
        // Handle appropriately
    }
    ...
}
```

---

## 15. 🔍 Deep Dive — Internals

### 15.1 HandlerExceptionResolver Chain

When an exception escapes a controller, `DispatcherServlet` passes it through a chain of `HandlerExceptionResolver` implementations:

```
1. ExceptionHandlerExceptionResolver
   → Handles @ExceptionHandler methods (local + @ControllerAdvice)
   → Highest priority by default (order = 0)

2. ResponseStatusExceptionResolver
   → Handles @ResponseStatus on exception classes
   → order = 1

3. DefaultHandlerExceptionResolver
   → Handles Spring MVC standard exceptions (TypeMismatchException, etc.)
   → Returns empty body with HTTP status
   → order = 2

4. If none resolved → Exception propagates to the Servlet container
   → Servlet container routes to /error
   → BasicErrorController handles it
```

```java
// You can see all resolvers with:
@Autowired
private List<HandlerExceptionResolver> resolvers;

// In @PostConstruct:
// 1. DefaultErrorAttributes (order=-1, from spring-boot-autoconfigure)
// 2. ExceptionHandlerExceptionResolver (order=0)
// 3. ResponseStatusExceptionResolver (order=1)
// 4. DefaultHandlerExceptionResolver (order=2)
```

### 15.2 How @ExceptionHandler is Discovered

```
Application startup:
1. ExceptionHandlerExceptionResolver is initialized
2. It scans all beans for @ControllerAdvice
3. For each @ControllerAdvice, it creates an ExceptionHandlerMethodResolver
4. ExceptionHandlerMethodResolver builds a map: ExceptionType → Method

At runtime (exception thrown):
1. ExceptionHandlerExceptionResolver finds the right handler:
   a. Checks the specific controller first (local @ExceptionHandler)
   b. Then checks @ControllerAdvice beans in @Order order
   c. Most specific exception type wins (closest in inheritance hierarchy)
2. Invokes the handler method via reflection
3. Processes the return value (ResponseEntity, @ResponseBody, etc.)
```

### 15.3 Exception Specificity Rules

```java
// If you have:
@ExceptionHandler(RuntimeException.class)   // More general
public ResponseEntity<ApiError> handleRuntime(RuntimeException ex) { ... }

@ExceptionHandler(IllegalArgumentException.class)  // More specific
public ResponseEntity<ApiError> handleIllegal(IllegalArgumentException ex) { ... }

// And IllegalArgumentException is thrown:
// → handleIllegal() is called (most specific wins)

// Even if handleRuntime() is in a @Order(1) advice and handleIllegal() is in @Order(2):
// Specificity > Order within the same advice class.
// Between different advice classes: Order determines which advice is checked first.
```

### 15.4 ProblemDetail Internals (Spring 6)

```java
// ProblemDetail is backed by a Map internally
// spring.mvc.problemdetails.enabled=true makes ResponseEntityExceptionHandler
// return ProblemDetail instead of empty body

// The ResponseEntityExceptionHandler in Spring 6 was refactored to return
// ProblemDetail for all built-in exceptions when enabled.

// Custom properties are stored in a LinkedHashMap
ProblemDetail detail = ProblemDetail.forStatus(404);
detail.setProperty("custom", "value");  // Goes into the map
// JSON: { "type": "about:blank", "status": 404, "custom": "value" }
```

---

## 16. 📝 Summary & Key Takeaways

| Concept | Key Point |
|---------|-----------|
| **Exception types** | Use unchecked exceptions in Spring; checked exceptions bypass `@Transactional` rollback |
| **@ExceptionHandler** | Controller-local; handles exceptions from that controller only |
| **@ControllerAdvice** | Global; use `@Order` for priority; extend `ResponseEntityExceptionHandler` |
| **@RestControllerAdvice** | Same as `@ControllerAdvice + @ResponseBody`; use for REST APIs |
| **Local vs Global** | Local `@ExceptionHandler` ALWAYS beats `@ControllerAdvice` |
| **Filter exceptions** | NOT caught by `@ControllerAdvice`; handle in filter or via `HandlerExceptionResolver` |
| **@Async exceptions** | NOT propagated to caller; use `AsyncUncaughtExceptionHandler` |
| **@Transactional rollback** | Only on `RuntimeException/Error` by default; use `rollbackFor` for checked |
| **ProblemDetail** | Spring Boot 3 native RFC 7807 support via `spring.mvc.problemdetails.enabled=true` |
| **Validation errors** | `MethodArgumentNotValidException` (body) vs `ConstraintViolationException` (params) |
| **@Repository** | Auto-translates JPA/Hibernate exceptions to Spring `DataAccessException` hierarchy |
| **Correlation ID** | Always add to MDC at filter level; include in error responses |
| **Error response** | Standardize: status, errorCode, message, path, timestamp, correlationId |

### Architecture Recommendation

```
Filter Layer:
  - CorrelationIdFilter (inject MDC)
  - AuthFilter (handle auth errors directly)

Controller Layer:
  - @Valid on request bodies
  - No try-catch (let exceptions propagate)

Service Layer:
  - Throw domain exceptions (ResourceNotFoundException, etc.)
  - Use @Transactional(rollbackFor = Exception.class) for safety

Global Exception Handler (@RestControllerAdvice):
  - Extend ResponseEntityExceptionHandler
  - Handle ApplicationException hierarchy
  - Handle validation exceptions with field-level details
  - Catch-all Exception handler as fallback

Custom Exception Hierarchy:
  ApplicationException
  ├── BusinessException → 4xx
  └── SystemException  → 5xx
```

---

## 17. ❓ Practice Questions

### Conceptual Questions

1. **What is the difference between `@ExceptionHandler` in a `@RestController` vs `@RestControllerAdvice`?**  
   *(Answer: Scope — local controller only vs global; local always wins)*

2. **Why does `@Transactional` not roll back on checked exceptions by default? How do you fix it?**  
   *(Answer: Historical Java EE convention; fix with `rollbackFor = Exception.class`)*

3. **Does `@ControllerAdvice` catch exceptions thrown in a Servlet Filter?**  
   *(Answer: No — filters execute before DispatcherServlet; use HandlerExceptionResolver or handle in filter)*

4. **What happens when two `@ControllerAdvice` classes both have `@ExceptionHandler(RuntimeException.class)`?**  
   *(Answer: `@Order` determines priority; lower value wins; if equal, order is undefined)*

5. **You have an `@Async` method that throws a `RuntimeException`. Your `@ControllerAdvice` has a handler for it. Will it be invoked?**  
   *(Answer: No — @Async runs on a different thread pool; use `AsyncUncaughtExceptionHandler` instead)*

6. **What is `ResponseEntityExceptionHandler` and why should you extend it?**  
   *(Answer: Base class handling all Spring MVC built-in exceptions with proper JSON responses; without it, standard errors return empty bodies)*

7. **Explain the difference between `MethodArgumentNotValidException` and `ConstraintViolationException`.**  
   *(Answer: Former is from `@Valid` on `@RequestBody`, latter from `@Validated` on service method params or path/query params)*

8. **What is RFC 7807 and how does Spring Boot 3 support it?**  
   *(Answer: Standard for HTTP API error responses; Spring Boot 3 adds `ProblemDetail` class and `spring.mvc.problemdetails.enabled=true`)*

### Coding Challenges

9. **Write a `@RestControllerAdvice` that handles all common Spring Boot exceptions with proper HTTP status codes and a custom JSON error response.**

10. **Design a custom exception hierarchy for an e-commerce application with at least 5 exception types mapped to appropriate HTTP status codes.**

11. **Write a `Filter` that adds a correlation ID to every request and properly delegates exceptions to Spring MVC's exception resolution mechanism.**

12. **Implement an `AsyncUncaughtExceptionHandler` that logs the error, extracts method metadata, and sends an alert.**

13. **Given a `@Transactional` service method that throws a checked exception, demonstrate the bug and provide two fixes.**

14. **Write a `WebClient`-based external service call with proper error handling, retry logic, and exception translation.**

15. **Implement RFC 7807 `ProblemDetail` responses for your `ResourceNotFoundException` and `ValidationException` using Spring Boot 3 APIs.**

---

> [!TIP]
> **Interview tip**: Always mention these in exception handling discussions:
> 1. Correlation IDs for traceability
> 2. RFC 7807 for standardization  
> 3. The filter exception gap (not caught by `@ControllerAdvice`)
> 4. The `@Transactional` + checked exception trap
> 5. The `@Async` exception propagation gap
> These show depth of production experience and will differentiate you from candidates who only know basic `@ExceptionHandler`.
