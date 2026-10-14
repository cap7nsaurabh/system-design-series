# Spring Boot Fundamentals — Deep Dive

> **Module 01 | Spring Boot Series**
> Spring Boot 3.x / Spring Framework 6.x | Java 17+

---

## Table of Contents

1. [Spring Boot vs Spring Framework](#1-spring-boot-vs-spring-framework)
2. [Inversion of Control (IoC)](#2-inversion-of-control-ioc)
3. [Dependency Injection (DI)](#3-dependency-injection-di)
4. [Application Context](#4-application-context)
5. [Bean Lifecycle](#5-bean-lifecycle)
6. [Bean Scopes](#6-bean-scopes)
7. [@SpringBootApplication](#7-springbootapplication)
8. [Auto-Configuration](#8-auto-configuration)
9. [SpringApplication.run()](#9-springapplicationrun)
10. [Profiles](#10-profiles)
11. [Configuration Properties](#11-configuration-properties)
12. [Spring Boot Starters](#12-spring-boot-starters)
13. [⚠️ Interview Traps & Gotchas](#-interview-traps--gotchas)
14. [🔍 Deep Dive — Internal Mechanics](#-deep-dive--internal-mechanics)
15. [📝 Summary & Key Takeaways](#-summary--key-takeaways)
16. [❓ Practice Questions](#-practice-questions)

---

## 1. Spring Boot vs Spring Framework

### 1.1 The Problem Spring Boot Solves

Before Spring Boot (pre-2014), setting up a Spring application required enormous boilerplate:

- Manually declaring hundreds of XML beans or Java `@Configuration` classes
- Managing conflicting dependency versions in `pom.xml`
- Setting up an embedded server (Tomcat/Jetty) separately
- Writing `web.xml` for servlet context
- Configuring `DispatcherServlet`, `ViewResolver`, `DataSource`, `TransactionManager` by hand

A typical Spring MVC + Hibernate setup required **50+ lines of configuration** before writing a single line of business logic.

### 1.2 What Spring Boot Actually Is

Spring Boot is **not** a replacement for the Spring Framework — it is an **opinionated layer on top of it**. It provides:

| Feature | Spring Framework | Spring Boot |
|---|---|---|
| Dependency Management | Manual (you choose versions) | Curated BOM (`spring-boot-dependencies`) |
| Configuration | Explicit XML or Java config | Auto-configuration via `@EnableAutoConfiguration` |
| Server | Deploy WAR to external server | Embedded Tomcat/Jetty/Undertow (fat JAR) |
| Startup | `new ClassPathXmlApplicationContext(...)` | `SpringApplication.run(...)` |
| Monitoring | Not included | Actuator included via starter |
| Opinionated Defaults | None | Sensible defaults for every layer |

### 1.3 Spring Boot's Core Philosophy: Convention Over Configuration

Spring Boot follows **three pillars**:

1. **Starters** — pre-packaged dependency groups
2. **Auto-Configuration** — beans auto-wired based on classpath content
3. **Actuator** — production-ready monitoring endpoints

```java
// A complete REST API in Spring Boot (no XML, no web.xml, no config)
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

The equivalent Spring MVC setup without Spring Boot would require: `web.xml`, `spring-mvc-config.xml`, `applicationContext.xml`, `persistence.xml`, and dozens of bean declarations.

### 1.4 Auto-Configuration Magic — The 30-Second Overview

Spring Boot scans the classpath at startup. If it finds `spring-webmvc.jar` + `Tomcat`, it automatically configures a `DispatcherServlet`. If it finds `h2.jar` + `spring-data-jpa.jar`, it automatically configures a `DataSource` and `EntityManagerFactory`.

This is done through:
- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+/3.x)
- Conditional annotations like `@ConditionalOnClass`, `@ConditionalOnMissingBean`

We'll go deep on this in [Section 8](#8-auto-configuration).

---

## 2. Inversion of Control (IoC)

### 2.1 What Is IoC?

**Inversion of Control** is a design principle where the **control flow is inverted** — instead of your code creating and managing object dependencies, a **framework (the container)** does it for you.

#### Traditional Approach (No IoC):

```java
public class OrderService {
    // You are responsible for creating the dependency
    private final PaymentService paymentService = new PaymentService();
    private final InventoryService inventoryService = new InventoryService(new DatabaseConnection());

    public void placeOrder(Order order) {
        inventoryService.reserve(order);
        paymentService.charge(order);
    }
}
```

**Problems with this approach:**
- `OrderService` is tightly coupled to concrete implementations
- Hard to test (cannot mock `PaymentService`)
- Changing `PaymentService` requires changing `OrderService`
- Object graph creation is scattered across the codebase
- Lifecycle management is manual and error-prone

#### IoC Approach:

```java
@Service
public class OrderService {
    // Container injects the dependency — you don't create it
    private final PaymentService paymentService;
    private final InventoryService inventoryService;

    public OrderService(PaymentService paymentService, 
                        InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }

    public void placeOrder(Order order) {
        inventoryService.reserve(order);
        paymentService.charge(order);
    }
}
```

Now `OrderService` declares **what it needs** (the "what"), and the **container decides how to provide it** (the "how").

### 2.2 The IoC Container

The Spring IoC container is the heart of the framework. It is responsible for:

1. **Instantiating** beans (managed objects)
2. **Configuring** beans (setting properties)
3. **Assembling** beans (wiring dependencies)
4. **Managing** bean lifecycle (init and destroy callbacks)

The container reads **configuration metadata** which can come from:

```
┌─────────────────────────────────────────────┐
│           Configuration Metadata             │
├───────────────┬───────────────┬──────────────┤
│  XML Config   │  Java Config  │  Annotations │
│  (legacy)     │  @Configuration│  @Component  │
│  beans.xml    │  @Bean methods│  @Service    │
│               │               │  @Repository │
└───────────────┴───────────────┴──────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │   Spring IoC Container │
            │  (ApplicationContext)  │
            └───────────┬───────────┘
                        │ produces
                        ▼
            ┌───────────────────────┐
            │   Fully Configured    │
            │    Application        │
            │    (Ready Beans)      │
            └───────────────────────┘
```

### 2.3 Why IoC Matters

1. **Testability** — You can inject mock implementations in tests
2. **Loose Coupling** — Classes depend on interfaces, not implementations  
3. **Single Responsibility** — Classes don't manage object lifecycle
4. **Configurability** — Swap implementations without changing code

```java
// Interface
public interface NotificationService {
    void send(String message, String recipient);
}

// Production implementation
@Service
@Profile("!test")
public class EmailNotificationService implements NotificationService {
    @Override
    public void send(String message, String recipient) {
        // Send real email via SMTP
    }
}

// Test implementation — same interface, different behavior
@Service
@Profile("test")
public class MockNotificationService implements NotificationService {
    private final List<String> sentMessages = new ArrayList<>();

    @Override
    public void send(String message, String recipient) {
        sentMessages.add(message); // Just record it
    }

    public List<String> getSentMessages() {
        return sentMessages;
    }
}

// OrderService doesn't know or care which implementation it gets
@Service
public class OrderService {
    private final NotificationService notificationService;

    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

---

## 3. Dependency Injection (DI)

Dependency Injection is the **primary mechanism** Spring uses to implement IoC. There are three types of DI in Spring:

### 3.1 Constructor Injection (Recommended)

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;

    // @Autowired is optional when there's a single constructor (Spring 4.3+)
    public UserService(UserRepository userRepository,
                       PasswordEncoder passwordEncoder,
                       EmailService emailService) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.emailService = emailService;
    }

    public User registerUser(RegisterRequest request) {
        String hashedPassword = passwordEncoder.encode(request.password());
        User user = new User(request.username(), hashedPassword, request.email());
        User saved = userRepository.save(user);
        emailService.sendWelcomeEmail(saved);
        return saved;
    }
}
```

**Why constructor injection is the best choice:**

| Criterion | Constructor | Field | Setter |
|---|---|---|---|
| Immutability | ✅ Fields can be `final` | ❌ Cannot use `final` | ❌ Cannot use `final` |
| Testability | ✅ No reflection needed | ❌ Requires Spring context or reflection | ⚠️ Possible but awkward |
| Fail-fast | ✅ Fails at startup if deps missing | ❌ Fails at runtime | ❌ Fails at runtime |
| Circular dep detection | ✅ Detected at startup | ❌ May silently use partial state | ⚠️ Detected at startup |
| Required dependencies | ✅ Explicit, mandatory | ❌ Optional by default | ⚠️ Can be optional |
| Null safety | ✅ Guaranteed non-null | ❌ Can be null if not injected | ⚠️ Depends |
| Code clarity | ✅ Explicit contract | ❌ Hidden dependencies | ⚠️ Medium |

### 3.2 Field Injection (Avoid in Production Code)

```java
@Service
public class ProductService {

    @Autowired  // ← Spring uses reflection to inject this
    private ProductRepository productRepository;

    @Autowired
    private CacheService cacheService;

    public Product findById(Long id) {
        return cacheService.get(id)
                .orElseGet(() -> productRepository.findById(id).orElseThrow());
    }
}
```

**Hidden dangers of field injection:**

```java
// This test FAILS without a Spring context — you can't easily inject mocks
class ProductServiceTest {
    @Test
    void testFindById() {
        ProductService service = new ProductService(); 
        // productRepository is null! NullPointerException at runtime
        service.findById(1L);
    }
}

// With constructor injection, this works perfectly:
class ProductServiceTest {
    @Test
    void testFindById() {
        ProductRepository mockRepo = mock(ProductRepository.class);
        CacheService mockCache = mock(CacheService.class);
        ProductService service = new ProductService(mockRepo, mockCache);
        // Test away without any Spring context
    }
}
```

### 3.3 Setter Injection (For Optional Dependencies)

```java
@Service
public class ReportService {

    private ReportFormatter formatter;
    private DataExporter exporter; // optional

    // Required dependency via constructor
    public ReportService(ReportFormatter formatter) {
        this.formatter = formatter;
    }

    // Optional dependency via setter
    @Autowired(required = false)
    public void setExporter(DataExporter exporter) {
        this.exporter = exporter;
    }

    public Report generate(ReportRequest request) {
        Report report = formatter.format(request);
        if (exporter != null) {
            exporter.export(report);
        }
        return report;
    }
}
```

> [!TIP]
> Use **constructor injection for required dependencies** and **setter injection for optional dependencies** only. Never use field injection in production code.

### 3.4 Qualifying Beans — Resolving Ambiguity

When multiple beans of the same type exist:

```java
// Two implementations of the same interface
@Service("stripePayment")
public class StripePaymentService implements PaymentService { ... }

@Service("paypalPayment")
public class PayPalPaymentService implements PaymentService { ... }

// Disambiguation Option 1: @Qualifier
@Service
public class CheckoutService {
    private final PaymentService paymentService;

    public CheckoutService(@Qualifier("stripePayment") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// Disambiguation Option 2: @Primary — marks one as the default
@Service
@Primary
public class StripePaymentService implements PaymentService { ... }

// Disambiguation Option 3: Match by field/parameter name
@Service
public class CheckoutService {
    // Spring matches by parameter name "stripePayment" to bean name
    public CheckoutService(PaymentService stripePayment) {
        this.paymentService = stripePayment;
    }
}

// Disambiguation Option 4: Inject all implementations
@Service
public class PaymentOrchestrator {
    private final List<PaymentService> paymentServices;

    public PaymentOrchestrator(List<PaymentService> paymentServices) {
        this.paymentServices = paymentServices; // Gets all PaymentService beans
    }

    public boolean processWithFallback(PaymentRequest request) {
        for (PaymentService service : paymentServices) {
            try {
                return service.process(request);
            } catch (Exception e) {
                // try next provider
            }
        }
        return false;
    }
}
```

### 3.5 Lazy Injection

```java
@Service
public class HeavyService {

    // Injected as a proxy — actual bean only created on first use
    @Lazy
    private final ExpensiveService expensiveService;

    public HeavyService(@Lazy ExpensiveService expensiveService) {
        this.expensiveService = expensiveService;
    }

    public void doWorkIfNeeded(boolean needed) {
        if (needed) {
            expensiveService.doWork(); // Bean created here, not at startup
        }
    }
}
```

> [!NOTE]
> `@Lazy` on constructor parameters tells Spring to inject a proxy instead of eagerly creating the dependency. Useful for heavy resources or breaking circular dependencies.

---

## 4. Application Context

### 4.1 BeanFactory vs ApplicationContext

The Spring container hierarchy:

```
BeanFactory (core interface)
    └── ApplicationContext (extends BeanFactory + more features)
            ├── ConfigurableApplicationContext
            │       ├── AbstractApplicationContext
            │       │       ├── AnnotationConfigApplicationContext
            │       │       ├── ClassPathXmlApplicationContext
            │       │       └── GenericWebApplicationContext
            │       └── ConfigurableWebApplicationContext
            │               └── AnnotationConfigServletWebApplicationContext
            └── WebApplicationContext
```

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| Bean instantiation/wiring | ✅ | ✅ |
| Bean lifecycle callbacks | ✅ | ✅ |
| Internationalization (i18n) | ❌ | ✅ (`MessageSource`) |
| Event propagation | ❌ | ✅ (`ApplicationEventPublisher`) |
| AOP integration | Limited | ✅ Full |
| `BeanPostProcessor` auto-detection | ❌ Manual | ✅ Automatic |
| `BeanFactoryPostProcessor` auto-detection | ❌ Manual | ✅ Automatic |
| Resource loading | ❌ | ✅ (`ResourceLoader`) |
| Eager singleton initialization | ❌ (lazy by default) | ✅ (eager by default) |

> [!IMPORTANT]
> Always use `ApplicationContext` in production Spring apps. `BeanFactory` is primarily for resource-constrained environments (like IoT devices). `ApplicationContext` auto-detects and registers `BeanPostProcessor`s — if you use `BeanFactory`, you must register them manually, which is almost always a bug-prone mistake.

### 4.2 Types of ApplicationContext

```java
// 1. Standalone (no web) — annotation-based
AnnotationConfigApplicationContext ctx = 
    new AnnotationConfigApplicationContext(AppConfig.class);
UserService userService = ctx.getBean(UserService.class);
ctx.close();

// 2. Standalone (no web) — classpath XML (legacy)
ClassPathXmlApplicationContext xmlCtx = 
    new ClassPathXmlApplicationContext("applicationContext.xml");

// 3. Web application context (typically done by Spring Boot automatically)
// AnnotationConfigServletWebServerApplicationContext is used by Spring Boot web apps

// 4. Spring Boot context (chosen automatically based on classpath)
// - NONE: AnnotationConfigApplicationContext
// - SERVLET: AnnotationConfigServletWebServerApplicationContext  
// - REACTIVE: AnnotationConfigReactiveWebServerApplicationContext
```

### 4.3 Using ApplicationContext Programmatically

```java
@Component
public class BeanInspector implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    public void inspectBeans() {
        String[] beanNames = applicationContext.getBeanDefinitionNames();
        System.out.println("Total beans registered: " + beanNames.length);

        for (String name : beanNames) {
            System.out.printf("  Bean: %-50s | Type: %s%n",
                name,
                applicationContext.getType(name));
        }
    }

    // Get bean by type — preferred over getBean(String name)
    public UserService getUserService() {
        return applicationContext.getBean(UserService.class);
    }

    // Check if a bean exists before retrieving
    public Optional<AuditService> getAuditService() {
        if (applicationContext.containsBean("auditService")) {
            return Optional.of(applicationContext.getBean(AuditService.class));
        }
        return Optional.empty();
    }
}
```

### 4.4 ApplicationContext as Event Publisher

```java
// Define a custom event
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;

    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }

    public Order getOrder() { return order; }
}

// Publish the event
@Service
public class OrderService implements ApplicationEventPublisherAware {

    private ApplicationEventPublisher eventPublisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.eventPublisher = publisher;
    }

    // Or inject directly via constructor (simpler):
    // public OrderService(ApplicationEventPublisher eventPublisher) { ... }

    public Order placeOrder(OrderRequest request) {
        Order order = processOrder(request);
        eventPublisher.publishEvent(new OrderPlacedEvent(this, order));
        return order;
    }
}

// Listen to the event
@Component
public class NotificationListener {

    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Order placed: " + event.getOrder().getId());
        // Send email, SMS, etc.
    }

    // Async event handling
    @EventListener
    @Async
    public void handleOrderPlacedAsync(OrderPlacedEvent event) {
        // Runs in a separate thread pool
    }
}
```

---

## 5. Bean Lifecycle

This is one of the most important topics for Spring interviews. Understanding the **complete lifecycle** of a bean from creation to destruction is critical.

### 5.1 Complete Bean Lifecycle Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SPRING BEAN LIFECYCLE                             │
│                                                                          │
│  Container                                                               │
│  Startup                                                                 │
│    │                                                                     │
│    ▼                                                                     │
│  1. INSTANTIATION                                                        │
│     Spring creates the bean instance via constructor                     │
│     (or factory method if @Bean)                                         │
│    │                                                                     │
│    ▼                                                                     │
│  2. POPULATE PROPERTIES                                                  │
│     Spring injects dependencies (constructor args already done,         │
│     field/setter injection happens here)                                 │
│    │                                                                     │
│    ▼                                                                     │
│  3. AWARE INTERFACES (in order):                                         │
│     a. BeanNameAware.setBeanName(String name)                           │
│     b. BeanClassLoaderAware.setBeanClassLoader(ClassLoader cl)          │
│     c. BeanFactoryAware.setBeanFactory(BeanFactory bf)                  │
│     d. EnvironmentAware.setEnvironment(Environment env)                 │
│     e. EmbeddedValueResolverAware                                        │
│     f. ResourceLoaderAware                                               │
│     g. ApplicationEventPublisherAware                                    │
│     h. MessageSourceAware                                                │
│     i. ApplicationContextAware.setApplicationContext(ctx)               │
│    │                                                                     │
│    ▼                                                                     │
│  4. BeanPostProcessor.postProcessBeforeInitialization()                  │
│     (All registered BPPs run their "before" phase)                       │
│     ← @PostConstruct is processed HERE by CommonAnnotationBPP           │
│    │                                                                     │
│    ▼                                                                     │
│  5. INITIALIZATION                                                       │
│     a. @PostConstruct method runs (via CommonAnnotationBeanPostProcessor)│
│     b. InitializingBean.afterPropertiesSet()                            │
│     c. Custom init-method (if specified)                                 │
│    │                                                                     │
│    ▼                                                                     │
│  6. BeanPostProcessor.postProcessAfterInitialization()                   │
│     (All registered BPPs run their "after" phase)                        │
│     ← AOP proxies are typically created HERE by AbstractAutoProxy...BPP │
│    │                                                                     │
│    ▼                                                                     │
│  7. BEAN IS READY FOR USE                                                │
│     Bean is placed in the singleton pool / returned to caller           │
│    │                                                                     │
│    │  (Application runs...)                                              │
│    │                                                                     │
│    ▼                                                                     │
│  Container                                                               │
│  Shutdown                                                                │
│    │                                                                     │
│    ▼                                                                     │
│  8. DESTRUCTION                                                          │
│     a. @PreDestroy method runs                                           │
│     b. DisposableBean.destroy()                                          │
│     c. Custom destroy-method (if specified)                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Complete Lifecycle Demonstration

```java
package com.example.lifecycle;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

/**
 * Demonstrates the complete Spring bean lifecycle.
 * Run this and observe the log order.
 */
@Component("lifecycleDemo")
public class LifecycleDemoBean
        implements BeanNameAware,
                   BeanFactoryAware,
                   ApplicationContextAware,
                   InitializingBean,
                   DisposableBean {

    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;
    private final DependencyService dependencyService;

    // Step 1: INSTANTIATION — constructor called first
    public LifecycleDemoBean(DependencyService dependencyService) {
        System.out.println("=== [1] INSTANTIATION: Constructor called ===");
        System.out.println("    dependencyService injected: " + (dependencyService != null));
        this.dependencyService = dependencyService;
    }

    // Step 3a: BeanNameAware
    @Override
    public void setBeanName(String name) {
        System.out.println("=== [3a] BeanNameAware.setBeanName: " + name + " ===");
        this.beanName = name;
    }

    // Step 3c: BeanFactoryAware
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("=== [3c] BeanFactoryAware.setBeanFactory called ===");
        this.beanFactory = beanFactory;
    }

    // Step 3i: ApplicationContextAware
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("=== [3i] ApplicationContextAware.setApplicationContext called ===");
        this.applicationContext = applicationContext;
    }

    // Step 5a: @PostConstruct (processed by CommonAnnotationBeanPostProcessor)
    @PostConstruct
    public void postConstruct() {
        System.out.println("=== [5a] @PostConstruct: postConstruct() called ===");
        System.out.println("    Bean name: " + beanName);
        System.out.println("    Doing initialization work (e.g., cache warmup, connection validation)");
        dependencyService.validate();
    }

    // Step 5b: InitializingBean.afterPropertiesSet()
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("=== [5b] InitializingBean.afterPropertiesSet() called ===");
        // Note: runs AFTER @PostConstruct
    }

    // The bean is now READY for use
    public void doWork() {
        System.out.println("Bean is doing work...");
    }

    // Step 8a: @PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("=== [8a] @PreDestroy: preDestroy() called ===");
        System.out.println("    Releasing resources (closing connections, flushing buffers)");
    }

    // Step 8b: DisposableBean.destroy()
    @Override
    public void destroy() throws Exception {
        System.out.println("=== [8b] DisposableBean.destroy() called ===");
    }
}
```

```java
// Custom BeanPostProcessor — runs around every bean's init
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
        if (beanName.equals("lifecycleDemo")) {
            System.out.println("=== [4] BPP.postProcessBeforeInitialization for: " + beanName + " ===");
        }
        return bean; // IMPORTANT: always return the (possibly modified) bean
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) 
            throws BeansException {
        if (beanName.equals("lifecycleDemo")) {
            System.out.println("=== [6] BPP.postProcessAfterInitialization for: " + beanName + " ===");
            System.out.println("    (This is where AOP proxies are created)");
        }
        return bean; // Could return a proxy instead of the original bean!
    }
}
```

```java
// Supporting bean
@Service
public class DependencyService {
    
    public DependencyService() {
        System.out.println("=== [1] INSTANTIATION: DependencyService constructor ===");
    }

    public void validate() {
        System.out.println("    DependencyService.validate() called");
    }
}
```

**Expected Console Output:**

```
=== [1] INSTANTIATION: DependencyService constructor ===
=== [1] INSTANTIATION: Constructor called ===
    dependencyService injected: true
=== [3a] BeanNameAware.setBeanName: lifecycleDemo ===
=== [3c] BeanFactoryAware.setBeanFactory called ===
=== [3i] ApplicationContextAware.setApplicationContext called ===
=== [4] BPP.postProcessBeforeInitialization for: lifecycleDemo ===
=== [5a] @PostConstruct: postConstruct() called ===
    Bean name: lifecycleDemo
    Doing initialization work...
    DependencyService.validate() called
=== [5b] InitializingBean.afterPropertiesSet() called ===
=== [6] BPP.postProcessAfterInitialization for: lifecycleDemo ===
    (This is where AOP proxies are created)
--- Application Running ---
--- Context Closing ---
=== [8a] @PreDestroy: preDestroy() called ===
    Releasing resources...
=== [8b] DisposableBean.destroy() called ===
```

### 5.3 `@PostConstruct` vs Constructor for Initialization

> [!WARNING]
> **Interview Trap**: Many candidates think you can do initialization work in the constructor. You often can't, because at constructor time, dependencies may not be fully initialized, and Spring infrastructure (like AOP proxies) isn't set up yet.

```java
@Service
public class CacheWarmupService {

    private final ProductRepository productRepository;
    private final Map<Long, Product> cache = new HashMap<>();

    public CacheWarmupService(ProductRepository productRepository) {
        this.productRepository = productRepository;
        // ❌ DON'T do this in constructor:
        // warmupCache(); // productRepository might not be fully initialized
        //                // Transactional proxies don't exist yet
        //                // @Transactional on this method would be IGNORED
    }

    @PostConstruct
    public void warmupCache() {
        // ✅ CORRECT: Do initialization here
        // - All dependencies are fully injected
        // - AOP proxies are in place (but wait — BPP "after" phase runs AFTER @PostConstruct)
        // - @Transactional works IF this bean itself is proxied BEFORE this call...
        // Actually: @PostConstruct runs BEFORE postProcessAfterInitialization!
        // So @Transactional might NOT work in @PostConstruct. Use ApplicationRunner instead.
        
        List<Product> products = productRepository.findAll();
        products.forEach(p -> cache.put(p.getId(), p));
        System.out.println("Cache warmed up with " + cache.size() + " products");
    }
}
```

> [!IMPORTANT]
> `@PostConstruct` runs **before** `postProcessAfterInitialization`. This means if you need AOP-proxied behavior (like `@Transactional`) inside `@PostConstruct`, it **won't work as expected**. Use `ApplicationRunner` or `CommandLineRunner` instead, which run after the entire context is fully initialized.

```java
// Correct way to run transactional startup code
@Component
public class StartupDataLoader implements ApplicationRunner {

    private final ProductRepository productRepository;

    public StartupDataLoader(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Override
    @Transactional // Works correctly here
    public void run(ApplicationArguments args) throws Exception {
        // This runs AFTER the full ApplicationContext is ready
        // @Transactional proxies are active
        productRepository.findAll().forEach(this::process);
    }

    private void process(Product product) { /* ... */ }
}
```

### 5.4 Custom Init and Destroy Methods

```java
// Via @Bean annotation
@Configuration
public class DataSourceConfig {

    @Bean(initMethod = "connect", destroyMethod = "disconnect")
    public CustomDataSource customDataSource() {
        return new CustomDataSource();
    }
}

public class CustomDataSource {
    public void connect() {
        System.out.println("Connecting to data source...");
    }
    
    public void disconnect() {
        System.out.println("Disconnecting from data source...");
    }
}
```

---

## 6. Bean Scopes

### 6.1 Overview of All Scopes

| Scope | Description | Availability |
|---|---|---|
| `singleton` | One instance per Spring container (default) | All contexts |
| `prototype` | New instance every time bean is requested | All contexts |
| `request` | One instance per HTTP request | Web contexts only |
| `session` | One instance per HTTP session | Web contexts only |
| `application` | One instance per `ServletContext` | Web contexts only |
| `websocket` | One instance per WebSocket session | Web contexts only |

### 6.2 Singleton Scope (Default)

```java
// These are equivalent — singleton is the default
@Service
public class UserService { ... }

@Service
@Scope("singleton")
public class UserService { ... }

@Service
@Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
public class UserService { ... }
```

**Key characteristics:**
- Created once when the container starts (for eager singletons)
- Shared across all requests, threads, and callers
- Must be **thread-safe** for shared mutable state

```java
// ❌ DANGER: Singleton with mutable state (NOT thread-safe)
@Service
public class UnsafeCounterService {
    private int count = 0; // Shared across ALL threads!

    public void increment() {
        count++; // Race condition!
    }
}

// ✅ Thread-safe singleton with atomic operations
@Service
public class SafeCounterService {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // Thread-safe
    }

    public int getCount() {
        return count.get();
    }
}
```

### 6.3 Prototype Scope

```java
@Component
@Scope("prototype")
public class ReportBuilder {

    private final List<ReportSection> sections = new ArrayList<>();
    private String title;

    public ReportBuilder withTitle(String title) {
        this.title = title;
        return this;
    }

    public ReportBuilder addSection(ReportSection section) {
        this.sections.add(section);
        return this;
    }

    public Report build() {
        return new Report(title, sections);
    }
}

// Requesting a new prototype instance — two options:
@Service
public class ReportService implements ApplicationContextAware {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public Report generateReport(String title) {
        // Each call to getBean creates a NEW ReportBuilder instance
        ReportBuilder builder = ctx.getBean(ReportBuilder.class);
        return builder.withTitle(title)
                      .addSection(new SummarySection())
                      .build();
    }
}
```

> [!WARNING]
> **Spring does NOT manage the destroy lifecycle of prototype beans!** Once Spring hands you a prototype, it forgets about it. `@PreDestroy` and `DisposableBean.destroy()` are **never called** for prototype-scoped beans. You are responsible for cleanup.

### 6.4 ⚠️ Critical Trap: Prototype Bean Injected Into Singleton

This is the **most common Spring scope interview trap**:

```java
@Component
@Scope("prototype")
public class ShoppingCart {
    private final List<Item> items = new ArrayList<>();
    private final String cartId = UUID.randomUUID().toString();

    public void addItem(Item item) { items.add(item); }
    public String getCartId() { return cartId; }
}

// ❌ BROKEN: Singleton injects prototype — only ONE ShoppingCart is ever created!
@Service
public class CheckoutService {

    @Autowired
    private ShoppingCart shoppingCart; // Injected ONCE at startup
    // All users share the SAME cart! This is a catastrophic bug!

    public void addToCart(Item item) {
        shoppingCart.addItem(item); // Cross-contamination between users!
    }
}
```

**How to fix it — 3 approaches:**

```java
// Fix 1: ApplicationContext lookup (simple but couples to Spring API)
@Service
public class CheckoutService implements ApplicationContextAware {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void addToCart(Item item) {
        ShoppingCart cart = ctx.getBean(ShoppingCart.class); // New instance each time
        cart.addItem(item);
    }
}

// Fix 2: ObjectProvider (Spring-idiomatic, recommended)
@Service
public class CheckoutService {

    private final ObjectProvider<ShoppingCart> cartProvider;

    public CheckoutService(ObjectProvider<ShoppingCart> cartProvider) {
        this.cartProvider = cartProvider;
    }

    public void addToCart(Item item) {
        ShoppingCart cart = cartProvider.getObject(); // New prototype instance
        cart.addItem(item);
    }
}

// Fix 3: Method injection via @Lookup (elegant, Spring-proxy magic)
@Service
public abstract class CheckoutService {

    @Lookup
    public abstract ShoppingCart getShoppingCart(); // Spring overrides this method

    public void addToCart(Item item) {
        ShoppingCart cart = getShoppingCart(); // Always a fresh prototype
        cart.addItem(item);
    }
}
```

### 6.5 Request and Session Scopes

```java
// Request-scoped bean: one instance per HTTP request
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, 
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    
    private String correlationId = UUID.randomUUID().toString();
    private final Map<String, Object> attributes = new HashMap<>();

    public String getCorrelationId() { return correlationId; }
    
    public void setAttribute(String key, Object value) {
        attributes.put(key, value);
    }
}

// Session-scoped bean: one instance per user session
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession {

    private String userId;
    private List<String> recentSearches = new ArrayList<>();

    // getters and setters
    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }
    public void addRecentSearch(String query) {
        recentSearches.add(0, query);
        if (recentSearches.size() > 10) recentSearches.remove(10);
    }
}

// Injecting into singleton — works because of ScopedProxyMode!
@Service
public class SearchService {

    // Spring injects a PROXY, not the actual bean.
    // Each method call on the proxy delegates to the current request/session's bean.
    private final UserSession userSession;
    private final RequestContext requestContext;

    public SearchService(UserSession userSession, RequestContext requestContext) {
        this.userSession = userSession;
        this.requestContext = requestContext;
    }

    public SearchResult search(String query) {
        userSession.addRecentSearch(query); // Uses current user's session
        String correlationId = requestContext.getCorrelationId(); // Uses current request
        return performSearch(query, correlationId);
    }
}
```

> [!NOTE]
> **`proxyMode = ScopedProxyMode.TARGET_CLASS`** is critical for request/session scoped beans injected into singletons. Without it, Spring would try to inject the actual bean at startup time, which doesn't exist yet (no request/session exists at startup). The proxy acts as a stand-in and delegates to the correct scoped instance at runtime.

---

## 7. @SpringBootApplication

### 7.1 What It Actually Is

`@SpringBootApplication` is a **composite annotation** — a meta-annotation that combines three annotations:

```java
// Actual source of @SpringBootApplication (simplified):
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration    // ← Alias for @Configuration
@EnableAutoConfiguration    // ← Triggers auto-configuration
@ComponentScan              // ← Scans for @Component, @Service, etc.
public @interface SpringBootApplication {
    
    // Shortcut for @EnableAutoConfiguration.exclude
    Class<?>[] exclude() default {};
    
    // Shortcut for @ComponentScan.basePackages
    String[] scanBasePackages() default {};
    
    // Shortcut for @ComponentScan.basePackageClasses  
    Class<?>[] scanBasePackageClasses() default {};
    
    boolean proxyBeanMethods() default true;
}
```

### 7.2 @SpringBootConfiguration

Marks the class as a configuration class (source of `@Bean` definitions). It is itself annotated with `@Configuration`, which triggers CGLIB subclassing to ensure `@Bean` methods return singletons.

```java
// These are equivalent:
@SpringBootApplication
public class MyApp { }

// vs
@SpringBootConfiguration  // = @Configuration
@EnableAutoConfiguration
@ComponentScan
public class MyApp { }
```

### 7.3 @EnableAutoConfiguration

The magic ingredient. This annotation tells Spring Boot to:
1. Look at `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
2. Load all listed auto-configuration classes
3. Apply them conditionally based on `@ConditionalOn*` annotations

### 7.4 @ComponentScan

Scans for Spring-managed components in the **same package as the annotated class and all sub-packages**:

```
com.example.myapp/
├── MyApp.java              ← @SpringBootApplication here
├── controller/
│   └── UserController.java ← @RestController — SCANNED ✅
├── service/
│   └── UserService.java    ← @Service — SCANNED ✅
└── repository/
    └── UserRepository.java ← @Repository — SCANNED ✅

com.example.other/
└── SomeBean.java           ← NOT scanned ❌ (different package tree)
```

```java
// Customizing component scan
@SpringBootApplication(
    scanBasePackages = {"com.example.myapp", "com.example.shared"},
    exclude = {DataSourceAutoConfiguration.class}  // Exclude specific auto-config
)
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}

// Excluding auto-configuration via properties (preferred for flexibility)
// application.yml:
// spring:
//   autoconfigure:
//     exclude: org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 7.5 Stereotype Annotations

All of these trigger component scanning and bean registration:

| Annotation | Use Case | Adds Behavior |
|---|---|---|
| `@Component` | Generic component | None |
| `@Service` | Business logic layer | None (semantic marker) |
| `@Repository` | Data access layer | Exception translation |
| `@Controller` | MVC controller | Request mapping |
| `@RestController` | REST API controller | `@Controller` + `@ResponseBody` |
| `@Configuration` | Config class with `@Bean` | CGLIB proxying |

> [!NOTE]
> `@Service` and `@Repository` do not add technical behavior beyond `@Component` except that `@Repository` enables Spring's **PersistenceExceptionTranslationPostProcessor**, which translates database-specific exceptions (like `SQLException`) into Spring's unified `DataAccessException` hierarchy.

---

## 8. Auto-Configuration

### 8.1 How Auto-Configuration Works — Step by Step

```
1. @EnableAutoConfiguration triggers AutoConfigurationImportSelector
                    │
                    ▼
2. Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   (Spring Boot 2.7+ / 3.x — previously spring.factories)
                    │
                    ▼
3. Loads ~150+ AutoConfiguration classes
   (DataSourceAutoConfiguration, WebMvcAutoConfiguration, JpaAutoConfiguration, etc.)
                    │
                    ▼
4. Each AutoConfiguration class evaluated for conditions:
   @ConditionalOnClass — applies only if certain class is on classpath
   @ConditionalOnMissingBean — applies only if no user-defined bean exists
   @ConditionalOnProperty — applies only if property is set
   @ConditionalOnWebApplication — applies only in web context
   etc.
                    │
                    ▼
5. Qualifying configurations create beans in the ApplicationContext
```

### 8.2 Anatomy of an Auto-Configuration Class

Let's look at a simplified version of what Spring Boot's `DataSourceAutoConfiguration` looks like:

```java
// Simplified example (similar to actual Spring Boot internals)
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration {
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @ConditionalOnProperty(prefix = "spring.datasource", name = "url")
    @Import(DataSourceConfiguration.Hikari.class)
    protected static class PooledDataSourceConfiguration {
    }
}
```

```java
// Writing your own auto-configuration
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass(MyCustomLibrary.class)  // Only if library is on classpath
@ConditionalOnMissingBean(MyCustomService.class) // Only if user hasn't defined one
@EnableConfigurationProperties(MyCustomProperties.class)
public class MyCustomAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyCustomService myCustomService(MyCustomProperties properties) {
        return new MyCustomServiceImpl(properties.getApiKey(), properties.getTimeout());
    }

    @Bean
    @ConditionalOnProperty(prefix = "mycustom", name = "cache.enabled", 
                           havingValue = "true", matchIfMissing = true)
    public MyCustomCache myCustomCache() {
        return new MyCustomCache();
    }
}
```

```properties
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.autoconfigure.MyCustomAutoConfiguration
```

### 8.3 Conditional Annotations Reference

| Annotation | Condition |
|---|---|
| `@ConditionalOnClass` | Class present on classpath |
| `@ConditionalOnMissingClass` | Class NOT on classpath |
| `@ConditionalOnBean` | Bean of given type exists |
| `@ConditionalOnMissingBean` | Bean of given type does NOT exist |
| `@ConditionalOnProperty` | Property has specific value |
| `@ConditionalOnResource` | Resource (file) exists |
| `@ConditionalOnWebApplication` | Running in a web context |
| `@ConditionalOnNotWebApplication` | NOT in a web context |
| `@ConditionalOnExpression` | SpEL expression evaluates to true |
| `@ConditionalOnJava` | Specific Java version |
| `@ConditionalOnSingleCandidate` | Exactly one bean of type |

### 8.4 User Configuration Overrides Auto-Configuration

The `@ConditionalOnMissingBean` pattern is what allows you to override defaults:

```java
// Spring Boot would auto-configure Hikari DataSource
// But if YOU define a DataSource bean, auto-config backs off:
@Configuration
public class CustomDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        // Your custom DataSource — Spring Boot's auto-config sees this and skips its own
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("app_user");
        config.setPassword("secret");
        config.setMaximumPoolSize(20);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        return new HikariDataSource(config);
    }
}
```

### 8.5 Debugging Auto-Configuration

```bash
# Enable auto-configuration report
java -jar myapp.jar --debug

# Or in application.properties:
# debug=true
```

This prints a conditions evaluation report:

```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
DataSourceAutoConfiguration matched:
  - @ConditionalOnClass found required classes 'javax.sql.DataSource', 
    'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType' (OnClassCondition)

Negative matches:
-----------------
ActiveMQAutoConfiguration:
  Did not match:
  - @ConditionalOnClass did not find required class 
    'jakarta.jms.ConnectionFactory' (OnClassCondition)
```

---

## 9. SpringApplication.run()

### 9.1 What Happens When You Call run()

```java
SpringApplication.run(MyApp.class, args);
```

This single line triggers an elaborate startup sequence. Here's what happens internally:

```
SpringApplication.run(MyApp.class, args)
    │
    ├── 1. Creates SpringApplication instance
    │       ├── Infers application type (SERVLET / REACTIVE / NONE)
    │       │   by checking classpath for DispatcherServlet / DispatcherHandler
    │       ├── Loads SpringApplicationRunListeners (from spring.factories)
    │       ├── Loads ApplicationContextInitializers
    │       └── Sets main application class
    │
    ├── 2. run() begins
    │       ├── StopWatch.start()
    │       ├── Bootstraps BootstrapContext
    │       └── Configures headless mode
    │
    ├── 3. SpringApplicationRunListeners.starting()
    │       └── Publishes ApplicationStartingEvent
    │
    ├── 4. Prepares Environment
    │       ├── Creates StandardServletEnvironment (or StandardEnvironment)
    │       ├── Loads properties sources in order:
    │       │   1. Command line args
    │       │   2. SPRING_APPLICATION_JSON
    │       │   3. ServletConfig/Context init params
    │       │   4. JNDI (java:comp/env)
    │       │   5. System properties
    │       │   6. System environment variables
    │       │   7. application-{profile}.properties/yml (outside jar)
    │       │   8. application-{profile}.properties/yml (inside jar)
    │       │   9. application.properties/yml (outside jar)
    │       │   10. application.properties/yml (inside jar)
    │       │   11. @PropertySource on @Configuration classes
    │       │   12. Default properties
    │       └── Publishes ApplicationEnvironmentPreparedEvent
    │
    ├── 5. Prints Banner (that Spring Boot ASCII art)
    │
    ├── 6. Creates ApplicationContext
    │       └── Based on type: AnnotationConfigServletWebServerApplicationContext
    │
    ├── 7. prepareContext()
    │       ├── Applies ApplicationContextInitializers
    │       ├── Publishes ApplicationContextInitializedEvent
    │       ├── Registers singleton beans (SpringApplicationArguments, Banner)
    │       ├── Loads BeanDefinitions via BeanDefinitionLoader
    │       └── Publishes ApplicationPreparedEvent
    │
    ├── 8. refreshContext()  ← THE BIG ONE
    │       └── Calls AbstractApplicationContext.refresh():
    │           ├── prepareRefresh()
    │           ├── obtainFreshBeanFactory()
    │           ├── prepareBeanFactory()
    │           ├── postProcessBeanFactory()
    │           ├── invokeBeanFactoryPostProcessors()  ← ConfigurationClassPostProcessor runs
    │           │   └── Processes @Configuration, @Bean, @ComponentScan, @Import
    │           ├── registerBeanPostProcessors()
    │           ├── initMessageSource()
    │           ├── initApplicationEventMulticaster()
    │           ├── onRefresh()  ← Creates embedded server (Tomcat starts here!)
    │           ├── registerListeners()
    │           ├── finishBeanFactoryInitialization()  ← All singleton beans created
    │           └── finishRefresh()  ← Publishes ContextRefreshedEvent
    │
    ├── 9. afterRefresh()
    │       └── Calls ApplicationRunner and CommandLineRunner beans
    │
    ├── 10. StopWatch.stop()
    │       └── Logs "Started MyApp in X.XXX seconds"
    │
    └── 11. Publishes ApplicationStartedEvent, ApplicationReadyEvent
```

### 9.2 Customizing SpringApplication

```java
@SpringBootApplication
public class MyApp {

    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MyApp.class);
        
        // Disable banner
        app.setBannerMode(Banner.Mode.OFF);
        
        // Add default properties
        Map<String, Object> defaultProps = new HashMap<>();
        defaultProps.put("server.port", "8080");
        defaultProps.put("spring.application.name", "my-service");
        app.setDefaultProperties(defaultProps);
        
        // Register additional listeners
        app.addListeners(new ApplicationStartingListener());
        
        // Set application type explicitly
        app.setWebApplicationType(WebApplicationType.SERVLET);
        
        // Lazy initialization — all beans created on first use (not at startup)
        app.setLazyInitialization(true); // Faster startup, slower first request
        
        app.run(args);
    }
}

// Custom startup listener
public class ApplicationStartingListener 
        implements ApplicationListener<ApplicationStartingEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartingEvent event) {
        System.out.println("Application starting...");
    }
}

// ApplicationRunner — runs after full context initialization
@Component
@Order(1) // Controls execution order when multiple runners exist
public class DatabaseMigrationRunner implements ApplicationRunner {

    private final Flyway flyway;

    public DatabaseMigrationRunner(Flyway flyway) {
        this.flyway = flyway;
    }

    @Override
    public void run(ApplicationArguments args) {
        flyway.migrate();
        System.out.println("Database migrations complete");
    }
}

// CommandLineRunner — similar but with raw String[] args
@Component
@Order(2)
public class SeedDataRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        System.out.println("Seeding initial data...");
    }
}
```

---

## 10. Profiles

### 10.1 What Are Profiles?

Profiles allow you to **segregate parts of your application configuration** and activate them based on the deployment environment.

```
Environments:
├── dev   → H2 in-memory DB, verbose logging, mock services
├── test  → TestContainers, separate schema, test-specific beans  
├── staging → Real DB (preprod), reduced logging
└── prod  → Production DB, minimal logging, full security
```

### 10.2 Profile-Specific Configuration Files

```
src/main/resources/
├── application.yml          ← Base config (always loaded)
├── application-dev.yml      ← Loaded when 'dev' profile active
├── application-test.yml     ← Loaded when 'test' profile active
├── application-staging.yml  ← Loaded when 'staging' profile active
└── application-prod.yml     ← Loaded when 'prod' profile active
```

```yaml
# application.yml (base)
spring:
  application:
    name: order-service
  
server:
  port: 8080

logging:
  level:
    root: INFO

app:
  feature:
    new-checkout: false
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
    driver-class-name: org.h2.Driver
  h2:
    console:
      enabled: true
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop

logging:
  level:
    com.example: DEBUG
    org.springframework: DEBUG

app:
  feature:
    new-checkout: true # Enable feature flag in dev
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}           # From environment variable
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate    # Never auto-alter schema in prod!
    show-sql: false

logging:
  level:
    root: WARN
    com.example: INFO
```

### 10.3 Activating Profiles

```bash
# 1. Command line argument (highest priority)
java -jar myapp.jar --spring.profiles.active=prod

# 2. Environment variable
SPRING_PROFILES_ACTIVE=prod java -jar myapp.jar

# 3. application.properties
# spring.profiles.active=dev

# 4. Programmatically
SpringApplication app = new SpringApplication(MyApp.class);
app.setAdditionalProfiles("dev");
app.run(args);

# 5. In tests
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest { ... }
```

### 10.4 @Profile on Beans

```java
// Only registers this bean when 'dev' or 'test' profile is active
@Service
@Profile({"dev", "test"})
public class MockEmailService implements EmailService {
    private final List<String> sentEmails = new ArrayList<>();

    @Override
    public void sendEmail(String to, String subject, String body) {
        String email = String.format("TO: %s | SUBJECT: %s | BODY: %s", to, subject, body);
        sentEmails.add(email);
        System.out.println("[MOCK EMAIL] " + email);
    }

    public List<String> getSentEmails() { return sentEmails; }
}

// Production implementation — active when 'dev' and 'test' are NOT active
@Service
@Profile("!dev & !test")
public class SmtpEmailService implements EmailService {
    private final JavaMailSender mailSender;

    public SmtpEmailService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    @Override
    public void sendEmail(String to, String subject, String body) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(to);
        message.setSubject(subject);
        message.setText(body);
        mailSender.send(message);
    }
}

// Profile expressions (Spring 5.1+)
@Profile("prod | staging")          // 'prod' OR 'staging'
@Profile("prod & !maintenance")     // 'prod' AND NOT 'maintenance'
@Profile("!dev")                    // NOT 'dev'
```

### 10.5 Profile Groups (Spring Boot 2.4+)

```yaml
# application.yml
spring:
  profiles:
    group:
      production:
        - prod
        - monitoring
        - security
      development:
        - dev
        - debug-logging
```

Now activating `production` automatically activates `prod`, `monitoring`, and `security` profiles.

### 10.6 Default Profile

```yaml
# application.yml
spring:
  profiles:
    default: dev  # Used if no profile is explicitly set
```

```java
@Service
@Profile("default")  // Only active when no other profile is active
public class DefaultFallbackService implements SomeService { ... }
```

---

## 11. Configuration Properties

### 11.1 `@Value` — Simple Property Injection

```java
@Service
public class PaymentService {

    @Value("${payment.gateway.url}")                          // Required
    private String gatewayUrl;

    @Value("${payment.gateway.timeout:5000}")                 // Default: 5000
    private int timeoutMs;

    @Value("${payment.enabled:true}")                         // Default: true
    private boolean paymentEnabled;

    @Value("${payment.supported-currencies:USD,EUR,GBP}")     // Default list
    private List<String> supportedCurrencies;

    @Value("${PAYMENT_API_KEY}")                              // Environment variable
    private String apiKey;

    @Value("#{systemProperties['user.home']}")                // SpEL expression
    private String userHome;

    @Value("#{T(java.lang.Math).PI}")                        // SpEL: constant
    private double pi;

    @Value("#{paymentConfig.getMaxAmount()}")                 // SpEL: bean method call
    private BigDecimal maxAmount;
}
```

### 11.2 `@ConfigurationProperties` — Type-Safe Configuration (Preferred)

```yaml
# application.yml
app:
  payment:
    gateway:
      url: https://api.stripe.com/v1
      timeout: 5000
      retry-attempts: 3
      api-key: ${STRIPE_API_KEY}
    supported-currencies:
      - USD
      - EUR
      - GBP
      - JPY
    fee-schedule:
      domestic: 0.029
      international: 0.039
    limits:
      min-amount: 0.50
      max-amount: 999999.99
```

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated  // Enables Bean Validation on properties
public class PaymentProperties {

    private Gateway gateway = new Gateway();
    private List<String> supportedCurrencies = new ArrayList<>();
    private FeeSchedule feeSchedule = new FeeSchedule();
    private Limits limits = new Limits();

    // Getters and setters required (or use @ConstructorBinding for immutable config)
    public Gateway getGateway() { return gateway; }
    public void setGateway(Gateway gateway) { this.gateway = gateway; }
    public List<String> getSupportedCurrencies() { return supportedCurrencies; }
    public void setSupportedCurrencies(List<String> currencies) { this.supportedCurrencies = currencies; }
    public FeeSchedule getFeeSchedule() { return feeSchedule; }
    public void setFeeSchedule(FeeSchedule schedule) { this.feeSchedule = schedule; }
    public Limits getLimits() { return limits; }
    public void setLimits(Limits limits) { this.limits = limits; }

    @Valid
    public static class Gateway {
        @NotBlank
        private String url;
        
        @Min(100) @Max(60000)
        private int timeout = 5000;
        
        @Min(0) @Max(10)
        private int retryAttempts = 3;
        
        @NotBlank
        private String apiKey;

        // getters and setters...
        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }
        public int getTimeout() { return timeout; }
        public void setTimeout(int timeout) { this.timeout = timeout; }
        public int getRetryAttempts() { return retryAttempts; }
        public void setRetryAttempts(int retryAttempts) { this.retryAttempts = retryAttempts; }
        public String getApiKey() { return apiKey; }
        public void setApiKey(String apiKey) { this.apiKey = apiKey; }
    }

    public static class FeeSchedule {
        private BigDecimal domestic = new BigDecimal("0.029");
        private BigDecimal international = new BigDecimal("0.039");
        // getters and setters...
        public BigDecimal getDomestic() { return domestic; }
        public void setDomestic(BigDecimal domestic) { this.domestic = domestic; }
        public BigDecimal getInternational() { return international; }
        public void setInternational(BigDecimal international) { this.international = international; }
    }

    public static class Limits {
        @DecimalMin("0.01")
        private BigDecimal minAmount = new BigDecimal("0.50");
        
        @DecimalMax("9999999.99")
        private BigDecimal maxAmount = new BigDecimal("999999.99");
        // getters and setters...
        public BigDecimal getMinAmount() { return minAmount; }
        public void setMinAmount(BigDecimal minAmount) { this.minAmount = minAmount; }
        public BigDecimal getMaxAmount() { return maxAmount; }
        public void setMaxAmount(BigDecimal maxAmount) { this.maxAmount = maxAmount; }
    }
}
```

```java
// Enable the properties class
@SpringBootApplication
@EnableConfigurationProperties(PaymentProperties.class)
public class MyApp { ... }

// OR — use @ConfigurationPropertiesScan (Spring Boot 2.2+)
@SpringBootApplication
@ConfigurationPropertiesScan("com.example")
public class MyApp { ... }

// Using the properties
@Service
public class PaymentProcessor {

    private final PaymentProperties props;

    public PaymentProcessor(PaymentProperties props) {
        this.props = props;
    }

    public PaymentResult process(PaymentRequest request) {
        BigDecimal amount = request.getAmount();
        
        if (amount.compareTo(props.getLimits().getMinAmount()) < 0) {
            throw new PaymentException("Amount below minimum: " + props.getLimits().getMinAmount());
        }
        
        String url = props.getGateway().getUrl();
        int timeout = props.getGateway().getTimeout();
        // Make HTTP call with URL and timeout...
        return callGateway(url, request, timeout);
    }
}
```

### 11.3 Immutable @ConfigurationProperties (Spring Boot 2.2+)

```java
// Using @ConstructorBinding for immutable configuration
@ConfigurationProperties(prefix = "app.email")
public class EmailProperties {

    private final String host;
    private final int port;
    private final String username;
    private final String password;
    private final boolean startTls;

    // @ConstructorBinding is implicit when there's a single parameterized constructor
    // (Spring Boot 3.x — no need for @ConstructorBinding annotation anymore)
    public EmailProperties(String host, int port, String username,
                          String password, boolean startTls) {
        this.host = host;
        this.port = port;
        this.username = username;
        this.password = password;
        this.startTls = startTls;
    }

    // Only getters — no setters! Immutable.
    public String getHost() { return host; }
    public int getPort() { return port; }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public boolean isStartTls() { return startTls; }
}
```

### 11.4 Relaxed Binding

Spring Boot maps property names flexibly:

| Property Format | Valid? |
|---|---|
| `app.payment.gateway-url` | ✅ Kebab-case (recommended) |
| `app.payment.gatewayUrl` | ✅ camelCase |
| `app.payment.gateway_url` | ✅ Underscore |
| `APP_PAYMENT_GATEWAY_URL` | ✅ All caps with underscore (env var) |
| `APP.PAYMENT.GATEWAY-URL` | ✅ (for env vars) |

```yaml
# These all bind to the same property 'app.payment.retryAttempts':
app.payment.retry-attempts: 3      # ← kebab (recommended for YAML)
app.payment.retryAttempts: 3       # ← camelCase
app.payment.retry_attempts: 3      # ← underscore
APP_PAYMENT_RETRY_ATTEMPTS: 3      # ← env var style
```

### 11.5 @Value vs @ConfigurationProperties Comparison

| Aspect | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Type safety | ❌ Strings only (no hierarchy) | ✅ Full type-safe binding |
| Nested properties | ❌ Flat | ✅ Hierarchical |
| Validation | ❌ No `@Validated` support | ✅ Full JSR-303 validation |
| IDE support | ⚠️ Limited | ✅ Full autocomplete |
| Relaxed binding | ⚠️ Limited | ✅ Full relaxed binding |
| Testing | ⚠️ Hard to override | ✅ Easy to mock/override |
| Documentation | ❌ None | ✅ `@ConfigurationPropertiesBinding` metadata |
| Best for | Single values, SpEL expressions | Groups of related properties |

---

## 12. Spring Boot Starters

### 12.1 What Are Starters?

Starters are **curated dependency descriptors** — Maven/Gradle POMs that pull in a group of related dependencies needed for a specific feature. They follow the naming convention:

- `spring-boot-starter-*` — Official starters
- `*-spring-boot-starter` — Third-party starters

### 12.2 Common Starters

| Starter | What It Includes |
|---|---|
| `spring-boot-starter` | Core (logging, yaml, spring-core) |
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson, Validation |
| `spring-boot-starter-webflux` | WebFlux, Netty, Reactor |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, JDBC |
| `spring-boot-starter-data-mongodb` | Spring Data MongoDB |
| `spring-boot-starter-data-redis` | Spring Data Redis, Lettuce |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-boot-starter-actuator` | Health, metrics, info endpoints |
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-aop` | AspectJ, Spring AOP |
| `spring-boot-starter-mail` | JavaMail, Spring email support |
| `spring-boot-starter-validation` | Hibernate Validator, Bean Validation |
| `spring-boot-starter-batch` | Spring Batch |
| `spring-boot-starter-amqp` | RabbitMQ, Spring AMQP |
| `spring-boot-starter-kafka` | Apache Kafka |

### 12.3 How Starters Work Internally

A starter is mostly an **empty JAR** with a `pom.xml` (or `build.gradle`) that declares transitive dependencies:

```xml
<!-- spring-boot-starter-web's pom.xml (simplified) -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>  <!-- Core starter -->
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-json</artifactId>  <!-- Jackson -->
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>  <!-- Embedded Tomcat -->
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>  <!-- Spring MVC -->
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
    </dependency>
</dependencies>
```

The actual magic (auto-configuration) lives in `spring-boot-autoconfigure` JAR, which is always included.

### 12.4 Excluding a Starter's Default Server

```xml
<!-- Swap Tomcat for Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### 12.5 Dependency Management BOM

```xml
<!-- In your root pom.xml — manages all Spring Boot dependency versions -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- After importing BOM, you don't need versions for managed dependencies: -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- No version needed — managed by BOM -->
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <!-- Version managed by BOM — guaranteed compatible -->
    </dependency>
</dependencies>
```

---

## ⚠️ Interview Traps & Gotchas

### Trap 1: Singleton Scope with Prototype Bean Injection

**Question**: "I have a singleton service that needs a new instance of a stateful helper bean for every method call. I used `@Autowired` to inject the helper. What's wrong?"

**Answer**: The singleton is created once. When Spring injects the prototype into the singleton, it injects **exactly one instance** at startup and reuses it forever. The prototype behavior is lost.

```java
// ❌ BROKEN
@Service
public class OrderProcessor {
    @Autowired
    private OrderContext context; // @Scope("prototype") — but only ONE is injected!

    public void process(Order order) {
        context.setOrder(order); // Race condition — shared state!
    }
}

// ✅ FIXED with ObjectProvider
@Service
public class OrderProcessor {
    private final ObjectProvider<OrderContext> contextProvider;

    public OrderProcessor(ObjectProvider<OrderContext> contextProvider) {
        this.contextProvider = contextProvider;
    }

    public void process(Order order) {
        OrderContext context = contextProvider.getObject(); // Fresh instance
        context.setOrder(order);
        context.execute();
    }
}
```

---

### Trap 2: Field Injection and Circular Dependencies

```java
// Field injection can HIDE circular dependencies
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB; // Spring resolves by setting field after construction
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA; // Same — can cause a hidden circular dependency
}
// Spring 6+ may throw StackOverflowError or UnsatisfiedDependencyException
// even for field injection, depending on initialization order

// Constructor injection EXPOSES circular dependencies immediately:
@Service
public class ServiceA {
    public ServiceA(ServiceB serviceB) { ... } // ← Detected at startup!
}
@Service
public class ServiceB {
    public ServiceB(ServiceA serviceA) { ... } // ← BeanCurrentlyInCreationException
}

// Solution 1: Redesign (preferred) — break the cycle
// Solution 2: @Lazy on one constructor parameter
@Service
public class ServiceA {
    public ServiceA(@Lazy ServiceB serviceB) { ... } // ServiceB injected as proxy
}

// Solution 3: Setter injection for one of them
@Service
public class ServiceA {
    private ServiceB serviceB;

    @Autowired
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}
```

> [!WARNING]
> Spring Boot 2.6+ made circular dependencies an error by default (where they previously worked silently). You can re-enable with `spring.main.allow-circular-references=true`, but this is a band-aid. Fix the design instead.

---

### Trap 3: @Autowired on Static Fields

```java
// ❌ DOES NOT WORK — Spring does not inject static fields
@Service
public class StaticUtils {
    @Autowired
    private static UserRepository userRepository; // null at runtime!

    public static User findUser(Long id) {
        return userRepository.findById(id).orElseThrow(); // NullPointerException!
    }
}

// ✅ Option 1: Use Spring beans — avoid static utility classes
@Service
public class UserUtils {
    private final UserRepository userRepository;
    // ...
}

// ✅ Option 2: If static method is absolutely needed — use @PostConstruct trick
@Service
public class StaticUtils {
    private static UserRepository staticRepo;
    
    private final UserRepository userRepository; // Instance field — injected normally
    
    public StaticUtils(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @PostConstruct
    private void init() {
        staticRepo = this.userRepository; // Copy to static after injection
    }
    
    public static User findUser(Long id) {
        return staticRepo.findById(id).orElseThrow(); // Works!
    }
}
// Note: Still problematic for testing. Avoid if possible.
```

**Why doesn't `@Autowired` work on static fields?**

Spring creates instances of beans and injects dependencies into **instance fields** via reflection (`field.set(beanInstance, dependency)`). Static fields belong to the **class**, not an instance. Spring has no mechanism to set static fields through its normal injection pipeline.

---

### Trap 4: @PostConstruct and @Transactional Don't Mix Well

```java
// ❌ @Transactional in @PostConstruct may not work
@Service
public class ProductLoader {

    private final ProductRepository repo;

    public ProductLoader(ProductRepository repo) { this.repo = repo; }

    @PostConstruct
    @Transactional  // WARNING: Transaction proxy may not be applied yet!
    public void loadInitialProducts() {
        // This runs in postProcessBeforeInitialization phase
        // AOP proxy (which enables @Transactional) is created in postProcessAfterInitialization
        // So @Transactional here is UNRELIABLE
        repo.save(new Product("default-product", BigDecimal.ONE));
    }
}

// ✅ CORRECT: Use ApplicationRunner
@Component
public class ProductLoader implements ApplicationRunner {

    private final ProductRepository repo;

    public ProductLoader(ProductRepository repo) { this.repo = repo; }

    @Override
    @Transactional  // Works correctly — full context is initialized
    public void run(ApplicationArguments args) {
        repo.save(new Product("default-product", BigDecimal.ONE));
    }
}
```

---

### Trap 5: @Configuration Proxy and proxyBeanMethods

```java
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(config());  // Calls config() below
    }

    @Bean
    public HikariConfig config() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/db");
        return config;
    }
}
// With @Configuration (proxyBeanMethods=true, the default):
// - AppConfig is CGLIB-proxied
// - When dataSource() calls config(), Spring intercepts and returns
//   the SINGLETON bean from the context
// - config() is NOT called twice — correct singleton behavior

@Configuration(proxyBeanMethods = false)  // "lite" mode
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(config()); // config() called as regular Java method!
    }

    @Bean
    public HikariConfig config() {
        return new HikariConfig(); // DIFFERENT instance each time! Bug!
    }
}
// With proxyBeanMethods=false:
// - No CGLIB proxy — slightly better startup performance
// - @Bean methods called directly as Java methods
// - Each call to config() creates a NEW HikariConfig
// - Fine if @Bean methods don't call each other, or dependencies are injected via params
```

---

### Trap 6: Spring Boot Profiles and Test Contamination

```java
// ❌ Forgetting @ActiveProfiles in tests — uses default profile
@SpringBootTest
class UserServiceTest {
    // Might accidentally use prod properties if 'prod' is the default!
}

// ✅ Always explicitly activate test profile
@SpringBootTest
@ActiveProfiles("test")
class UserServiceTest {
    @Autowired
    private UserService userService; // Gets test-profile beans

    @Test
    void testUserRegistration() { ... }
}
```

---

### Trap 7: Prototype Scope — Destroy NOT Called

```java
@Component
@Scope("prototype")
public class ExpensiveResource {
    private Connection connection;

    @PostConstruct
    public void init() {
        this.connection = openConnection(); // Opens resource
    }

    @PreDestroy  // ❌ This is NEVER called for prototype beans!
    public void cleanup() {
        connection.close(); // NEVER runs — memory/connection leak!
    }
}

// ✅ If you need cleanup, manage prototype lifecycle manually:
@Service
public class ResourceConsumer {
    private final ObjectProvider<ExpensiveResource> provider;

    public ResourceConsumer(ObjectProvider<ExpensiveResource> provider) {
        this.provider = provider;
    }

    public void doWork() {
        ExpensiveResource resource = provider.getObject();
        try {
            resource.doWork();
        } finally {
            resource.cleanup(); // Call manually
        }
    }
}
```

---

## 🔍 Deep Dive — Internal Mechanics

### How @Autowired Resolution Works

When Spring encounters `@Autowired`, it follows this resolution algorithm:

```
1. Find all beans that match the REQUIRED TYPE
   │
   ├── 0 matches → Throw NoSuchBeanDefinitionException (if required=true)
   │
   ├── 1 match → INJECT IT ✅
   │
   └── 2+ matches → Disambiguation:
           │
           ├── Is one marked @Primary? → Inject the @Primary one ✅
           │
           ├── Does a @Qualifier match the injection point's @Qualifier? → Inject it ✅
           │
           ├── Does the field/parameter NAME match a bean name? → Inject it ✅
           │
           └── Still ambiguous? → Throw NoUniqueBeanDefinitionException ❌
```

### How Auto-Configuration Ordering Works

```java
// @AutoConfiguration replaces @Configuration for auto-config classes (Spring Boot 2.7+)
@AutoConfiguration(
    after = DataSourceAutoConfiguration.class,  // Run after DataSource is configured
    before = WebMvcAutoConfiguration.class      // Run before MVC is configured
)
public class MyLibraryAutoConfiguration { ... }

// Older approach using @AutoConfigureAfter / @AutoConfigureBefore
@Configuration
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@AutoConfigureBefore(WebMvcAutoConfiguration.class)
public class LegacyAutoConfiguration { ... }
```

### How BeanPostProcessors Enable AOP

The most important BeanPostProcessor in Spring is `AbstractAutoProxyCreator`. When it encounters a bean that matches a pointcut (e.g., a class with `@Transactional` methods), it returns a **CGLIB proxy** from `postProcessAfterInitialization` instead of the original bean.

```
Original Bean (UserService)
        │
        ▼ postProcessAfterInitialization
AbstractAutoProxyCreator
        │
        ├── Does UserService have @Transactional methods? YES
        │
        ▼
CGLIB Proxy (UserService$$EnhancerBySpringCGLIB)
        │
        └── Wraps every @Transactional method with:
                1. Begin transaction
                2. Call original method
                3. Commit (or rollback on exception)
```

This is why you must **always inject the Spring-managed bean** and never call `new UserService()` directly — the `new` instance has no proxy, so `@Transactional` doesn't work.

### BeanDefinition vs Bean

Spring separates **metadata** (BeanDefinition) from **instances** (beans):

```java
// A BeanDefinition describes how to create a bean:
BeanDefinition bd = new RootBeanDefinition(UserService.class);
bd.setScope("singleton");
bd.setLazyInit(false);
bd.getConstructorArgumentValues().addIndexedArgumentValue(0, userRepository);

// The BeanDefinitionRegistry stores these definitions:
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
factory.registerBeanDefinition("userService", bd);

// The actual bean instance is only created on first getBean() call (or at refresh for singletons)
UserService service = factory.getBean(UserService.class);
```

### How Component Scanning Works Internally

`ConfigurationClassPostProcessor` (a `BeanFactoryPostProcessor`) is responsible for:

1. Processing `@Configuration` classes
2. Executing `@ComponentScan` — uses `ClassPathBeanDefinitionScanner`
3. Processing `@Import` — imports additional configuration
4. Processing `@Bean` methods — registers bean definitions
5. Processing `@PropertySource` — registers property sources

All of this happens in `invokeBeanFactoryPostProcessors()` during `refresh()`, **before** any singleton beans are created.

---

## 📝 Summary & Key Takeaways

| Concept | Key Point |
|---|---|
| IoC | Framework controls object creation and lifecycle, not you |
| DI | Prefer constructor injection; avoid field injection in prod |
| ApplicationContext | Use over BeanFactory; adds i18n, events, AOP support |
| Bean Lifecycle | 8-phase lifecycle; `@PostConstruct` before AOP proxies |
| Singleton Scope | Default; must be thread-safe if holding mutable state |
| Prototype Scope | New instance per request; Spring never calls `@PreDestroy` |
| `@SpringBootApplication` | = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| Auto-Configuration | `@ConditionalOn*` backs off when you define your own beans |
| Profiles | Separate config per environment; prefer `application-{profile}.yml` |
| `@ConfigurationProperties` | Prefer over `@Value` for groups of related properties |
| Starters | Curated dependency groups; auto-configuration lives in `spring-boot-autoconfigure` |
| Circular Dependencies | Fix the design; constructor injection exposes them early |
| `@PostConstruct` + `@Transactional` | Unreliable; use `ApplicationRunner` instead |
| Static `@Autowired` | Never works; Spring injects instance fields only |

### Architecture Principle Checklist

- [ ] All beans use **constructor injection** for required dependencies
- [ ] No `@Autowired` on fields in production code
- [ ] Stateful beans are **prototype**-scoped with manual lifecycle management
- [ ] `@ConfigurationProperties` used for all configuration groups
- [ ] Profile-specific files used for environment-specific config
- [ ] Secrets come from environment variables, not config files
- [ ] Circular dependencies are designed away, not patched with `@Lazy`
- [ ] `@Transactional` startup code uses `ApplicationRunner`, not `@PostConstruct`
- [ ] Custom auto-configurations use `@ConditionalOnMissingBean` for user override

---

## ❓ Practice Questions

### Conceptual Questions

1. **What is the difference between `BeanFactory` and `ApplicationContext`? In what real-world scenario would you ever prefer `BeanFactory`?**

2. **Explain the complete bean lifecycle in order. At which exact point is an AOP proxy created?**

3. **You have a singleton `ReportService` that needs a new `ReportContext` for each method call. `ReportContext` is `@Scope("prototype")`. Walk me through 3 different solutions.**

4. **Why doesn't `@Autowired` work on static fields? What's the underlying mechanism that prevents it?**

5. **A developer puts `@Transactional` on a `@PostConstruct` method. Will the transaction work? Why or why not?**

6. **What are the three annotations that `@SpringBootApplication` combines? What does each one do?**

7. **How does Spring Boot decide which auto-configuration classes to apply? Walk me through the mechanism.**

8. **What is relaxed binding in `@ConfigurationProperties`? Give 3 equivalent formats for the property `app.payment.retry-attempts`.**

9. **Spring Boot 2.6+ throws an error for circular dependencies by default. Why did Spring change this behavior?**

10. **What is `proxyBeanMethods = false` on `@Configuration`? When would you use it?**

### Scenario Questions

11. **Your application is taking 45 seconds to start. How would you use Spring Boot's lazy initialization feature? What are the trade-offs?**

12. **You need different `DataSource` beans for dev (H2), test (TestContainers), and prod (PostgreSQL). How would you architect this?**

13. **A `@RequestScope` bean is being injected into a singleton service. The application fails at startup. What's the problem and how do you fix it?**

14. **You're writing a library that should auto-configure a cache when both `spring-data-redis` and your library's annotations are on the classpath. Write the auto-configuration class skeleton.**

15. **Your team wants all configuration values to be validated at startup. How do you ensure that invalid configuration fails the application startup rather than causing runtime errors?**

### Code Review Questions

16. **What's wrong with this code?**
```java
@Service
@Scope("prototype")
public class DatabaseCleaner {
    @Autowired
    private EntityManager em;

    @PreDestroy
    public void cleanup() {
        em.clear();
    }
}
```

17. **This code has a subtle bug. Find it:**
```java
@Configuration
public class ServiceConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(repository());
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(repository());
    }

    @Bean
    public ExpensiveRepository repository() {
        return new ExpensiveRepository(); // Very expensive to create
    }
}
```
*(Hint: what if this were `@Configuration(proxyBeanMethods = false)`?)*

18. **Will the following cause a startup failure in Spring Boot 2.6+?**
```java
@Service
public class A {
    @Autowired private B b;
}
@Service
public class B {
    @Autowired private A a;
}
```

### Answers (Brief)

16. **Answer**: `@PreDestroy` is **never called** for `@Scope("prototype")` beans. The `EntityManager` cleanup code will never run, causing resource leaks.

17. **Answer**: If `@Configuration(proxyBeanMethods = false)` is used, `repository()` would be called **twice** as a plain Java method, creating **two different** `ExpensiveRepository` instances. `serviceA` and `serviceB` would each get a different instance. With default `proxyBeanMethods = true`, the CGLIB proxy intercepts the second `repository()` call and returns the singleton bean.

18. **Answer**: Spring Boot 2.6+ makes circular dependencies a startup error by default. This **will fail** with `BeanCurrentlyInCreationException`. The fix is to redesign to break the cycle, or in extreme cases add `spring.main.allow-circular-references=true` (discouraged).

---

*Module 01 Complete | Next: Module 02 — Spring MVC & REST APIs Deep Dive*
