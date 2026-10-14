# 02 — Spring Boot Annotations: The Complete Reference

> **Series:** Spring Boot Mastery | **Level:** Intermediate → Advanced  
> **Spring Boot Version:** 3.x / Spring Framework 6.x  
> **Estimated Reading Time:** ~60 minutes

---

## Table of Contents

1. [Introduction — Why Annotations Matter](#1-introduction)
2. [Stereotype Annotations](#2-stereotype-annotations)
3. [Dependency Injection Annotations](#3-dependency-injection-annotations)
4. [Configuration Annotations](#4-configuration-annotations)
5. [Web / REST Annotations](#5-web--rest-annotations)
6. [Data / JPA Annotations](#6-data--jpa-annotations)
7. [Exception Handling Annotations](#7-exception-handling-annotations)
8. [Scheduling & Async Annotations](#8-scheduling--async-annotations)
9. [Caching Annotations](#9-caching-annotations)
10. [Security Annotations](#10-security-annotations)
11. [Testing Annotations](#11-testing-annotations)
12. [Lifecycle Annotations](#12-lifecycle-annotations)
13. [Conditional Annotations](#13-conditional-annotations)
14. [Validation Annotations](#14-validation-annotations)
15. [⚠️ Interview Traps & Gotchas — The Proxy Problem](#15-️-interview-traps--gotchas)
16. [🔍 Deep Dive — How Spring AOP Proxies Work](#16--deep-dive--how-spring-aop-proxies-work)
17. [📝 Summary & Key Takeaways](#17--summary--key-takeaways)
18. [❓ Practice Questions](#18--practice-questions)

---

## 1. Introduction

Spring Boot's annotation-driven programming model is the backbone of every Spring application. Annotations are not just syntax sugar — they are **metadata instructions** that the Spring container processes at startup, through **reflection**, **classpath scanning**, **proxy generation**, and **aspect weaving**.

Understanding *why* an annotation works (or fails) requires understanding three core mechanisms:

| Mechanism | What It Powers |
|---|---|
| **Classpath Scanning** | `@Component`, `@Service`, `@Repository`, `@Controller` |
| **Proxy-based AOP** | `@Transactional`, `@Async`, `@Cacheable`, `@Scheduled` |
| **Reflection + `BeanDefinition`** | `@Bean`, `@Configuration`, `@Import` |

> [!IMPORTANT]
> Spring AOP is **proxy-based**, not bytecode-level (unlike AspectJ full weaving). This single fact explains the majority of annotation gotchas. When you call a method *within the same class*, you bypass the proxy, and the annotation has **no effect**.

---

## 2. Stereotype Annotations

Stereotype annotations tell Spring's component scanner that a class should be registered as a **Spring-managed bean**. They all ultimately derive from `@Component`.

### 2.1 `@Component`

**Purpose:** Generic stereotype — marks any class as a Spring-managed bean. It is the root annotation for all other stereotype annotations.

```java
import org.springframework.stereotype.Component;

@Component
public class EmailValidator {

    public boolean isValid(String email) {
        return email != null && email.contains("@");
    }
}
```

**Internals:**  
During startup, `ClassPathBeanDefinitionScanner` scans the classpath for classes annotated with `@Component` (and its meta-annotations). Each such class gets a `BeanDefinition` registered in the `BeanDefinitionRegistry`.

**Bean Naming:**  
By default, the bean name is the simple class name with the first letter lowercased:

```java
@Component("mySpecialValidator")  // override default name
public class EmailValidator { ... }
```

---

### 2.2 `@Service`

**Purpose:** Semantic marker for the **service layer** (business logic). Functionally identical to `@Component` — it adds **no new behavior** in Spring itself, but it:

1. Communicates **intent** to developers
2. Is a recognized target for AOP pointcuts in some frameworks
3. Some tools (e.g., Spring Data REST, certain monitoring agents) treat it differently

```java
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    // Constructor injection (preferred)
    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    public Order placeOrder(OrderRequest request) {
        Order order = new Order(request);
        paymentService.charge(request.getPaymentDetails());
        return orderRepository.save(order);
    }
}
```

> [!NOTE]
> `@Service` is **not** special in Spring core — it is simply `@Component` with a different name. The distinction is **architectural** and enables better code navigation and tooling support.

---

### 2.3 `@Repository`

**Purpose:** Marks a class as a **data access object (DAO)**. Unlike `@Component` and `@Service`, `@Repository` **does** have one unique behavioral difference: it enables **exception translation**.

**Exception Translation:**  
Spring wraps technology-specific exceptions (e.g., `SQLException`, Hibernate's `HibernateException`) into Spring's unified `DataAccessException` hierarchy. This is powered by `PersistenceExceptionTranslationPostProcessor`, which is automatically registered when you use `@Repository`.

```java
import org.springframework.stereotype.Repository;
import org.springframework.dao.DataAccessException;

@Repository
public class ProductRepositoryImpl implements ProductRepository {

    private final JdbcTemplate jdbcTemplate;

    public ProductRepositoryImpl(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Product findById(Long id) {
        // If a SQLException is thrown here, Spring translates it
        // to a DataAccessException automatically
        return jdbcTemplate.queryForObject(
            "SELECT * FROM products WHERE id = ?",
            new ProductRowMapper(),
            id
        );
    }
}
```

**Why This Matters:**  
Without `@Repository`, a `ConstraintViolationException` from your database driver leaks through to your service layer. With `@Repository`, it becomes `DataIntegrityViolationException` — a Spring-managed, portable exception.

> [!WARNING]
> When using Spring Data JPA's `JpaRepository` interface, exception translation is handled automatically — you do **not** need to add `@Repository` explicitly (Spring Data adds it under the hood). However, for custom DAO classes, you must add it manually.

---

### 2.4 `@Controller`

**Purpose:** Marks a class as a Spring MVC **web controller**. Handles HTTP requests and typically returns **view names** (for Thymeleaf, JSP, etc.) or data models.

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("message", "Welcome!");
        return "home"; // Resolves to templates/home.html (Thymeleaf)
    }
}
```

**Key Distinction:** Methods in `@Controller` return **view names** by default. To return data directly from a method (JSON/XML), you must also annotate the method with `@ResponseBody`.

---

### 2.5 `@RestController`

**Purpose:** A convenience annotation that combines `@Controller` + `@ResponseBody`. Every method in a `@RestController` automatically serializes its return value to the HTTP response body (as JSON by default).

```java
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.GetMapping;

@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/users/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        // Return value is automatically serialized to JSON
        return userService.findById(id);
    }
}
```

**Stereotype Annotation Comparison Table:**

| Annotation | Extends | Layer | Exception Translation | Return Type Default |
|---|---|---|---|---|
| `@Component` | — | Any | No | N/A |
| `@Service` | `@Component` | Business Logic | No | N/A |
| `@Repository` | `@Component` | Data Access | **Yes** | N/A |
| `@Controller` | `@Component` | Presentation | No | View name (String) |
| `@RestController` | `@Controller` + `@ResponseBody` | REST API | No | HTTP response body |

---

## 3. Dependency Injection Annotations

### 3.1 `@Autowired`

**Purpose:** Instructs Spring to **inject a dependency** automatically. Can be applied to:
- Constructor parameters (preferred)
- Setter methods
- Fields (least preferred — hinders testing)

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class NotificationService {

    // Field injection (avoid in production code)
    @Autowired
    private EmailSender emailSender;

    // Setter injection
    @Autowired
    public void setSmsSender(SmsSender smsSender) {
        this.smsSender = smsSender;
    }
    private SmsSender smsSender;

    // Constructor injection (PREFERRED - immutable, testable)
    private final PushNotificationSender pushSender;

    @Autowired // Optional since Spring 4.3 when there's only one constructor
    public NotificationService(PushNotificationSender pushSender) {
        this.pushSender = pushSender;
    }
}
```

**Resolution Order:**  
Spring resolves `@Autowired` using the following strategy:
1. **By type** — finds all beans matching the field/parameter type
2. If multiple matches: **by name** — uses the field/parameter name as a hint
3. If still ambiguous: **throws `NoUniqueBeanDefinitionException`**

**`required = false`:**
```java
@Autowired(required = false)
private OptionalFeature optionalFeature; // null if not present in context
```

> [!TIP]
> In Spring Boot 3.x / Spring 6.x, if a class has **exactly one constructor**, `@Autowired` is optional — Spring automatically uses that constructor for injection. This is the recommended style.

---

### 3.2 `@Qualifier`

**Purpose:** Disambiguates when multiple beans of the same type exist. Works alongside `@Autowired`.

```java
public interface MessageSender {
    void send(String message);
}

@Component("emailSender")
public class EmailMessageSender implements MessageSender { ... }

@Component("smsSender")
public class SmsMessageSender implements MessageSender { ... }

@Service
public class AlertService {

    private final MessageSender sender;

    public AlertService(@Qualifier("smsSender") MessageSender sender) {
        this.sender = sender;
    }
}
```

**Custom Qualifier Annotation:**  
For cleaner code, define your own qualifier annotations:

```java
import org.springframework.beans.factory.annotation.Qualifier;
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface SmsChannel { }

// Usage
@SmsChannel
@Component
public class SmsMessageSender implements MessageSender { ... }

@Service
public class AlertService {
    public AlertService(@SmsChannel MessageSender sender) { ... }
}
```

---

### 3.3 `@Primary`

**Purpose:** When multiple beans of the same type exist, marks one as the **default** bean to inject when no `@Qualifier` is specified.

```java
@Primary
@Component
public class DefaultPaymentProcessor implements PaymentProcessor {
    // This is injected when no @Qualifier is specified
}

@Component
public class StripePaymentProcessor implements PaymentProcessor { ... }

@Service
public class CheckoutService {
    @Autowired
    private PaymentProcessor processor; // DefaultPaymentProcessor is injected
}
```

**`@Primary` vs `@Qualifier` Precedence:**

| Scenario | Winner |
|---|---|
| Only `@Primary` defined | `@Primary` bean wins |
| Only `@Qualifier` defined | `@Qualifier` bean wins |
| Both `@Primary` and `@Qualifier` defined | **`@Qualifier` wins** (more specific) |
| Neither defined, multiple candidates | `NoUniqueBeanDefinitionException` |

---

### 3.4 `@Resource`

**Purpose:** A **JSR-250** annotation (from `javax.annotation` / `jakarta.annotation`). Injects a bean **by name first**, then by type. Part of the Java EE standard, not Spring-specific.

```java
import jakarta.annotation.Resource;

@Service
public class ReportService {

    @Resource(name = "pdfRenderer")
    private Renderer renderer; // Injects the bean named "pdfRenderer"

    @Resource // Without name: uses field name "excelRenderer" to look up
    private Renderer excelRenderer;
}
```

**Resolution Strategy:**
1. **By name** (using the `name` attribute or field name)
2. **By type** (if name lookup fails)

---

### 3.5 `@Inject`

**Purpose:** A **JSR-330** annotation from `jakarta.inject`. Functionally equivalent to `@Autowired` but without the `required` attribute. Promotes portability across CDI containers.

```java
import jakarta.inject.Inject;
import jakarta.inject.Named;

@Service
public class ShippingService {

    @Inject
    @Named("fedexShipper") // @Named is the JSR-330 equivalent of @Qualifier
    private Shipper shipper;
}
```

**DI Annotation Comparison:**

| Annotation | Spec | Resolution | `required` attr | `@Qualifier` equivalent |
|---|---|---|---|---|
| `@Autowired` | Spring | Type → Name | Yes (`required=false`) | `@Qualifier` |
| `@Resource` | JSR-250 | Name → Type | No | `name` attribute |
| `@Inject` | JSR-330 | Type → Name | No | `@Named` |

> [!TIP]
> In modern Spring Boot apps, prefer **constructor injection with `@Autowired` (or no annotation)** over field injection. It makes your classes easier to test and reason about. `@Resource` and `@Inject` are useful when writing code that must be portable to other DI containers (like Quarkus or Micronaut).

---

## 4. Configuration Annotations

### 4.1 `@Configuration`

**Purpose:** Marks a class as a **source of bean definitions**. Spring treats these classes specially — they are subclassed (via CGLIB) to ensure `@Bean` methods return singleton beans.

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate rt = new RestTemplate();
        rt.setRequestFactory(new HttpComponentsClientHttpRequestFactory());
        return rt;
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .setSerializationInclusion(JsonInclude.Include.NON_NULL);
    }
}
```

**`@Configuration` vs `@Component` for `@Bean` methods:**

```java
@Configuration
public class ConfigA {
    @Bean
    public ServiceA serviceA() { return new ServiceA(serviceB()); }

    @Bean
    public ServiceB serviceB() { return new ServiceB(); }
    // serviceA() calls serviceB() — CGLIB intercepts and returns the SAME bean
    // ServiceB is a singleton regardless of how many times serviceB() is called
}

@Component // "lite" mode — no CGLIB proxy
public class ConfigB {
    @Bean
    public ServiceA serviceA() { return new ServiceA(serviceB()); }

    @Bean
    public ServiceB serviceB() { return new ServiceB(); }
    // serviceA() calls serviceB() — a NEW ServiceB is created each time!
    // This is a bug if ServiceB is supposed to be a singleton
}
```

> [!WARNING]
> Using `@Component` instead of `@Configuration` for classes with inter-calling `@Bean` methods is a subtle bug. The `@Bean` methods run in **"lite" mode** — no CGLIB proxy, so each call creates a new instance.

---

### 4.2 `@Bean`

**Purpose:** Declares a method as a producer of a Spring-managed bean. The return value is registered in the ApplicationContext.

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // strength factor
    }

    @Bean(name = "tokenCache", destroyMethod = "shutdown")
    public Cache tokenCache() {
        return CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .build();
    }

    // Bean with dependencies — Spring automatically injects parameters
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

**Attributes:**
- `name` / `value`: Bean name(s)
- `initMethod`: Method to call after construction
- `destroyMethod`: Method to call before destruction (default: `""`, which means Spring auto-detects `close()` or `shutdown()`)

---

### 4.3 `@Import`

**Purpose:** Imports one or more `@Configuration` classes, `ImportSelector` implementations, or `ImportBeanDefinitionRegistrar` implementations into the current context.

```java
@Configuration
@Import({DatabaseConfig.class, CacheConfig.class, SecurityConfig.class})
public class MainConfig { }
```

**Advanced `ImportSelector`:**
```java
public class FeatureImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // Dynamically decide which configuration classes to import
        boolean isPremium = checkLicense();
        if (isPremium) {
            return new String[]{"com.example.PremiumFeatureConfig"};
        }
        return new String[]{"com.example.BasicFeatureConfig"};
    }
}

@Configuration
@Import(FeatureImportSelector.class)
public class AppConfig { }
```

---

### 4.4 `@ImportResource`

**Purpose:** Imports Spring XML configuration files into a Java-based configuration context. Useful when migrating from legacy XML-based Spring config.

```java
@Configuration
@ImportResource("classpath:legacy-beans.xml")
public class MigrationConfig { }
```

---

### 4.5 `@PropertySource`

**Purpose:** Loads properties from an external file (`.properties` or `.yaml` with additional support) into the Spring `Environment`.

```java
@Configuration
@PropertySource("classpath:database.properties")
@PropertySource(value = "file:/etc/app/secrets.properties", ignoreResourceNotFound = true)
public class DatabaseConfig {

    @Value("${db.url}")
    private String dbUrl;

    @Value("${db.username}")
    private String username;

    @Value("${db.password:default_password}") // default value
    private String password;
}
```

> [!NOTE]
> In Spring Boot, `application.properties` / `application.yml` are loaded automatically. `@PropertySource` is needed for **additional** property files or when using plain Spring (non-Boot).

---

### 4.6 `@ConfigurationProperties`

**Purpose:** Binds a **group of properties** from `application.properties`/`application.yml` to a POJO. Much safer and more powerful than `@Value` for complex configurations.

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;

@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {

    private String host;
    private int port = 25;
    private String username;
    private String password;
    private boolean ssl;
    private Map<String, String> properties = new HashMap<>();

    // Getters and setters are required (or use records in Spring Boot 3.x)
    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    // ... etc
}
```

**application.yml:**
```yaml
app:
  mail:
    host: smtp.gmail.com
    port: 587
    username: noreply@example.com
    password: secret
    ssl: true
    properties:
      mail.smtp.auth: "true"
      mail.smtp.starttls.enable: "true"
```

**Record-based (Spring Boot 3.x / Java 17+):**
```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    String host,
    int port,
    String username,
    String password,
    boolean ssl
) { }
```

---

### 4.7 `@EnableConfigurationProperties`

**Purpose:** Enables processing of `@ConfigurationProperties` classes. Required when `@ConfigurationProperties` is NOT combined with `@Component`.

```java
@Configuration
@EnableConfigurationProperties(MailProperties.class)
public class MailConfig {

    @Bean
    public JavaMailSender mailSender(MailProperties props) {
        JavaMailSenderImpl sender = new JavaMailSenderImpl();
        sender.setHost(props.getHost());
        sender.setPort(props.getPort());
        return sender;
    }
}
```

**Three ways to activate `@ConfigurationProperties`:**

| Method | How |
|---|---|
| Add `@Component` to the properties class | Works but mixes concerns |
| Use `@EnableConfigurationProperties` in a `@Configuration` class | Clean separation |
| Use `@ConfigurationPropertiesScan` on main class | Scans entire package automatically |

---

## 5. Web / REST Annotations

### 5.1 `@RequestMapping`

**Purpose:** Maps HTTP requests to handler methods or classes. The most general mapping annotation — supports all HTTP methods.

```java
@RestController
@RequestMapping("/api/v1/products") // Class-level prefix
public class ProductController {

    @RequestMapping(method = RequestMethod.GET)
    public List<Product> getAll() { ... }

    @RequestMapping(
        value = "/{id}",
        method = RequestMethod.PUT,
        consumes = MediaType.APPLICATION_JSON_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public Product update(@PathVariable Long id, @RequestBody ProductDTO dto) { ... }
}
```

---

### 5.2 HTTP Method Shortcut Annotations

These are specialized forms of `@RequestMapping`:

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping                          // GET /api/v1/orders
    public List<OrderDTO> listOrders() { ... }

    @GetMapping("/{id}")                 // GET /api/v1/orders/{id}
    public OrderDTO getOrder(@PathVariable Long id) { ... }

    @PostMapping                         // POST /api/v1/orders
    @ResponseStatus(HttpStatus.CREATED)
    public OrderDTO createOrder(@Valid @RequestBody CreateOrderRequest req) { ... }

    @PutMapping("/{id}")                 // PUT /api/v1/orders/{id}
    public OrderDTO updateOrder(
            @PathVariable Long id,
            @Valid @RequestBody UpdateOrderRequest req) { ... }

    @PatchMapping("/{id}/status")        // PATCH /api/v1/orders/{id}/status
    public OrderDTO updateStatus(
            @PathVariable Long id,
            @RequestParam OrderStatus status) { ... }

    @DeleteMapping("/{id}")              // DELETE /api/v1/orders/{id}
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteOrder(@PathVariable Long id) { ... }
}
```

| Annotation | HTTP Method | Typical Use |
|---|---|---|
| `@GetMapping` | GET | Fetch resource(s) |
| `@PostMapping` | POST | Create resource |
| `@PutMapping` | PUT | Full update (replace) |
| `@PatchMapping` | PATCH | Partial update |
| `@DeleteMapping` | DELETE | Remove resource |

---

### 5.3 Request Data Binding Annotations

```java
@RestController
@RequestMapping("/api/v1/search")
public class SearchController {

    @GetMapping
    public Page<Product> search(
            // Required path variable
            @PathVariable Long categoryId,

            // Query parameters (?keyword=laptop&page=0&size=20)
            @RequestParam(name = "keyword", required = false) String keyword,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,

            // HTTP headers
            @RequestHeader("X-Correlation-ID") String correlationId,
            @RequestHeader(value = "Accept-Language", defaultValue = "en") String lang,

            // Cookie values
            @CookieValue(value = "sessionId", required = false) String sessionId
    ) {
        // ...
        return productService.search(keyword, page, size);
    }

    @PostMapping("/bulk")
    public List<Product> bulkCreate(
            @RequestBody @Valid List<CreateProductRequest> requests
    ) {
        return productService.createAll(requests);
    }
}
```

**`@RequestParam` vs `@PathVariable`:**

| Annotation | Source | Example URL | Required by Default |
|---|---|---|---|
| `@PathVariable` | URI template | `/users/42` | Yes |
| `@RequestParam` | Query string | `/users?id=42` | Yes (can set `required=false`) |

---

### 5.4 `@RequestBody` and `@ResponseBody`

```java
// @ResponseBody tells Spring to serialize return value to response body
// (Already implied by @RestController)
@ResponseBody
@GetMapping("/info")
public Map<String, Object> getInfo() {
    return Map.of("version", "1.0", "status", "UP");
}

// @RequestBody deserializes JSON/XML request body to a Java object
@PostMapping("/users")
public ResponseEntity<UserDTO> createUser(@RequestBody @Valid CreateUserRequest req) {
    UserDTO created = userService.create(req);
    URI location = URI.create("/api/v1/users/" + created.getId());
    return ResponseEntity.created(location).body(created);
}
```

---

### 5.5 `@ResponseStatus`

**Purpose:** Sets the HTTP status code for a handler method or exception class.

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED) // 201 Created
public UserDTO createUser(@RequestBody CreateUserRequest req) { ... }

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT) // 204 No Content
public void deleteUser(@PathVariable Long id) { ... }

// On a custom exception class
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}
```

---

### 5.6 `@CrossOrigin`

**Purpose:** Configures **CORS** (Cross-Origin Resource Sharing) for specific controllers or methods.

```java
@RestController
@CrossOrigin(
    origins = {"https://myapp.com", "https://admin.myapp.com"},
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowedHeaders = {"Content-Type", "Authorization"},
    maxAge = 3600 // Preflight cache duration in seconds
)
@RequestMapping("/api/v1/public")
public class PublicController { ... }
```

> [!TIP]
> For production apps, configure CORS globally in a `WebMvcConfigurer` rather than on each controller. This avoids repetition and is easier to manage.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://myapp.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

---

## 6. Data / JPA Annotations

### 6.1 Entity Mapping Annotations

```java
import jakarta.persistence.*;

@Entity
@Table(
    name = "products",
    schema = "inventory",
    indexes = {
        @Index(name = "idx_product_sku", columnList = "sku"),
        @Index(name = "idx_product_category", columnList = "category_id")
    },
    uniqueConstraints = @UniqueConstraint(columnNames = "sku")
)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_sequence", allocationSize = 50)
    private Long id;

    @Column(name = "product_name", nullable = false, length = 255)
    private String name;

    @Column(name = "sku", unique = true, nullable = false, length = 50)
    private String sku;

    @Column(name = "price", precision = 10, scale = 2)
    private BigDecimal price;

    @Column(name = "description", columnDefinition = "TEXT")
    private String description;

    @Enumerated(EnumType.STRING) // Store enum as string, not ordinal
    @Column(name = "status", nullable = false)
    private ProductStatus status;

    @Version // Optimistic locking
    private Long version;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

**`@GeneratedValue` Strategies:**

| Strategy | Description | Best For |
|---|---|---|
| `AUTO` | JPA chooses strategy | Default, not recommended |
| `IDENTITY` | Database auto-increment | MySQL, SQL Server |
| `SEQUENCE` | Database sequence | PostgreSQL, Oracle (preferred) |
| `TABLE` | Simulates sequence in a table | Portability (slow) |
| `UUID` | UUID generation (JPA 3.1+) | Distributed systems |

> [!WARNING]
> `GenerationType.IDENTITY` disables batch inserts in Hibernate because the database must generate the ID first. Use `SEQUENCE` with `allocationSize > 1` for better performance.

---

### 6.2 Relationship Annotations

```java
@Entity
public class Order {

    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    // Many-to-One: many orders belong to one customer
    @ManyToOne(fetch = FetchType.LAZY) // ALWAYS use LAZY for ManyToOne
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    // One-to-Many: one order has many order items
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("lineNumber ASC")
    private List<OrderItem> items = new ArrayList<>();

    // One-to-One: one order has one shipping address
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "shipping_address_id", unique = true)
    private Address shippingAddress;
}

@Entity
public class Product {

    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    // Many-to-Many: products can belong to many categories
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "product_categories",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "category_id")
    )
    private Set<Category> categories = new HashSet<>();
}
```

**Fetch Type Rules:**

| Relationship | Default Fetch | Recommended |
|---|---|---|
| `@ManyToOne` | EAGER ⚠️ | **LAZY** |
| `@OneToOne` | EAGER ⚠️ | **LAZY** |
| `@OneToMany` | LAZY ✓ | LAZY |
| `@ManyToMany` | LAZY ✓ | LAZY |

> [!IMPORTANT]
> **Always use `FetchType.LAZY` unless you have a specific reason for EAGER.** EAGER loading on `@ManyToOne` is a common source of N+1 query problems in production. The default EAGER for `@ManyToOne` and `@OneToOne` is a JPA specification decision that most teams override.

---

### 6.3 `@Transactional`

**Purpose:** Marks a method (or class) as transactional. Spring wraps the method in a transaction — committing on success and rolling back on exception.

```java
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Propagation;

@Service
@Transactional(readOnly = true) // Class-level default: read-only
public class AccountService {

    private final AccountRepository accountRepository;
    private final AuditLogRepository auditLogRepository;

    // Override class-level for write operations
    @Transactional(
        propagation = Propagation.REQUIRED,        // Default: join or create new
        isolation = Isolation.SERIALIZABLE,         // Highest isolation level
        rollbackFor = {InsufficientFundsException.class},
        noRollbackFor = {AuditException.class},
        timeout = 30                                // Seconds
    )
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new AccountNotFoundException(fromId));
        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new AccountNotFoundException(toId));

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Insufficient balance");
        }

        from.debit(amount);
        to.credit(amount);

        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

**Propagation Behaviors:**

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing TX or create new |
| `REQUIRES_NEW` | Always create new TX, suspend existing |
| `SUPPORTS` | Join if TX exists, run without if not |
| `NOT_SUPPORTED` | Always run without TX, suspend existing |
| `MANDATORY` | Must join existing TX or throw exception |
| `NEVER` | Must run without TX or throw exception |
| `NESTED` | Run in nested TX (savepoint-based) |

**Isolation Levels:**

| Isolation | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | ✅ possible | ✅ possible | ✅ possible |
| `READ_COMMITTED` | ❌ prevented | ✅ possible | ✅ possible |
| `REPEATABLE_READ` | ❌ prevented | ❌ prevented | ✅ possible |
| `SERIALIZABLE` | ❌ prevented | ❌ prevented | ❌ prevented |

> [!WARNING]
> By default, `@Transactional` only rolls back on **unchecked exceptions** (`RuntimeException` and `Error`). Checked exceptions (like `IOException`) do **NOT** trigger rollback unless you specify `rollbackFor = IOException.class`.

---

### 6.4 `@Query`

**Purpose:** Defines a JPQL or native SQL query directly on a repository method.

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // JPQL query
    @Query("SELECT p FROM Product p WHERE p.category.name = :categoryName AND p.price < :maxPrice")
    List<Product> findByCategoryAndMaxPrice(
        @Param("categoryName") String categoryName,
        @Param("maxPrice") BigDecimal maxPrice
    );

    // Native SQL query
    @Query(
        value = "SELECT * FROM products p WHERE p.created_at > NOW() - INTERVAL '7 days'",
        nativeQuery = true
    )
    List<Product> findRecentProducts();

    // Modifying query
    @Modifying
    @Transactional
    @Query("UPDATE Product p SET p.status = :status WHERE p.id IN :ids")
    int updateStatusBatch(@Param("ids") List<Long> ids, @Param("status") ProductStatus status);

    // Projection query
    @Query("SELECT p.id as id, p.name as name, p.price as price FROM Product p WHERE p.status = 'ACTIVE'")
    List<ProductSummary> findActiveSummaries();
}
```

---

## 7. Exception Handling Annotations

### 7.1 `@ExceptionHandler`

**Purpose:** Handles exceptions thrown from `@Controller` or `@RestController` methods within the same controller class.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.findById(id); // may throw UserNotFoundException
    }

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleUserNotFound(UserNotFoundException ex) {
        return new ErrorResponse("USER_NOT_FOUND", ex.getMessage());
    }
}
```

**Limitation:** `@ExceptionHandler` on a controller class only handles exceptions from **that specific controller**. For global handling, use `@ControllerAdvice`.

---

### 7.2 `@ControllerAdvice` and `@RestControllerAdvice`

**Purpose:** A global exception handler that intercepts exceptions from **all** controllers. `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.

```java
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.http.ProblemDetail;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleNotFound(UserNotFoundException ex, HttpServletRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        pd.setInstance(URI.create(request.getRequestURI()));
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(fe -> errors.put(fe.getField(), fe.getDefaultMessage()));

        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation Failed");
        pd.setProperty("errors", errors);
        return pd;
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ProblemDetail handleDataIntegrity(DataIntegrityViolationException ex) {
        log.error("Data integrity violation", ex);
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT,
            "A resource with that identifier already exists"
        );
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGenericException(Exception ex) {
        log.error("Unexpected error", ex);
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred"
        );
    }
}
```

> [!TIP]
> Spring Boot 3.x introduces `ProblemDetail` (RFC 7807) as a standard error response format. Prefer it over custom error DTOs for standardized API error responses.

**Scoping `@ControllerAdvice`:**
```java
// Only applies to controllers in this package
@ControllerAdvice(basePackages = "com.example.api.v1")
public class V1ExceptionHandler { ... }

// Only applies to specific controller classes
@ControllerAdvice(assignableTypes = {UserController.class, OrderController.class})
public class SpecificExceptionHandler { ... }
```

---

## 8. Scheduling & Async Annotations

### 8.1 `@EnableScheduling` and `@Scheduled`

**Purpose:** `@EnableScheduling` activates the scheduling infrastructure. `@Scheduled` marks a method to run on a schedule.

```java
@SpringBootApplication
@EnableScheduling
public class Application { ... }

@Component
public class ScheduledTasks {

    private static final Logger log = LoggerFactory.getLogger(ScheduledTasks.class);

    // Fixed rate: every 5 seconds (regardless of execution time)
    @Scheduled(fixedRate = 5000)
    public void reportCurrentTime() {
        log.info("Current time: {}", LocalDateTime.now());
    }

    // Fixed delay: 5 seconds AFTER previous execution completes
    @Scheduled(fixedDelay = 5000)
    public void processQueue() {
        queueService.processNext();
    }

    // Initial delay + fixed rate
    @Scheduled(initialDelay = 10000, fixedRate = 60000)
    public void syncWithExternalService() {
        syncService.sync();
    }

    // Cron expression: every day at 2:30 AM
    @Scheduled(cron = "0 30 2 * * ?")
    public void nightlyCleanup() {
        cleanupService.cleanup();
    }

    // Using properties for flexible scheduling
    @Scheduled(cron = "${app.reports.cron:0 0 8 * * MON-FRI}")
    public void generateDailyReport() {
        reportService.generate();
    }
}
```

**Cron Expression Format:**
```
┌───────────── second (0 - 59)
│ ┌───────────── minute (0 - 59)
│ │ ┌───────────── hour (0 - 23)
│ │ │ ┌───────────── day of month (1 - 31)
│ │ │ │ ┌───────────── month (1 - 12)
│ │ │ │ │ ┌───────────── day of week (0 - 7, MON-SUN)
│ │ │ │ │ │
* * * * * *
```

> [!WARNING]
> `@Scheduled` methods run in a **single-threaded** scheduler by default (thread pool size = 1). If one task takes too long, it blocks all other scheduled tasks. Configure a `TaskScheduler` bean with multiple threads in production.

```java
@Configuration
public class SchedulingConfig {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-task-");
        scheduler.initialize();
        return scheduler;
    }
}
```

---

### 8.2 `@EnableAsync` and `@Async`

**Purpose:** `@EnableAsync` activates asynchronous method execution. `@Async` marks a method to run in a separate thread pool.

```java
@SpringBootApplication
@EnableAsync
public class Application { ... }

@Service
public class EmailService {

    // Runs in a separate thread; caller doesn't wait
    @Async
    public void sendWelcomeEmail(String to, String name) {
        // This method runs asynchronously
        emailSender.send(buildWelcomeEmail(to, name));
        log.info("Welcome email sent to {}", to);
    }

    // Async with return value - returns CompletableFuture
    @Async
    public CompletableFuture<Boolean> sendEmailAsync(String to, String subject, String body) {
        try {
            emailSender.send(to, subject, body);
            return CompletableFuture.completedFuture(true);
        } catch (Exception e) {
            log.error("Failed to send email to {}", to, e);
            return CompletableFuture.completedFuture(false);
        }
    }
}

// Use it in a service:
@Service
public class RegistrationService {

    private final EmailService emailService;
    private final UserRepository userRepository;

    public User register(RegistrationRequest req) {
        User user = userRepository.save(new User(req));
        // Fire-and-forget: doesn't block the main thread
        emailService.sendWelcomeEmail(user.getEmail(), user.getName());
        return user;
    }
}
```

**Configure Async Thread Pool:**
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-exec-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async exception in method {}: {}", method.getName(), ex.getMessage(), ex);
            // Alert, metrics, etc.
        };
    }
}
```

> [!IMPORTANT]
> Exceptions thrown in `@Async` methods with `void` return type are **silently swallowed** unless you configure an `AsyncUncaughtExceptionHandler`. Always set one in production.

---

## 9. Caching Annotations

### 9.1 `@EnableCaching`

Activates Spring's annotation-driven caching. Add to a `@Configuration` class or `@SpringBootApplication`.

```java
@SpringBootApplication
@EnableCaching
public class Application { ... }
```

---

### 9.2 `@Cacheable`

**Purpose:** Caches the result of a method. On subsequent calls with the same key, returns the cached result without invoking the method.

```java
import org.springframework.cache.annotation.Cacheable;

@Service
public class ProductService {

    @Cacheable(
        cacheNames = "products",
        key = "#id",
        condition = "#id > 0",         // Only cache if id > 0
        unless = "#result == null"      // Don't cache null results
    )
    public Product findById(Long id) {
        log.info("Fetching product {} from database", id);
        return productRepository.findById(id).orElse(null);
    }

    // Composite key
    @Cacheable(cacheNames = "productSearch", key = "#category + '-' + #page + '-' + #size")
    public Page<Product> findByCategory(String category, int page, int size) {
        return productRepository.findByCategory(category, PageRequest.of(page, size));
    }

    // SpEL-based key using method name
    @Cacheable(cacheNames = "config", key = "#root.methodName + '-' + #region")
    public AppConfig getConfig(String region) {
        return configService.loadFor(region);
    }
}
```

---

### 9.3 `@CachePut`

**Purpose:** Always executes the method and **updates** the cache with the result. Unlike `@Cacheable`, it never skips method execution.

```java
@CachePut(cacheNames = "products", key = "#result.id")
public Product updateProduct(Long id, UpdateProductRequest req) {
    Product product = productRepository.findById(id)
        .orElseThrow(() -> new ProductNotFoundException(id));
    product.update(req);
    return productRepository.save(product); // Cache updated with new value
}
```

---

### 9.4 `@CacheEvict`

**Purpose:** Removes entries from the cache. Used after data updates or deletes.

```java
// Evict single entry
@CacheEvict(cacheNames = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id);
}

// Evict all entries in cache
@CacheEvict(cacheNames = "products", allEntries = true)
public void clearProductCache() { }

// Evict before method runs (default is after)
@CacheEvict(cacheNames = "products", key = "#id", beforeInvocation = true)
public void safeDelete(Long id) {
    productRepository.deleteById(id); // Even if this throws, cache is already cleared
}
```

---

### 9.5 `@Caching`

**Purpose:** Groups multiple cache operations on a single method.

```java
@Caching(
    cacheable = @Cacheable(cacheNames = "products", key = "#id"),
    evict = {
        @CacheEvict(cacheNames = "productList", allEntries = true),
        @CacheEvict(cacheNames = "productSearch", allEntries = true)
    }
)
public Product getAndRefreshProduct(Long id) {
    return productRepository.findById(id).orElseThrow();
}
```

**Configure a Real Cache (Redis with Spring Boot):**
```java
@Configuration
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration redisCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            )
            .disableCachingNullValues();
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(redisCacheConfiguration())
            .withCacheConfiguration("products",
                redisCacheConfiguration().entryTtl(Duration.ofHours(1)))
            .build();
    }
}
```

---

## 10. Security Annotations

### 10.1 `@EnableMethodSecurity`

**Purpose:** Enables method-level security. In Spring Security 6.x, this replaces the deprecated `@EnableGlobalMethodSecurity`.

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,   // @PreAuthorize and @PostAuthorize
    securedEnabled = true,   // @Secured
    jsr250Enabled = true     // @RolesAllowed
)
public class SecurityConfig { ... }
```

---

### 10.2 `@PreAuthorize` and `@PostAuthorize`

```java
@Service
public class DocumentService {

    // Access allowed only if user has ADMIN role
    @PreAuthorize("hasRole('ADMIN')")
    public List<Document> getAllDocuments() { ... }

    // Access allowed for users with ROLE_USER or ROLE_ADMIN
    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")
    public Document getDocument(Long id) { ... }

    // SpEL expression with method parameters
    @PreAuthorize("hasRole('ADMIN') or #username == authentication.principal.username")
    public UserProfile getUserProfile(String username) { ... }

    // @PostAuthorize checks AFTER method execution
    // Useful when you need to check the return value's ownership
    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Document findById(Long id) {
        return documentRepository.findById(id).orElseThrow();
        // If the returned document doesn't belong to the current user, access is denied
    }

    // Combining conditions
    @PreAuthorize("hasRole('EDITOR') and #document.status != 'PUBLISHED'")
    public Document updateDocument(Document document) { ... }
}
```

---

### 10.3 `@Secured`

**Purpose:** Simpler role-based access control. No SpEL support.

```java
@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
public void deleteUser(Long id) { ... }

@Secured("ROLE_USER")
public UserProfile getProfile() { ... }
```

---

### 10.4 `@RolesAllowed`

**Purpose:** JSR-250 standard annotation for role-based access. Equivalent to `@Secured`.

```java
import jakarta.annotation.security.RolesAllowed;

@RolesAllowed({"ROLE_ADMIN"})
public void adminOperation() { ... }

@RolesAllowed({"ROLE_USER", "ROLE_ADMIN"})
public List<Item> getUserItems() { ... }
```

**Security Annotation Comparison:**

| Annotation | SpEL | Parameter Access | Standard | Recommended |
|---|---|---|---|---|
| `@PreAuthorize` | ✅ Yes | ✅ Yes | Spring | ✅ Yes |
| `@PostAuthorize` | ✅ Yes | ✅ Yes (return) | Spring | ✅ Yes |
| `@Secured` | ❌ No | ❌ No | Spring | ⚠️ Legacy |
| `@RolesAllowed` | ❌ No | ❌ No | JSR-250 | ⚠️ Legacy |

---

## 11. Testing Annotations

### 11.1 `@SpringBootTest`

**Purpose:** Loads the **full** Spring ApplicationContext for integration tests. Slowest but most comprehensive.

```java
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
)
@ActiveProfiles("test")
class UserControllerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @LocalServerPort
    private int port;

    @Test
    void createUser_Returns201() {
        CreateUserRequest req = new CreateUserRequest("John", "john@example.com");
        ResponseEntity<UserDTO> response = restTemplate.postForEntity(
            "http://localhost:" + port + "/api/v1/users",
            req,
            UserDTO.class
        );
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getEmail()).isEqualTo("john@example.com");
    }
}
```

**WebEnvironment Options:**

| Option | Description |
|---|---|
| `MOCK` (default) | Mocked servlet environment, no real HTTP |
| `RANDOM_PORT` | Real embedded server on random port |
| `DEFINED_PORT` | Real embedded server on configured port |
| `NONE` | No web environment |

---

### 11.2 `@WebMvcTest`

**Purpose:** Tests only the **web layer** (controllers, filters, exception handlers). Does NOT load `@Service`, `@Repository`, or full context. Use `@MockBean` to mock dependencies.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void getUser_WhenExists_Returns200() throws Exception {
        UserDTO user = new UserDTO(1L, "Alice", "alice@example.com");
        when(userService.findById(1L)).thenReturn(user);

        mockMvc.perform(get("/api/v1/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @Test
    void createUser_WithInvalidEmail_Returns400() throws Exception {
        CreateUserRequest req = new CreateUserRequest("Bob", "not-an-email");

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isBadRequest());
    }
}
```

---

### 11.3 `@DataJpaTest`

**Purpose:** Tests only the **JPA layer** (repositories, entity mappings). Auto-configures an in-memory H2 database and rolls back each test by default.

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void findByCategoryName_ReturnsMatchingProducts() {
        Category electronics = entityManager.persist(new Category("Electronics"));
        entityManager.persist(new Product("Laptop", electronics));
        entityManager.persist(new Product("Phone", electronics));
        entityManager.flush();

        List<Product> result = productRepository.findByCategoryName("Electronics");

        assertThat(result).hasSize(2);
        assertThat(result).extracting(Product::getName)
            .containsExactlyInAnyOrder("Laptop", "Phone");
    }
}
```

---

### 11.4 `@MockBean` and `@SpyBean`

```java
@SpringBootTest
class OrderServiceTest {

    @MockBean // Replaces the real bean with a Mockito mock in the ApplicationContext
    private PaymentGateway paymentGateway;

    @SpyBean // Wraps the real bean with a Mockito spy (real methods called unless stubbed)
    private InventoryService inventoryService;

    @Autowired
    private OrderService orderService;

    @Test
    void placeOrder_ChargesPaymentGateway() {
        when(paymentGateway.charge(any())).thenReturn(PaymentResult.success("txn-123"));
        doReturn(true).when(inventoryService).isAvailable(anyLong());

        Order order = orderService.place(new OrderRequest(/* ... */));

        verify(paymentGateway).charge(any());
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }
}
```

**`@MockBean` vs `@SpyBean`:**

| Annotation | Real method called | Use when |
|---|---|---|
| `@MockBean` | ❌ No (all methods are stubs) | Full mock needed |
| `@SpyBean` | ✅ Yes (unless stubbed) | Partial mock needed |

---

### 11.5 `@TestConfiguration`

**Purpose:** Provides additional or overriding beans for a specific test. Like `@Configuration` but only active during tests.

```java
@TestConfiguration
public class TestSecurityConfig {

    @Bean
    @Primary
    public UserDetailsService testUserDetailsService() {
        UserDetails user = User.builder()
            .username("testuser")
            .password("{noop}password")
            .roles("USER")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
}

// Use in test:
@WebMvcTest(SecureController.class)
@Import(TestSecurityConfig.class)
class SecureControllerTest { ... }
```

---

## 12. Lifecycle Annotations

### 12.1 `@PostConstruct`

**Purpose:** Marks a method to run **after** the bean is fully initialized (all dependencies injected, all properties set). Part of JSR-250.

```java
import jakarta.annotation.PostConstruct;

@Component
public class CacheWarmupService {

    private final ProductRepository productRepository;
    private final CacheManager cacheManager;

    public CacheWarmupService(ProductRepository productRepository, CacheManager cacheManager) {
        this.productRepository = productRepository;
        this.cacheManager = cacheManager;
    }

    @PostConstruct
    public void warmupCache() {
        log.info("Warming up product cache...");
        List<Product> topProducts = productRepository.findTop100ByOrderBySalesDesc();
        Cache cache = cacheManager.getCache("products");
        topProducts.forEach(p -> cache.put(p.getId(), p));
        log.info("Cache warm-up complete: {} products loaded", topProducts.size());
    }
}
```

**Bean Initialization Order:**
1. Constructor executed
2. Dependencies injected (via `@Autowired`, constructor, etc.)
3. `@PostConstruct` method executed
4. Bean is ready for use

> [!TIP]
> `@PostConstruct` is a cleaner alternative to `InitializingBean#afterPropertiesSet()` or the `initMethod` in `@Bean`. It's self-contained in the class and doesn't require implementing an interface.

---

### 12.2 `@PreDestroy`

**Purpose:** Marks a method to run **before** the bean is removed from the container (on application shutdown). Used for cleanup: closing connections, flushing buffers, releasing resources.

```java
import jakarta.annotation.PreDestroy;

@Component
public class ConnectionPoolManager {

    private final DataSource dataSource;

    @PreDestroy
    public void shutdown() {
        log.info("Closing connection pool...");
        if (dataSource instanceof HikariDataSource hikari) {
            hikari.close();
        }
        log.info("Connection pool closed");
    }
}
```

**Full Bean Lifecycle:**

```
Constructor → @Autowired injection → @PostConstruct
  → Bean in use →
@PreDestroy → Bean destroyed
```

---

## 13. Conditional Annotations

Conditional annotations control whether a bean is created based on the environment, configuration, or classpath.

### 13.1 `@ConditionalOnProperty`

**Purpose:** Creates a bean only if a specific property exists and has the expected value.

```java
@Configuration
public class FeatureConfig {

    @Bean
    @ConditionalOnProperty(
        name = "app.features.analytics.enabled",
        havingValue = "true",
        matchIfMissing = false // default behavior: don't create if property is absent
    )
    public AnalyticsService analyticsService() {
        return new GoogleAnalyticsService();
    }

    @Bean
    @ConditionalOnProperty(name = "app.cache.type", havingValue = "redis")
    public CacheService redisCacheService(RedisTemplate<String, Object> template) {
        return new RedisCacheService(template);
    }

    @Bean
    @ConditionalOnProperty(name = "app.cache.type", havingValue = "redis", matchIfMissing = false)
    @ConditionalOnMissingBean(CacheService.class) // Won't create if another CacheService exists
    public CacheService inMemoryCacheService() {
        return new CaffeineCache Service();
    }
}
```

---

### 13.2 `@ConditionalOnClass` and `@ConditionalOnMissingClass`

**Purpose:** Creates a bean only if a specific class is present (or absent) on the classpath.

```java
@Configuration
@ConditionalOnClass(name = "com.amazonaws.services.s3.AmazonS3")
public class S3Config {

    @Bean
    public StorageService s3StorageService(AmazonS3 amazonS3) {
        return new S3StorageService(amazonS3);
    }
}

@Configuration
@ConditionalOnMissingClass("com.amazonaws.services.s3.AmazonS3")
public class LocalStorageConfig {

    @Bean
    public StorageService localStorageService() {
        return new LocalFileStorageService("/tmp/uploads");
    }
}
```

---

### 13.3 `@ConditionalOnBean` and `@ConditionalOnMissingBean`

**Purpose:** Creates a bean only if another specific bean is (or is not) already present in the context.

```java
@Configuration
public class MetricsConfig {

    // Only create this bean if Micrometer's MeterRegistry is present
    @Bean
    @ConditionalOnBean(MeterRegistry.class)
    public MetricsInterceptor metricsInterceptor(MeterRegistry registry) {
        return new MetricsInterceptor(registry);
    }

    // Create a default implementation only if no other UserService bean exists
    @Bean
    @ConditionalOnMissingBean(UserService.class)
    public UserService defaultUserService(UserRepository repo) {
        return new DefaultUserService(repo);
    }
}
```

---

### 13.4 `@ConditionalOnExpression`

**Purpose:** Creates a bean based on a SpEL expression evaluation.

```java
@Bean
@ConditionalOnExpression(
    "'${app.env}' == 'production' and '${app.region}' == 'us-east-1'"
)
public PrimaryDatabaseConfig primaryDatabase() {
    return new PrimaryDatabaseConfig();
}
```

**Conditional Annotations Summary:**

| Annotation | Condition |
|---|---|
| `@ConditionalOnProperty` | Property has specific value |
| `@ConditionalOnClass` | Class present on classpath |
| `@ConditionalOnMissingClass` | Class absent from classpath |
| `@ConditionalOnBean` | Bean already defined in context |
| `@ConditionalOnMissingBean` | Bean NOT yet defined in context |
| `@ConditionalOnExpression` | SpEL expression is true |
| `@ConditionalOnWebApplication` | App is a web application |
| `@ConditionalOnCloudPlatform` | Running on specific cloud platform |
| `@ConditionalOnJava` | Specific Java version |

---

## 14. Validation Annotations

### 14.1 `@Valid` and `@Validated`

```java
@RestController
@RequestMapping("/api/v1/users")
@Validated // Enables validation on path variables and request params too
public class UserController {

    @PostMapping
    public ResponseEntity<UserDTO> createUser(
            @RequestBody @Valid CreateUserRequest req // @Valid triggers bean validation
    ) {
        return ResponseEntity.status(201).body(userService.create(req));
    }

    // @Validated enables validation on method parameters (not just @RequestBody)
    @GetMapping
    public List<UserDTO> getUsers(
            @RequestParam @Min(0) int page,
            @RequestParam @Max(100) int size
    ) {
        return userService.findAll(page, size);
    }
}
```

**`@Valid` vs `@Validated`:**

| Feature | `@Valid` | `@Validated` |
|---|---|---|
| Spec | JSR-303/380 (Bean Validation) | Spring-specific |
| Validation Groups | ❌ No | ✅ Yes |
| Method parameter validation | ❌ No | ✅ Yes |
| Class-level | ❌ No | ✅ Yes |

---

### 14.2 Bean Validation Constraint Annotations

```java
import jakarta.validation.constraints.*;

public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    @Size(max = 255)
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120, message = "Invalid age")
    private Integer age;

    @NotBlank
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
        message = "Password must be at least 8 chars with uppercase, lowercase, digit, and special char"
    )
    private String password;

    @NotNull
    @Future(message = "Subscription end date must be in the future")
    private LocalDate subscriptionEndDate;

    @Valid // Cascade validation to nested objects
    @NotNull
    private AddressRequest address;
}

public class AddressRequest {
    @NotBlank
    private String street;

    @NotBlank
    @Size(min = 2, max = 100)
    private String city;

    @NotBlank
    @Pattern(regexp = "^[0-9]{5}$", message = "ZIP code must be 5 digits")
    private String zipCode;
}
```

**Common Constraint Annotations:**

| Annotation | Validates | Null Handling |
|---|---|---|
| `@NotNull` | Not null | Fails on null |
| `@NotEmpty` | Not null, not empty | Fails on null |
| `@NotBlank` | Not null, not empty, not whitespace | Fails on null |
| `@Size(min, max)` | Length/size range | Passes on null |
| `@Min(value)` | Numeric minimum | Passes on null |
| `@Max(value)` | Numeric maximum | Passes on null |
| `@Email` | Valid email format | Passes on null |
| `@Pattern(regexp)` | Regex match | Passes on null |
| `@Positive` | > 0 | Passes on null |
| `@PositiveOrZero` | >= 0 | Passes on null |
| `@Past` | Date in the past | Passes on null |
| `@Future` | Date in the future | Passes on null |
| `@AssertTrue` | Must be true | Passes on null |
| `@AssertFalse` | Must be false | Passes on null |

**Custom Validator:**
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email is already in use";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true;
        return !userRepository.existsByEmail(email);
    }
}

// Usage:
public class CreateUserRequest {
    @Email
    @UniqueEmail
    private String email;
}
```

---

## 15. ⚠️ Interview Traps & Gotchas

This is the most important section for interview preparation. Every trap below is rooted in one fundamental concept: **Spring AOP uses proxy-based method interception**, not bytecode weaving.

---

### Trap 1: `@Transactional` on Private Methods

```java
@Service
public class OrderService {

    @Transactional // ⚠️ THIS DOES NOTHING
    private void processInternalPayment(Order order) {
        // No transaction wrapping occurs
        paymentRepository.save(order.getPayment());
    }
}
```

**Why?**  
Spring creates a **CGLIB proxy** (a subclass of `OrderService`). When `processInternalPayment` is called, the proxy intercepts it. However, CGLIB **cannot override private methods** — they are not part of the proxy contract. The actual method runs directly on the target object, not through the proxy, so `@Transactional` is never applied.

**Solution:**  
- Make the method `public` or `protected`
- Use self-injection (controversial) or refactor to a separate class
- Use full AspectJ weaving (compile-time) if private method transactions are truly needed

---

### Trap 2: `@Transactional` / `@Async` / `@Cacheable` on Same-Class Calls (Self-Invocation)

```java
@Service
public class UserService {

    @Cacheable("users")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    public User getOrCreateUser(Long id) {
        // ⚠️ Calls findById directly on 'this', NOT through the proxy
        // @Cacheable has NO effect here
        return findById(id);
    }
}
```

**Why?**  
When `getOrCreateUser()` calls `findById()` via `this.findById(id)`, it bypasses the proxy entirely. The proxy only intercepts **external calls** — calls that come in from outside the bean through the proxy reference.

```
External Caller
      ↓
[UserService Proxy] ← @Cacheable intercepts HERE
      ↓
[Real UserService.findById()] 

vs. self-invocation:

[Real UserService.getOrCreateUser()]
      ↓ (direct call on 'this')
[Real UserService.findById()] ← Proxy NEVER involved
```

**Solutions:**

**Option 1 — Extract to a separate bean (preferred):**
```java
@Service
public class UserService {

    private final UserLookupService userLookupService; // injected

    public User getOrCreateUser(Long id) {
        return userLookupService.findById(id); // Goes through proxy ✅
    }
}

@Service
public class UserLookupService {
    @Cacheable("users")
    public User findById(Long id) { ... }
}
```

**Option 2 — Self-injection (Spring 4.3+):**
```java
@Service
public class UserService {

    @Lazy
    @Autowired
    private UserService self; // Injects the PROXY, not 'this'

    public User getOrCreateUser(Long id) {
        return self.findById(id); // Goes through proxy ✅
    }

    @Cacheable("users")
    public User findById(Long id) { ... }
}
```

> [!WARNING]
> Self-injection is a code smell. It signals that your class has too many responsibilities. Prefer extracting into separate classes.

---

### Trap 3: `@Async` — Same-Class Calls and Return Type Misuse

```java
@Service
public class EmailService {

    @Async
    public void sendEmail(String to) { ... }

    public void sendBulkEmails(List<String> recipients) {
        // ⚠️ @Async has NO effect — same-class invocation
        recipients.forEach(this::sendEmail);
    }

    // ⚠️ Don't return non-Future types from @Async methods expecting async behavior
    @Async
    public String getAsyncResult() {
        return "result"; // Caller STILL blocks — the return is wrapped in a Future
    }
}
```

**Correct async return type:**
```java
@Async
public CompletableFuture<String> getAsyncResult() {
    return CompletableFuture.completedFuture("result");
}

// Caller:
CompletableFuture<String> future = emailService.getAsyncResult();
// Do other work...
String result = future.get(); // Block only when you need the result
```

---

### Trap 4: `@Scheduled` Without `@EnableScheduling`

```java
@Component
public class MyTask {
    @Scheduled(fixedRate = 5000)
    public void run() {
        System.out.println("This never runs");
    }
}
// If @EnableScheduling is missing from any @Configuration class,
// the @Scheduled annotation is completely ignored
```

**Fix:** Add `@EnableScheduling` to any `@Configuration` class (or `@SpringBootApplication`).

---

### Trap 5: `@ConfigurationProperties` Not Being Processed

```java
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    // getters/setters...
}

// If you don't do ONE of the following, MailProperties is NOT processed:
// 1. Add @Component to MailProperties
// 2. Add @EnableConfigurationProperties(MailProperties.class) to a @Configuration class
// 3. Add @ConfigurationPropertiesScan to the main class
```

---

### Trap 6: `@Controller` vs `@RestController` — Returning Data vs Views

```java
@Controller
public class ApiController {

    @GetMapping("/data")
    public Map<String, String> getData() {
        return Map.of("key", "value");
        // ⚠️ Spring tries to resolve a view named "{key=value}"
        // You get a TemplateInputException, NOT JSON
    }

    @GetMapping("/data-correct")
    @ResponseBody // Must add this for data to be serialized
    public Map<String, String> getDataCorrectly() {
        return Map.of("key", "value"); // ✅ Returns JSON
    }
}
```

---

### Trap 7: `@Service` vs `@Component` — Is There Any Behavioral Difference?

**Short Answer:** In Spring core, **no**. Both are processed identically by the component scanner.

**Longer Answer:**
- Some annotation processors and documentation generators treat them differently
- AOP pointcuts can target specific annotations: `@within(org.springframework.stereotype.Service)` would match `@Service` but not `@Component`
- Certain Spring Data, Spring Cloud, or monitoring tools may add special behavior for specific stereotypes
- **Exception translation** applies only to `@Repository` (not `@Service` or `@Component`)

---

### Trap 8: `@Repository` and Exception Translation — When It Doesn't Work

```java
// ⚠️ Exception translation only works if PersistenceExceptionTranslationPostProcessor
// is in the ApplicationContext. Spring Boot adds it automatically.
// In plain Spring (non-Boot) apps, you must register it explicitly:
@Bean
public PersistenceExceptionTranslationPostProcessor exceptionTranslation() {
    return new PersistenceExceptionTranslationPostProcessor();
}
```

Also: **Exception translation does NOT apply if** the repository is not accessed through a Spring proxy (i.e., if you `new` it manually instead of injecting it).

---

### Trap 9: `@Transactional` Default Rollback Rules

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment() throws IOException {
        savePayment();
        readExternalFile(); // Throws IOException (CHECKED exception)
        // ⚠️ @Transactional does NOT roll back for checked exceptions by default!
        // The transaction commits even though an exception was thrown.
    }

    @Transactional(rollbackFor = IOException.class) // ✅ Explicit rollback for checked exception
    public void processPaymentFixed() throws IOException {
        savePayment();
        readExternalFile();
    }
}
```

---

### Trap 10: `@Bean` in `@Component` vs `@Configuration` — The Lite Mode Problem

Already covered in Section 4.1, but worth repeating:

```java
@Component // NOT @Configuration — "lite" mode
public class MyConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // ⚠️ Calls serviceB() directly
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(); // New instance every time serviceB() is called!
    }
}
// Result: serviceA has a DIFFERENT ServiceB instance than what's in the context
```

---

### Trap 11: `@Transactional` on Interface Methods (Potential Pitfall)

```java
public interface UserService {
    @Transactional
    void save(User user); // ⚠️ Depends on proxy type
}

@Service
public class UserServiceImpl implements UserService {
    @Override
    public void save(User user) { ... }
}
```

With **JDK Dynamic Proxies** (when the class implements an interface), `@Transactional` on the interface method works.  
With **CGLIB proxies** (when the class doesn't implement an interface, or `proxyTargetClass=true`), interface annotations are NOT always visible to the proxy — Spring typically picks them up from the implementation anyway, but it's safest to annotate the **implementation class** methods.

**Best practice:** Always place `@Transactional` on the **concrete implementation** class, not on interfaces.

---

### Trap 12: `@Async` Exceptions Are Silently Ignored

```java
@Async
public void riskyOperation() {
    throw new RuntimeException("Something went wrong");
    // ⚠️ This exception is SILENTLY SWALLOWED unless you configure
    // an AsyncUncaughtExceptionHandler
}
```

---

## 16. 🔍 Deep Dive — How Spring AOP Proxies Work

Understanding this section explains **all** the interview traps above.

### How Spring Creates Proxies

When Spring sees `@Transactional`, `@Async`, `@Cacheable`, or any AOP-powered annotation, it:

1. **Detects the annotation** during bean creation (via `BeanPostProcessor`)
2. **Creates a proxy** around the target bean
3. **Registers the proxy** in the ApplicationContext (not the real bean)

```
@Service                         ApplicationContext
UserServiceImpl  ─────────────→  UserServiceProxy (CGLIB subclass)
  (real object)                     ↕ intercepts method calls
                                  TransactionInterceptor
                                  CacheInterceptor
                                  AsyncExecutionInterceptor
```

### Two Types of Proxies

**1. JDK Dynamic Proxy** (used when target class implements an interface):
```java
// Spring creates: Proxy.newProxyInstance(interface)
// Only proxies methods defined in the interface
// Calls on non-interface methods bypass the proxy
```

**2. CGLIB Proxy** (used when no interface, or `proxyTargetClass = true`):
```java
// Spring creates: new UserServiceImpl$$SpringCGLIB$$0()
// Subclasses the target class
// Overrides public/protected methods to add interception
// Cannot override: private, static, or final methods
```

**Spring Boot 3.x default:** CGLIB for `@Configuration` and `@Transactional` targets (even if they implement interfaces), configurable via `spring.aop.proxy-target-class=false`.

### How Method Interception Works

```java
// Conceptual CGLIB proxy (highly simplified):
public class UserServiceImpl$$SpringCGLIB$$0 extends UserServiceImpl {

    private TransactionInterceptor txInterceptor;

    @Override
    public User save(User user) {
        // Before advice
        Transaction tx = txInterceptor.begin();
        try {
            User result = super.save(user); // Calls REAL method
            tx.commit();
            return result;
        } catch (RuntimeException e) {
            tx.rollback();
            throw e;
        }
    }
}
```

### Why Self-Invocation Bypasses Proxies

```java
@Service
public class UserService {

    public void methodA() {
        this.methodB(); // 'this' is the REAL UserService, not the proxy
    }

    @Transactional
    public void methodB() { ... }
}

// External call:
userService.methodA();
// → Hits Proxy.methodA()
// → Proxy delegates to real UserService.methodA()
// → UserService.methodA() calls this.methodB()
// → 'this' is NOT the proxy — it's the real UserService
// → @Transactional is NEVER activated
```

### The AspectJ Alternative

If you need proxy limitations removed (e.g., private method transactions), use **AspectJ compile-time or load-time weaving**:

```xml
<!-- pom.xml — AspectJ compile-time weaving -->
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <configuration>
        <aspectLibraries>
            <aspectLibrary>
                <groupId>org.springframework</groupId>
                <artifactId>spring-aspects</artifactId>
            </aspectLibrary>
        </aspectLibraries>
    </configuration>
</plugin>
```

```java
// application.properties
spring.aop.proxy-target-class=false   // Use JDK proxies for interfaces
// For full AspectJ:
spring.aop.auto=false // Disable Spring AOP, use AspectJ weaver instead
```

With AspectJ, the bytecode is **modified at compile time**, so self-invocation and private methods ARE intercepted.

---

## 17. 📝 Summary & Key Takeaways

| Category | Key Annotations | Remember |
|---|---|---|
| **Stereotype** | `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController` | Only `@Repository` adds behavior (exception translation) |
| **DI** | `@Autowired`, `@Qualifier`, `@Primary`, `@Resource`, `@Inject` | `@Qualifier` beats `@Primary`; `@Resource` resolves by name first |
| **Configuration** | `@Configuration`, `@Bean`, `@Import`, `@PropertySource`, `@ConfigurationProperties` | `@Configuration` uses CGLIB; `@Component` on config = "lite mode" bug |
| **Web** | `@RequestMapping`, `@GetMapping`, `@RequestBody`, `@PathVariable` | `@RestController` = `@Controller` + `@ResponseBody` |
| **JPA** | `@Entity`, `@Transactional`, `@Query`, `@OneToMany` | Default fetch: `@ManyToOne` is EAGER (override to LAZY!) |
| **Error Handling** | `@ExceptionHandler`, `@RestControllerAdvice` | Use `@RestControllerAdvice` for global handling |
| **Scheduling** | `@Scheduled`, `@Async` | Need `@EnableScheduling` / `@EnableAsync`; single-threaded by default |
| **Caching** | `@Cacheable`, `@CachePut`, `@CacheEvict` | Need `@EnableCaching`; same-class call = no cache |
| **Security** | `@PreAuthorize`, `@PostAuthorize` | Prefer over `@Secured`; supports SpEL |
| **Testing** | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean` | Choose the right slice test for your layer |
| **Lifecycle** | `@PostConstruct`, `@PreDestroy` | Run after injection / before destruction |
| **Conditional** | `@ConditionalOnProperty`, `@ConditionalOnClass`, `@ConditionalOnMissingBean` | Foundation of Spring Boot auto-configuration |
| **Validation** | `@Valid`, `@Validated`, constraint annotations | `@Valid` for bean validation; `@Validated` for groups and method params |

### The Golden Rules

1. **AOP = Proxy = External calls only** — self-invocation bypasses `@Transactional`, `@Async`, `@Cacheable`
2. **Private methods cannot be proxied** — CGLIB cannot override them
3. **`@Transactional` only rolls back on `RuntimeException` by default** — specify `rollbackFor` for checked exceptions
4. **`@Async` swallows exceptions in `void` methods** — always configure `AsyncUncaughtExceptionHandler`
5. **`@Bean` in `@Component` = lite mode** — inter-`@Bean` calls create new instances; use `@Configuration`
6. **Always use `FetchType.LAZY`** — default EAGER on `@ManyToOne` causes N+1 problems
7. **Prefer constructor injection** — immutable, testable, no hidden dependencies

---

## 18. ❓ Practice Questions

### Conceptual Questions

1. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`? Which one provides unique Spring behavior?

2. Explain why calling `@Transactional` within the same class doesn't work. What are three ways to fix it?

3. What happens when you have two beans of the same type and inject without `@Qualifier` or `@Primary`? What exception is thrown?

4. Why does `@Bean` in a `@Configuration` class behave differently from `@Bean` in a `@Component` class?

5. Explain the difference between `FetchType.LAZY` and `FetchType.EAGER`. What is the N+1 problem?

6. What is `@ControllerAdvice` and how does it differ from `@ExceptionHandler` in a controller?

7. What does `proxyTargetClass = true` mean in the context of Spring AOP?

8. When does `@Transactional` NOT roll back by default, and how do you fix it?

### Code Analysis Questions

9. **What's wrong with this code?**
```java
@Service
public class NotificationService {
    @Async
    private void sendAlert(String message) {
        alertClient.send(message);
    }
    
    public void notifyAll(List<String> users) {
        users.forEach(u -> sendAlert("Hello " + u));
    }
}
```

10. **Will the cache work here? Why or why not?**
```java
@Service
public class ReportService {
    
    public Report getOrGenerateReport(String type) {
        return generateReport(type);
    }
    
    @Cacheable("reports")
    public Report generateReport(String type) {
        return reportGenerator.generate(type);
    }
}
```

11. **What exception does this throw at startup or runtime?**
```java
@Service
public class UserService {
    @Autowired
    private OrderService orderService;
}

@Service  
public class OrderService {
    @Autowired
    private UserService userService;
}
```

12. **What's wrong and how do you fix it?**
```java
@Configuration
public class Config {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());
    }
    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
    // Does serviceA() have the same ServiceB as the one in the context?
}
```

### Design Questions

13. You need a method that sends emails asynchronously AND updates the database transactionally. How do you combine `@Async` and `@Transactional` correctly?

14. Design a caching strategy using Spring's caching annotations for a product catalog that must:
    - Cache individual product lookups for 1 hour
    - Evict all cached products when a new product is added
    - Update cache when a product is modified

15. How would you implement conditional bean creation such that a `MockPaymentService` is used in tests and `StripePaymentService` is used in production?

---

*Next Module → [03 — Spring Boot Auto-Configuration & the `@SpringBootApplication` Annotation](./03-auto-configuration.md)*
