# Spring Boot Fundamentals — Deep Dive
[🏠 Course Index](./00-course-index.md) | **Module 01** | [Next: 02 — Annotations →](./02-annotations.md)

---

> **Module 01 | Spring Boot Series**  
> **Target Version:** Spring Boot 3.x / Spring Framework 6.x | **Java Version:** Java 17/21+ (Jakarta EE 10)  
> **Focus:** In-depth architectural fundamentals, multi-approach dependency wiring, lifecycle hooks, and production trade-offs.

---

## Table of Contents

1. [Spring Boot vs Spring Framework](#1-spring-boot-vs-spring-framework)
2. [Inversion of Control (IoC)](#2-inversion-of-control-ioc)
3. [Dependency Injection (DI) Approaches](#3-dependency-injection-di-approaches)
   - [3.1 Approach 1: Constructor-Based Dependency Injection](#31-approach-1-constructor-based-dependency-injection-recommended-standard)
   - [3.2 Approach 2: Setter-Based Dependency Injection](#32-approach-2-setter-based-dependency-injection-for-optional-collaborators)
   - [3.3 Approach 3: Field-Based Dependency Injection](#33-approach-3-field-based-dependency-injection-anti-pattern)
   - [3.4 Approach 4: Factory Method Injection (@Bean Method Parameters)](#34-approach-4-factory-method-injection-bean-method-parameters)
   - [3.5 Approach 5: Method Injection via @Lookup](#35-approach-5-method-injection-via-lookup-dynamic-prototype-creation)
   - [3.6 Resolving Bean Ambiguity & Polymorphic Wiring Approaches](#36-resolving-bean-ambiguity--polymorphic-wiring-approaches)
   - [3.7 Lazy Injection](#37-lazy-injection-performance--circular-reference-mitigation)
4. [Application Context & Event Architecture](#4-application-context--event-architecture)
   - [4.1 BeanFactory vs ApplicationContext](#41-beanfactory-vs-applicationcontext)
   - [4.2 Types of ApplicationContext](#42-types-of-applicationcontext)
   - [4.3 Using ApplicationContext Programmatically](#43-using-applicationcontext-programmatically)
   - [4.4 Application Event Architecture & Decoupling (4 Approaches)](#44-application-event-architecture--decoupling)
5. [Bean Lifecycle & Startup Code Execution](#5-bean-lifecycle--startup-code-execution)
   - [5.1 Complete 8-Phase Bean Lifecycle Flow](#51-complete-8-phase-bean-lifecycle-flow)
   - [5.2 Lifecycle Demonstration Code](#52-lifecycle-demonstration-code)
   - [5.3 Application Startup Code Execution (6 Approaches)](#53-application-startup-code-execution-6-distinct-approaches)
   - [5.4 Custom Init and Destroy Methods](#54-custom-init-and-destroy-methods)
6. [Bean Scopes & The Prototype-in-Singleton Problem](#6-bean-scopes--the-prototype-in-singleton-problem)
   - [6.1 Overview of All Scopes](#61-overview-of-all-scopes)
   - [6.2 Singleton Scope & Thread-Safety](#62-singleton-scope--thread-safety)
   - [6.3 Prototype Scope](#63-prototype-scope)
   - [6.4 Prototype Bean Injected into Singleton (4 Solutions)](#64-prototype-bean-injected-into-singleton-4-solutions)
   - [6.5 Request and Session Scopes](#65-request-and-session-scopes)
7. [@SpringBootApplication Composition](#7-springbootapplication-composition)
8. [Auto-Configuration Mechanics](#8-auto-configuration-mechanics)
9. [SpringApplication.run() Pipeline](#9-springapplicationrun-pipeline)
10. [Environment Profiles Activation (5 Approaches)](#10-environment-profiles-activation-5-approaches)
11. [Configuration & Property Injection (4 Approaches)](#11-configuration--property-injection-4-approaches)
12. [Spring Boot Starters & Dependency BOM](#12-spring-boot-starters--dependency-bom)
13. [⚠️ High-Frequency Interview Traps & Gotchas](#13-️-high-frequency-interview-traps--gotchas)
14. [🔍 Deep Dive — Internal Mechanics](#14--deep-dive--internal-mechanics)
15. [📝 Architecture Principle Checklist](#15--architecture-principle-checklist)
16. [❓ Practice Questions & Answers](#16--practice-questions--answers)

---

## 1. Spring Boot vs Spring Framework

### 1.1 The Problem Spring Boot Solves

Before Spring Boot (pre-2014), setting up an enterprise Spring application required massive configuration overhead:
- Manually declaring hundreds of lines of XML or Java `@Configuration` classes.
- Managing incompatible transitive dependency versions in build descriptors.
- Packaging external WAR files and manually configuring standalone Servlet containers (Tomcat, WebLogic, JBoss).
- Explicitly defining the `DispatcherServlet`, `ViewResolver`, `DataSource`, `TransactionManager`, and Jackson mappers.

Spring Boot was created to apply **opinionated defaults** and **convention over configuration**, shifting developer focus from infrastructure wiring to business domain logic.

### 1.2 Comparison Matrix

| Architectural Dimension | Spring Framework | Spring Boot 3.x |
|---|---|---|
| **Paradigm** | Core IoC and DI container with modular libraries | Opinionated, production-ready framework built atop Spring Framework 6 |
| **Configuration** | Explicit XML or Java `@Configuration` required | Automated auto-configuration based on classpath contents |
| **Dependency Management** | Manual version selection for every artifact | Curated BOM (`spring-boot-dependencies`) guaranteeing compatibility |
| **Deployment Model** | External WAR deployed to standalone application servers | Executable self-contained "fat JAR" with embedded Tomcat/Jetty/Undertow |
| **Observability** | Requires external libraries and manual plumbing | First-class Micrometer metrics, OpenTelemetry tracing, and Actuator endpoints |
| **Namespace Baseline** | Legacy `javax.*` (Spring 5.x) | Modern `jakarta.*` (Jakarta EE 10 baseline in Spring 6 / Boot 3) |
| **Java Baseline** | Java 8+ | Java 17 or Java 21+ (Virtual Threads / Project Loom native) |

---

## 2. Inversion of Control (IoC)

### 2.1 What Is IoC?

**Inversion of Control (IoC)** is an architectural pattern in which the **control flow of a program is inverted**. Instead of application code creating its own dependencies, instantiating classes, and managing their lifecycles, the framework (the IoC container) takes control.

```
Traditional Flow (No IoC):
Caller Code ──► new ServiceA() ──► new ServiceB() ──► new DatabaseConnection()
(Tightly coupled, hard-coded dependencies, impossible to mock in unit tests)

Inverted Flow (Spring IoC Container):
┌────────────────────────────────────────────────────────┐
│               Spring IoC Container                     │
│  Instantiates Beans ──► Resolves Graph ──► Injects     │
└──────────────────────────┬─────────────────────────────┘
                           ▼
Caller Code receives ready-to-use Collaborator: (OrderService ◄── [OrderRepository])
```

### 2.2 Why IoC Matters in Enterprise Systems
1. **Separation of Concerns**: Classes focus purely on business execution rather than dependency lifecycle management.
2. **Deterministic Mockability**: Dependencies can be seamlessly swapped with mocks during unit and slice testing without reflection hacks.
3. **Architectural Flexibility**: Changing an implementation (e.g. from `InMemoryOrderRepository` to `PostgresOrderRepository`) requires zero changes to the consuming domain service.

---

## 3. Dependency Injection (DI) Approaches

Dependency Injection is the **concrete mechanism** through which Spring realizes Inversion of Control. In modern Spring Boot 3.x, dependencies can be supplied through five distinct approaches.

---

### 3.1 Approach 1: Constructor-Based Dependency Injection (Recommended Standard)

#### Scenario
- Mandatory, immutable collaborators required for the component to function correctly.
- Production business services, repositories, security providers, and controllers where null safety and thread safety are non-negotiable.

#### Use-Case
An enterprise `OrderProcessingService` that strictly requires an `OrderRepository`, a `PaymentGateway`, and an `AuditService`. If any of these are missing, the service is conceptually in an invalid state and must fail during application bootstrap.

#### Benefits Over Others
- **Guaranteed Immutability**: All dependency fields can be declared `final`, preventing unintended mutation after construction.
- **Fail-Fast Startup**: The Spring container immediately fails during bootstrap (`NoSuchBeanDefinitionException`) if any required dependency cannot be satisfied.
- **Superior Testability**: Plain Java unit tests can instantiate the class using `new OrderProcessingService(mockRepo, mockPayment, mockAudit)` without needing Spring context runners, reflection, or Mockito annotations.
- **Compile-Time Contract Clarity**: The constructor signature explicitly documents all dependencies required by the class.
- **Early Circular Dependency Detection**: Since Spring Boot 2.6+, circular dependencies in constructors are detected and blocked at startup with `BeanCurrentlyInCreationException`.

#### Well-Documented Code Example

```java
package com.enterprise.order.service;

import com.enterprise.order.domain.Order;
import com.enterprise.order.domain.OrderRequest;
import com.enterprise.order.domain.PaymentResult;
import com.enterprise.order.repository.OrderRepository;
import com.enterprise.payment.service.PaymentGateway;
import com.enterprise.audit.service.AuditService;
import org.springframework.stereotype.Service;

import java.util.Objects;

@Service
public class OrderProcessingService {

    // 1. All dependencies are immutable (final)
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final AuditService auditService;

    /**
     * Single constructor injection.
     * Note: In Spring 4.3+ and Spring Boot 3.x, @Autowired is completely OPTIONAL 
     * on classes with a single constructor. Spring automatically detects and injects parameters.
     */
    public OrderProcessingService(OrderRepository orderRepository,
                                  PaymentGateway paymentGateway,
                                  AuditService auditService) {
        // Defensive validation ensures absolute null safety even if instantiated manually in unit tests
        this.orderRepository = Objects.requireNonNull(orderRepository, "orderRepository must not be null");
        this.paymentGateway = Objects.requireNonNull(paymentGateway, "paymentGateway must not be null");
        this.auditService = Objects.requireNonNull(auditService, "auditService must not be null");
    }

    public Order processOrder(OrderRequest request) {
        PaymentResult payment = paymentGateway.charge(request.customerId(), request.amount());
        Order order = new Order(request, payment.transactionId());
        Order savedOrder = orderRepository.save(order);
        auditService.recordEvent("ORDER_PROCESSED", savedOrder.getId());
        return savedOrder;
    }
}
```

---

### 3.2 Approach 2: Setter-Based Dependency Injection (For Optional Collaborators)

#### Scenario
- Optional dependencies, fallback strategies, reconfigurable parameters, or resolving unavoidable legacy circular dependencies.

#### Use-Case
A `ReportGenerationService` that requires a mandatory `ReportFormatter` (injected via constructor) but optionally accepts a `CloudStorageExporter`. If the exporter is not configured, the service defaults to writing reports to the local file system.

#### Benefits Over Others
- **Flexibility for Optional Collaborators**: Allows dependencies to be injected only if present in the environment or configuration.
- **Reconfigurability**: Enables properties to be re-injected or modified dynamically at runtime via JMX or actuator endpoints.
- **Breaks Circular Dependencies**: Unlike constructor injection which deadlocks at startup, setter injection allows Spring to instantiate both beans first and wire them afterward.

#### Well-Documented Code Example

```java
package com.enterprise.report.service;

import com.enterprise.report.domain.Report;
import com.enterprise.report.domain.ReportRequest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Objects;

@Service
public class ReportGenerationService {

    // Mandatory collaborator (must be final)
    private final ReportFormatter formatter;

    // Optional collaborator (cannot be final because it is injected via setter)
    private CloudStorageExporter cloudExporter;

    // 1. Mandatory dependency injected via constructor
    public ReportGenerationService(ReportFormatter formatter) {
        this.formatter = Objects.requireNonNull(formatter, "formatter is required");
    }

    /**
     * 2. Optional dependency injected via setter.
     * required = false tells Spring not to throw NoSuchBeanDefinitionException if no bean matches.
     */
    @Autowired(required = false)
    public void setCloudStorageExporter(CloudStorageExporter cloudExporter) {
        this.cloudExporter = cloudExporter;
    }

    public Report generate(ReportRequest request) {
        Report report = formatter.format(request);

        // Safe evaluation of the optional collaborator
        if (this.cloudExporter != null) {
            cloudExporter.exportToS3(report);
        } else {
            writeToLocalStorage(report);
        }
        return report;
    }

    private void writeToLocalStorage(Report report) {
        // Fallback local file writing logic
    }
}
```

---

### 3.3 Approach 3: Field-Based Dependency Injection (Anti-Pattern)

#### Scenario
- Rapid prototyping, proof-of-concept throwaway code, or quick test scaffolding where architectural rigor is explicitly sacrificed for speed.

#### Use-Case
Quick spike testing in a temporary scratchpad test class.

#### Why Field Injection is Discouraged in Production
- **Violates Encapsulation & Immutability**: Fields cannot be declared `final`.
- **Hinders Unit Testing**: Outside of a Spring context, `new ProductService()` leaves fields `null`. Injecting mocks requires heavy reflection tools (`ReflectionTestUtils.setField()`) or starting a full Spring Test context.
- **Hides Dependency Bloat**: Adding `@Autowired` lines makes it easy to accumulate 10+ hidden dependencies without noticing violations of the Single Responsibility Principle.
- **Masks Circular Dependencies**: Delays detection until runtime errors or mysterious `NullPointerException`s emerge.

#### Well-Documented Code Example

```java
package com.enterprise.catalog.service;

import com.enterprise.catalog.domain.Product;
import com.enterprise.catalog.repository.ProductRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    // ❌ ANTI-PATTERN: Field injection uses private reflection at container startup
    @Autowired
    private ProductRepository productRepository;

    public Product getById(Long id) {
        // If instantiated directly via 'new ProductService()', this throws NullPointerException!
        return productRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Product not found"));
    }
}
```

---

### 3.4 Approach 4: Factory Method Injection (`@Bean` Method Parameters)

#### Scenario
- Registering third-party library classes, SDK clients (e.g., AWS S3 Client, Stripe SDK, Kafka Producer), or dynamic configurations where you do not own the source code.

#### Use-Case
Configuring a third-party `S3Client` where credentials, timeouts, and metrics must be pulled from Spring-managed beans and passed to a builder pattern.

#### Benefits Over Others
- **Works with Unmodifiable Source Code**: Injects dependencies into third-party classes that don't have Spring stereotype annotations.
- **Explicit Parameter Passing**: Spring resolves all method parameters from the `ApplicationContext` before invoking the `@Bean` method.
- **Centralized Factory Logic**: Allows conditional construction, custom builders, and shutdown hooks (`destroyMethod`).

#### Well-Documented Code Example

```java
package com.enterprise.config;

import com.enterprise.config.properties.AwsProperties;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;

@Configuration(proxyBeanMethods = false) // High-performance configuration mode
public class AwsStorageConfiguration {

    /**
     * Factory Method Injection:
     * Spring inspects the parameters of the @Bean method (AwsProperties and MeterRegistry)
     * and automatically resolves and injects them from the ApplicationContext.
     */
    @Bean(destroyMethod = "close")
    public S3Client s3Client(AwsProperties awsProperties, MeterRegistry meterRegistry) {
        AwsBasicCredentials credentials = AwsBasicCredentials.create(
                awsProperties.getAccessKey(),
                awsProperties.getSecretKey()
        );

        S3Client client = S3Client.builder()
                .region(Region.of(awsProperties.getRegion()))
                .credentialsProvider(StaticCredentialsProvider.create(credentials))
                .build();

        // Register custom gauge or counter for client lifecycle
        meterRegistry.counter("aws.s3.client.initialized").increment();
        return client;
    }
}
```

---

### 3.5 Approach 5: Method Injection via `@Lookup` (Dynamic Prototype Creation)

#### Scenario
- A **Singleton** scoped bean needs to obtain a fresh instance of a **Prototype** scoped bean on every method execution without injecting the entire `ApplicationContext`.

#### Use-Case
A singleton `AsyncNotificationDispatcher` needs to generate a stateful, non-thread-safe `MessageEncodingSession` for every outbound notification task.

#### Benefits Over Others
- **Clean Object-Oriented Code**: Avoids coupling business code to the Spring `ApplicationContext` API (unlike `applicationContext.getBean()`).
- **Eliminates Manual Factory Boilerplate**: Spring automatically overrides the annotated method at runtime using CGLIB bytecode generation.
- **Guaranteed Fresh Instance**: Every call to the method invokes the container's prototype bean generation logic.

#### Well-Documented Code Example

```java
package com.enterprise.notification.service;

import com.enterprise.notification.domain.Notification;
import org.springframework.beans.factory.annotation.Lookup;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

import java.util.UUID;

// 1. Prototype-scoped stateful helper (new instance per request)
@Component
@Scope("prototype")
public class MessageEncodingSession {
    private final String sessionId = UUID.randomUUID().toString();

    public String encode(String rawPayload) {
        return "[" + sessionId + "] " + rawPayload.trim();
    }
}

// 2. Singleton-scoped business service
@Service
public abstract class NotificationDispatcher {

    /**
     * Method Injection:
     * Spring generates a CGLIB subclass of NotificationDispatcher and overrides
     * createSession() to return applicationContext.getBean(MessageEncodingSession.class).
     */
    @Lookup
    public abstract MessageEncodingSession createSession();

    public void dispatch(Notification notification) {
        // Every invocation gets a completely fresh prototype instance
        MessageEncodingSession session = createSession();
        String encoded = session.encode(notification.getContent());
        System.out.println("Dispatched encoded message: " + encoded);
    }
}
```

---

### Summary Comparison of Dependency Injection Approaches

| Metric | Constructor Injection | Setter Injection | Field Injection | Factory Method (`@Bean`) | `@Lookup` Method |
|---|---|---|---|---|---|
| **Primary Intent** | Required, immutable dependencies | Optional, configurable dependencies | Quick experiments / test scaffolding | Third-party object graph assembly | Prototype inside Singleton |
| **Immutability (`final`)** | ✅ Fully supported | ❌ Impossible | ❌ Impossible | ✅ Fully supported | N/A (Method level) |
| **Fail-Fast at Startup** | ✅ Yes | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Testability without Spring**| ✅ Trivial (`new Service(...)`) | ⚠️ Moderate (call setters) | ❌ Difficult (needs reflection) | ✅ Trivial | ⚠️ Requires mocking abstract method |
| **Circular Dependency Handling**| ❌ Throws startup exception | ✅ Successfully resolves | ⚠️ Silently resolves (risky) | ❌ Throws startup exception | ✅ Immune |
| **Production Recommendation** | 🌟 **Top Recommendation** | ⚠️ Use only for optional | 🚫 **Avoid in production** | 🌟 **Standard for 3rd-party**| 🌟 **Standard for prototype**|

---

### 3.6 Resolving Bean Ambiguity & Polymorphic Wiring Approaches

When multiple beans implement the same interface, Spring cannot determine which bean to inject by type alone. Spring Boot provides six distinct architectural approaches to resolve this ambiguity.

```
                  Interface: PaymentGateway
                             ▲
         ┌───────────────────┴───────────────────┐
         │                                       │
StripePaymentGateway                    PayPalPaymentGateway
(@Primary / @Qualifier("stripe"))       (@Qualifier("paypal"))
```

---

#### Approach 1: `@Primary` Annotation (Designating the Default Candidate)

##### Scenario
- Multiple implementations exist, but one serves as the default enterprise standard while others are specialized exceptions.

##### Use-Case
In an e-commerce platform, `StripePaymentGateway` is the default processor for 95% of domestic transactions, but `PayPalPaymentGateway` is also available.

##### Benefits Over Others
- Non-invasive: Callers that don't care about specialization simply declare `PaymentGateway` without any qualifier annotations.
- Easily overridden: A caller can still explicitly request a non-primary bean using `@Qualifier`.

##### Documented Code Example

```java
package com.enterprise.payment.service;

import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Service;

public interface PaymentGateway {
    void charge(double amount);
}

// 1. Marked as @Primary — the default choice
@Service
@Primary
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Processing $" + amount + " via Stripe");
    }
}

// 2. Secondary implementation
@Service
public class PayPalPaymentGateway implements PaymentGateway {
    @Override
    public void charge(double amount) {
        System.out.println("Processing $" + amount + " via PayPal");
    }
}

// 3. Injecting without qualifier receives the @Primary bean
@Service
public class StandardCheckoutService {
    private final PaymentGateway defaultGateway;

    public StandardCheckoutService(PaymentGateway defaultGateway) {
        this.defaultGateway = defaultGateway; // Automatically receives StripePaymentGateway
    }
}
```

---

#### Approach 2: `@Qualifier` with String Literals

##### Scenario
- Specific callers require precise, named bean bindings where no single implementation can be considered the "default".

##### Use-Case
A customer-facing checkout service explicitly routing PayPal transactions to `PayPalPaymentGateway` via qualifier tags.

##### Benefits Over Others
- Explicit contract: Directly specifies the target bean identifier.
- Higher precedence than `@Primary`: If a bean has both `@Primary` and `@Qualifier`, a caller using `@Qualifier` always selects the qualified bean.

##### Documented Code Example

```java
package com.enterprise.payment.service;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class PayPalCheckoutService {

    private final PaymentGateway paymentGateway;

    // Explicitly binds to the bean named "payPalPaymentGateway"
    public PayPalCheckoutService(@Qualifier("payPalPaymentGateway") PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void processUserPayment(double amount) {
        this.paymentGateway.charge(amount);
    }
}
```

---

#### Approach 3: Custom Meta-Annotation Qualifiers (Type-Safe Disambiguation)

##### Scenario
- Enterprise systems where string literals in `@Qualifier("string")` are prone to typos, refactoring errors, and lack compile-time safety.

##### Use-Case
A multi-cloud deployment where components are specifically targeted for AWS, Azure, or GCP infrastructure.

##### Benefits Over Others
- **Compile-Time Type Safety**: Refactoring tools and IDEs track annotations across the entire codebase.
- **Clean Code Aesthetics**: Removes raw string literals from constructor parameters.

##### Documented Code Example

```java
package com.enterprise.cloud.annotations;

import org.springframework.beans.factory.annotation.Qualifier;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

// 1. Define custom meta-annotation qualifiers
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface CloudVendor {
    VendorType value();
    enum VendorType { AWS, AZURE, GCP }
}
```

```java
package com.enterprise.cloud.service;

import com.enterprise.cloud.annotations.CloudVendor;
import static com.enterprise.cloud.annotations.CloudVendor.VendorType.*;
import org.springframework.stereotype.Service;

public interface CloudStorageService {
    void store(String fileId, byte[] data);
}

@Service
@CloudVendor(AWS)
public class S3StorageService implements CloudStorageService {
    @Override
    public void store(String fileId, byte[] data) {
        System.out.println("Stored to AWS S3: " + fileId);
    }
}

@Service
@CloudVendor(AZURE)
public class AzureBlobStorageService implements CloudStorageService {
    @Override
    public void store(String fileId, byte[] data) {
        System.out.println("Stored to Azure Blob: " + fileId);
    }
}

// 2. Type-safe injection without raw strings
@Service
public class BackupManager {
    private final CloudStorageService primaryStorage;

    public BackupManager(@CloudVendor(AWS) CloudStorageService primaryStorage) {
        this.primaryStorage = primaryStorage; // Type-safe binding!
    }
}
```

---

#### Approach 4: Fallback Matching by Parameter or Field Name

##### Scenario
- Legacy codebases or simple setups where neither `@Primary` nor `@Qualifier` is declared, but constructor argument names match target bean names.

##### Use-Case
Matching beans directly using Java compiler parameter names (`-parameters` compiler flag).

##### Benefits Over Others
- Zero extra annotations required.
- Relies on Spring's built-in fallback resolution rule: `type match -> name match -> fail`.

##### Documented Code Example

```java
package com.enterprise.payment.service;

import org.springframework.stereotype.Service;

@Service
public class NamedParameterCheckoutService {

    private final PaymentGateway stripePaymentGateway;

    /**
     * Spring matches the parameter name 'stripePaymentGateway' to the bean named 'stripePaymentGateway'.
     * Requirement: Code must be compiled with the '-parameters' flag (default in Spring Boot 3.x).
     */
    public NamedParameterCheckoutService(PaymentGateway stripePaymentGateway) {
        this.stripePaymentGateway = stripePaymentGateway;
    }
}
```

---

#### Approach 5: Strategy Pattern via Collection & Map Injection

##### Scenario
- Dynamic algorithm execution where the system chooses an implementation at runtime based on user input, request headers, or database flags.

##### Use-Case
A universal `PaymentRouter` that receives an HTTP request containing a `paymentMethod` field (`"stripe"`, `"paypal"`, `"crypto"`) and routes the charge dynamically without `if-else` or `switch` blocks.

##### Benefits Over Others
- **Open/Closed Principle (SOLID)**: Adding a new payment provider only requires creating a new `@Service` class; the `PaymentRouter` code never changes.
- **Zero Configuration Boilerplate**: Spring automatically populates `Map<String, T>` where the keys are Spring bean names and values are the bean instances.

##### Documented Code Example

```java
package com.enterprise.payment.service;

import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class DynamicPaymentRouter {

    // Spring injects all beans implementing PaymentGateway into a Map
    // Key: Bean name (e.g. "stripePaymentGateway", "payPalPaymentGateway")
    // Value: The concrete bean instance
    private final Map<String, PaymentGateway> paymentGateways;

    public DynamicPaymentRouter(Map<String, PaymentGateway> paymentGateways) {
        this.paymentGateways = paymentGateways;
    }

    public void executePayment(String providerName, double amount) {
        PaymentGateway gateway = paymentGateways.get(providerName.toLowerCase() + "PaymentGateway");
        if (gateway == null) {
            throw new IllegalArgumentException("Unsupported payment provider: " + providerName);
        }
        gateway.charge(amount);
    }
}
```

---

#### Approach 6: Dynamic Resolution via `ObjectProvider<T>`

##### Scenario
- Conditional or stream-based bean lookups, optional singletons, or prototype factory queries without throwing exceptions if no candidate exists.

##### Use-Case
An audit subsystem that filters all registered security inspection beans, executes only those matching specific criteria, or falls back to a no-op implementation if none are registered.

##### Benefits Over Others
- **Safe Null Handling**: Eliminates `NoSuchBeanDefinitionException` through `getIfAvailable()` and `getIfUnique()`.
- **Stream Support**: Enables functional Java 8+ stream filtering on all matching beans via `orderedStream()`.

##### Documented Code Example

```java
package com.enterprise.audit.service;

import com.enterprise.payment.service.PaymentGateway;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;

@Service
public class ResilientPaymentCoordinator {

    private final ObjectProvider<PaymentGateway> gatewayProvider;

    public ResilientPaymentCoordinator(ObjectProvider<PaymentGateway> gatewayProvider) {
        this.gatewayProvider = gatewayProvider;
    }

    public void process(double amount) {
        // 1. Safe retrieval: returns null if missing, throws if multiple non-primary exist
        PaymentGateway gateway = gatewayProvider.getIfAvailable(() -> new FallbackPaymentGateway());
        gateway.charge(amount);

        // 2. Functional iteration over all matching beans
        gatewayProvider.orderedStream().forEach(g -> {
            System.out.println("Registered gateway candidate: " + g.getClass().getSimpleName());
        });
    }

    private static class FallbackPaymentGateway implements PaymentGateway {
        @Override
        public void charge(double amount) {
            System.out.println("[FALLBACK] Storing transaction offline: $" + amount);
        }
    }
}
```

---

### Comparison of Bean Ambiguity Resolution Approaches

| Approach | Type Safety | Flexibility | Dynamic at Runtime | Open/Closed Friendly | Best For |
|---|---|---|---|---|---|
| **`@Primary`** | ✅ High | ⚠️ Low (single global default) | ❌ Static | ⚠️ Moderate | Default provider when specialized ones exist |
| **`@Qualifier`** | ⚠️ Medium (String) | ✅ High (explicit target) | ❌ Static | ❌ Low (hardcodes string) | Targeting a specific known bean |
| **Custom Meta-Qualifier**| ✅ Very High | ✅ High | ❌ Static | ⚠️ Moderate | Enterprise domain classification (Cloud, Tier) |
| **Parameter Name Match**| ⚠️ Low (implicit) | ⚠️ Low | ❌ Static | ❌ Fragile (name change breaks it) | Small prototypes with unique variable names |
| **Map / List Injection** | ✅ High | 🌟 Maximum | 🌟 Full Runtime Choice | 🌟 Absolute (Zero code changes for new beans) | Strategy Pattern, Plugin architectures |
| **`ObjectProvider<T>`** | ✅ High | 🌟 Maximum | 🌟 Full Runtime Choice | ✅ High | Optional plugins, stream processing, fallbacks |

---

### 3.7 Lazy Injection (Performance & Circular Reference Mitigation)

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

## 4. Application Context & Event Architecture

### 4.1 BeanFactory vs ApplicationContext

```
BeanFactory (core interface: basic lifecycle & DI)
    └── ApplicationContext (advanced enterprise container)
            ├── ConfigurableApplicationContext
            │       ├── AnnotationConfigApplicationContext
            │       └── AnnotationConfigServletWebServerApplicationContext (Boot MVC default)
            └── WebApplicationContext
```

| Feature | BeanFactory | ApplicationContext |
|---|---|---|
| **Bean Instantiation & DI** | ✅ Yes (lazy on getBean by default) | ✅ Yes (eager singleton pre-instantiation) |
| **Automatic BPP / BFPP Registration** | ❌ Manual registration required | ✅ Automatically detected and registered |
| **Internationalization (i18n)** | ❌ Unsupported | ✅ Full `MessageSource` support |
| **Event Propagation** | ❌ Unsupported | ✅ Full `ApplicationEventPublisher` integration |
| **AOP & Proxy Generation** | ⚠️ Complex manual wiring | ✅ Declarative via registered AutoProxyCreators |

---

### 4.2 Types of ApplicationContext in Spring Boot

Spring Boot automatically deduces the type of context to instantiate based on classpath analysis in `SpringApplication.deduceWebApplicationType()`:
1. **`SERVLET`** (`AnnotationConfigServletWebServerApplicationContext`): Triggered when `spring-boot-starter-web` (Tomcat/Jetty) is on the classpath.
2. **`REACTIVE`** (`AnnotationConfigReactiveWebServerApplicationContext`): Triggered when `spring-boot-starter-webflux` (Netty) is present without MVC.
3. **`NONE`** (`AnnotationConfigApplicationContext`): Used for standalone CLI utilities, background batch processors, or daemon jobs.

---

### 4.3 Using ApplicationContext Programmatically

```java
package com.enterprise.util;

import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class ContextInspector implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    public void printLoadedBeans() {
        String[] beanNames = applicationContext.getBeanDefinitionNames();
        System.out.println("Total Registered Beans: " + beanNames.length);
        for (String name : beanNames) {
            System.out.printf("Bean Name: %-40s | Type: %s%n", name, applicationContext.getType(name));
        }
    }
}
```

---

### 4.4 Application Event Architecture & Decoupling

Spring's event-driven subsystem implements the **Observer design pattern**, allowing loosely coupled communication between application components within the same JVM context.

Spring Boot 3.x supports **four primary approaches** to publishing and handling application events:

---

#### 4.4.1 Approach 1: Synchronous In-Memory Events (`@EventListener`)

##### Scenario
- Synchronous audit logging, cache eviction, or business reactions that must run immediately on the same thread before the calling method returns.

##### Use-Case
An order creation process where an `OrderPlacedEvent` immediately updates an in-memory inventory metric on the publisher's thread.

##### Benefits Over Others
- **Thread Local Preservation**: Inherits the publisher's `SecurityContextHolder`, `MDC` logging context, and transaction context directly.
- **Zero Configuration**: No executor thread pools or async infrastructure required.
- **Ordered Execution**: Multiple listeners can be ordered deterministically using `@Order(1)`, `@Order(2)`.

##### Documented Code Example

```java
package com.enterprise.order.event;

import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.event.EventListener;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

// 1. Plain Java domain event (In Spring 4.2+, extending ApplicationEvent is completely OPTIONAL)
public record OrderCreatedEvent(Long orderId, String customerEmail, double amount) {}

// 2. Event Publisher
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void createOrder(Long orderId, String email, double amount) {
        // Business logic...
        System.out.println("Order created: " + orderId);

        // Publishes synchronously: all @EventListener methods execute before publishEvent returns
        eventPublisher.publishEvent(new OrderCreatedEvent(orderId, email, amount));
        System.out.println("Synchronous event listeners completed.");
    }
}

// 3. Synchronous Event Listener
@Component
public class InventoryAuditListener {

    @EventListener
    @Order(1) // Runs first
    public void onOrderCreated(OrderCreatedEvent event) {
        System.out.println("Audit listener: recording audit for order #" + event.orderId());
    }
}
```

---

#### 4.4.2 Approach 2: Asynchronous Event Processing (`@Async @EventListener`)

##### Scenario
- Fire-and-forget side effects, slow I/O operations (sending email/SMS, generating PDF receipts, third-party webhook dispatch).

##### Use-Case
Dispatching welcome emails and SMS alerts after order placement without adding 800ms of external network latency to the user's HTTP response.

##### Benefits Over Others
- **Zero Request Thread Blocking**: The publisher thread returns immediately to client.
- **Isolated Thread Pool Execution**: Failures in the async listener do not abort or roll back the primary caller's workflow.

##### Documented Code Example

```java
package com.enterprise.notification.listener;

import com.enterprise.order.event.OrderCreatedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class AsyncNotificationListener {

    /**
     * @Async ensures this method runs in a dedicated thread from 'taskExecutor'.
     * Requires @EnableAsync on a configuration class.
     */
    @Async("notificationExecutor")
    @EventListener
    public void sendEmailConfirmation(OrderCreatedEvent event) {
        System.out.println("[" + Thread.currentThread().getName() + "] Sending email to " + event.customerEmail());
        try {
            Thread.sleep(1500); // Simulate external SMTP latency
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("Email successfully sent for order: " + event.orderId());
    }
}
```

---

#### 4.4.3 Approach 3: Transaction-Bound Events (`@TransactionalEventListener`)

##### Scenario
- Crucial side effects (e.g. publishing to Kafka, debiting external wallets, scheduling background workers) that **must only execute if the surrounding database transaction successfully COMMITS**.

##### Use-Case
Sending an order confirmation message to Kafka. If the database transaction rolls back due to an SQL constraint violation, the Kafka message must **never** be sent.

##### Benefits Over Others
- **Prevents Ghost Side-Effects**: Normal `@EventListener` fires when `publishEvent()` is called, even if the transaction subsequently rolls back! `@TransactionalEventListener(phase = AFTER_COMMIT)` guarantees the DB changes are permanent before triggering side effects.
- **Fine-Grained Phase Control**: Supports `AFTER_COMMIT` (default), `BEFORE_COMMIT`, `AFTER_ROLLBACK`, and `AFTER_COMPLETION`.

##### Documented Code Example

```java
package com.enterprise.order.listener;

import com.enterprise.order.event.OrderCreatedEvent;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Component
public class TransactionalOrderListener {

    /**
     * Fires strictly AFTER the surrounding @Transactional method has successfully committed to DB.
     * If the transaction rolls back, this listener is NEVER invoked.
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publishToKafkaAfterCommit(OrderCreatedEvent event) {
        System.out.println("Transaction COMMITTED. Safe to dispatch Kafka event for order: " + event.orderId());
    }

    /**
     * Fires only when the database transaction rolls back (e.g., error recovery, metric increment).
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderRollback(OrderCreatedEvent event) {
        System.err.println("Transaction ROLLED BACK for order: " + event.orderId() + ". Compensating actions triggered.");
    }
}
```

---

#### 4.4.4 Approach 4: Generic Application Events (Decoupled Typed Payloads)

##### Scenario
- Domain-driven design (DDD) microservices where domain classes publish pure POJO events without any dependencies on the Spring framework.

##### Use-Case
A reusable core domain module shared across multiple teams where classes cannot import `org.springframework.context.ApplicationEvent`.

##### Benefits Over Others
- **Clean Architecture & Domain Purity**: Pure Java `record` or `class` with zero framework dependencies.
- **Type-Safe Filtering with SpEL**: Allows conditional filtering directly on event fields using `condition = "#event.amount > 1000"`.

##### Documented Code Example

```java
package com.enterprise.order.listener;

import com.enterprise.order.event.OrderCreatedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class HighValueOrderListener {

    /**
     * SpEL Condition Filtering:
     * This listener only triggers if the event payload satisfies the condition expression.
     */
    @EventListener(condition = "#event.amount() >= 10000.00")
    public void alertVipAccountManager(OrderCreatedEvent event) {
        System.out.println("VIP ALERT: High-value order placed ($" + event.amount() + ") by " + event.customerEmail());
    }
}
```

---

### Comparison of Application Event Approaches

| Feature | Synchronous `@EventListener` | Asynchronous `@Async @EventListener` | `@TransactionalEventListener` |
|---|---|---|---|
| **Execution Thread** | Caller / Publisher Thread | Dedicated Worker Thread | Caller Thread (at TX boundary) |
| **Transaction Visibility** | Shares active transaction | New or No transaction | Bound to Phase (`AFTER_COMMIT`, etc.) |
| **Failure Impact** | Aborts caller transaction | Logged / Handled asynchronously | Cannot roll back already committed DB |
| **Best For** | In-memory cache updates, validation | Outbound I/O (Emails, SMS, Push) | Kafka outbox dispatch, external mutations |

---

## 5. Bean Lifecycle & Startup Code Execution

### 5.1 Complete 8-Phase Bean Lifecycle Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SPRING BEAN LIFECYCLE                           │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Instantiation (Constructor or Factory Method called)                │
│    │                                                                   │
│ 2. Populate Properties (Setter / Field Injection completed)            │
│    │                                                                   │
│ 3. Aware Callbacks (BeanNameAware -> BeanFactoryAware -> AppContext)   │
│    │                                                                   │
│ 4. BPP Before-Initialization (BeanPostProcessor.postProcessBeforeInit) │
│    │  └── @PostConstruct executed by CommonAnnotationBPP               │
│    │                                                                   │
│ 5. Initialization Callbacks                                            │
│    │  ├── InitializingBean.afterPropertiesSet()                        │
│    │  └── Custom initMethod defined in @Bean                           │
│    │                                                                   │
│ 6. BPP After-Initialization (BeanPostProcessor.postProcessAfterInit)   │
│    │  └── CGLIB AOP Dynamic Proxies created here (Async, Tx, Cache)   │
│    │                                                                   │
│ 7. Bean Ready for Use (Served to callers / Application runs)           │
│    │                                                                   │
│ 8. Destruction Phase (Container Shutdown)                              │
│       ├── @PreDestroy executed                                         │
│       ├── DisposableBean.destroy()                                     │
│       └── Custom destroyMethod defined in @Bean                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 5.2 Lifecycle Demonstration Code

```java
package com.enterprise.lifecycle;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class LifecycleInspectionBean implements 
        BeanNameAware, BeanFactoryAware, ApplicationContextAware, 
        InitializingBean, DisposableBean {

    public LifecycleInspectionBean() {
        System.out.println("[Step 1] Instantiation: Constructor invoked.");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("[Step 3a] BeanNameAware: Bean ID assigned -> " + name);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("[Step 3b] BeanFactoryAware: Injected BeanFactory reference.");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("[Step 3c] ApplicationContextAware: Injected ApplicationContext.");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("[Step 4/5a] @PostConstruct: Invoked by CommonAnnotationBeanPostProcessor.");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("[Step 5b] InitializingBean: afterPropertiesSet() invoked.");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("[Step 8a] @PreDestroy: Releasing open connections/file locks.");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("[Step 8b] DisposableBean: destroy() executed on shutdown.");
    }
}
```

---

### 5.3 Application Startup Code Execution (6 Distinct Approaches)

When executing startup tasks (e.g. database schema migrations, cache warmup, message queue listener registration), developers have six different approaches in Spring Boot.

---

#### Approach 1: `@PostConstruct` Callback Method
- **Scenario**: Local, bean-specific internal variable initialization immediately after property population.
- **Use-Case**: Pre-computing regex patterns, verifying that a mandatory secret key has an allowable bit length.
- **Benefits**: Self-contained within the bean; runs automatically without extra interfaces.
- **Limitation**: **Cannot use `@Transactional` or `@Async`** reliably because AOP proxies are created in Step 6 (BPP After-Init), which occurs *after* `@PostConstruct` (Step 4)!

```java
@Service
public class TokenSigner {
    private SecretKey secretKey;

    @PostConstruct
    public void generateKey() {
        // Safe internal initialization
        this.secretKey = Keys.secretKeyFor(SignatureAlgorithm.HS256);
    }
}
```

---

#### Approach 2: `InitializingBean.afterPropertiesSet()`
- **Scenario**: Framework-level components requiring standard interface compliance without annotation processors.
- **Use-Case**: Reusable shared libraries or legacy Spring components.
- **Benefits**: Avoids annotation reflection overhead.
- **Limitation**: Couples class to Spring's `InitializingBean` interface; runs prior to AOP proxying.

---

#### Approach 3: `ApplicationRunner` (Parsed CLI Arguments)
- **Scenario**: Post-bootstrap database warmup or cache loading requiring strongly typed command-line arguments (`--flag=value`).
- **Use-Case**: Running a data migration runner that checks for `--reindex-elasticsearch=true`.
- **Benefits**:
  - Runs **after the entire `ApplicationContext` is fully initialized** (AOP proxies are active, `@Transactional` works!).
  - Injects `ApplicationArguments` providing parsed option arguments (`args.getOptionValues("env")`) and non-option arguments.
  - Can be ordered using `@Order(1)`.

```java
@Component
@Order(1)
public class CacheWarmupRunner implements ApplicationRunner {
    private final ProductRepository productRepository;

    public CacheWarmupRunner(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Override
    @Transactional // ✅ Fully working transaction proxy!
    public void run(ApplicationArguments args) {
        if (args.containsOption("skip-warmup")) {
            System.out.println("Warmup skipped via command-line argument.");
            return;
        }
        System.out.println("Warming up product cache...");
        productRepository.preloadTopProducts();
    }
}
```

---

#### Approach 4: `CommandLineRunner` (Raw String Array Arguments)
- **Scenario**: Simple startup scripts needing direct access to `String... args`.
- **Use-Case**: Microservice seed data scripts reading raw positional file arguments.
- **Benefits**: Identical lifecycle timing to `ApplicationRunner`, but with standard `main(String[] args)` compatibility.

```java
@Component
@Order(2)
public class SeedDataRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        System.out.println("CommandLineRunner received " + args.length + " raw args.");
    }
}
```

---

#### Approach 5: `ApplicationListener<ApplicationReadyEvent>`
- **Scenario**: Executing tasks strictly after the embedded web server (Tomcat/Jetty) has started and is actively listening on its port.
- **Use-Case**: Notifying an external service registry (Consul/Eureka) or sending a Slack message that the microservice is healthy and receiving HTTP traffic.
- **Benefits**: Guarantees the application is 100% ready to receive incoming network traffic.

```java
@Component
public class ServerStartupNotifier implements ApplicationListener<ApplicationReadyEvent> {
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.println("Tomcat is fully bound and accepting requests!");
    }
}
```

---

#### Approach 6: `SmartLifecycle` (Phased Startup & Graceful Shutdown)
- **Scenario**: Custom network servers, consumer listeners, or socket connections requiring fine-grained control over startup order and **graceful shutdown** sequence.
- **Use-Case**: Starting a custom Netty TCP telemetry server after the database is ready, and stopping the TCP server before closing database connections on SIGTERM.
- **Benefits**:
  - Defines `getPhase()` (lower numbers start first, higher numbers stop first).
  - Implements `stop(Runnable callback)` enabling asynchronous graceful shutdown with Kubernetes termination grace periods.

```java
@Component
public class CustomTelemetryServer implements SmartLifecycle {
    private boolean isRunning = false;

    @Override
    public void start() {
        System.out.println("Starting TCP Telemetry socket server...");
        this.isRunning = true;
    }

    @Override
    public void stop() {
        this.isRunning = false;
    }

    @Override
    public void stop(Runnable callback) {
        System.out.println("Flushing buffers and shutting down socket server gracefully...");
        this.isRunning = false;
        callback.run(); // Informs Spring that shutdown is complete
    }

    @Override
    public boolean isRunning() {
        return isRunning;
    }

    @Override
    public int getPhase() {
        return 1000; // Late startup, early shutdown
    }
}
```

---

### Startup Approaches Comparison Matrix

| Approach | Lifecycle Timing | AOP & `@Transactional` Active? | CLI Arguments Access | Graceful Shutdown Support |
|---|---|---|---|---|
| **`@PostConstruct`** | Bean Init (Step 4/5) | ❌ **No** (Proxies created in Step 6) | ❌ None | ❌ No |
| **`InitializingBean`** | Bean Init (Step 5) | ❌ **No** | ❌ None | ❌ No |
| **`ApplicationRunner`** | Full Context Ready | ✅ **Yes** | ✅ Parsed (`ApplicationArguments`) | ❌ No |
| **`CommandLineRunner`** | Full Context Ready | ✅ **Yes** | ✅ Raw (`String[]`) | ❌ No |
| **`ApplicationReadyEvent`** | Web Server Listening | ✅ **Yes** | ⚠️ Via event context | ❌ No |
| **`SmartLifecycle`** | Configurable Phase | ✅ **Yes** | ❌ None | 🌟 **Yes (Async callback)** |

---

### 5.4 Custom Init and Destroy Methods

```java
@Configuration(proxyBeanMethods = false)
public class ConnectionPoolConfiguration {

    @Bean(initMethod = "startPool", destroyMethod = "drainPool")
    public ThirdPartyPool thirdPartyPool() {
        return new ThirdPartyPool();
    }
}
```

---

## 6. Bean Scopes & The Prototype-in-Singleton Problem

### 6.1 Overview of All Scopes

| Scope | Instances Created | Lifecycle / Context |
|---|---|---|
| **`singleton`** (Default) | Exactly **1 instance** per Spring Container | Entire application lifecycle |
| **`prototype`** | **New instance** on every `getBean()` or injection | Created by container, but **destroy phase is NOT managed** |
| **`request`** | **1 instance** per incoming HTTP Request | Servlet `WebApplicationContext` only |
| **`session`** | **1 instance** per HTTP Session | Servlet `WebApplicationContext` only |
| **`application`** | **1 instance** per `ServletContext` | Web applications only |
| **`websocket`** | **1 instance** per WebSocket session lifecycle | WebSocket context only |

---

### 6.2 Singleton Scope & Thread-Safety

By default, all Spring beans are Singletons. Because multiple HTTP request threads concurrently execute methods on the same singleton instance, **Singletons must be completely stateless or strictly thread-safe**.

```java
// ❌ CRITICAL BUG: Mutable instance state in Singleton causes concurrent race conditions!
@Service
public class UnsafeOrderCounter {
    private int totalOrders = 0; // Shared across all concurrent HTTP threads!

    public void recordOrder() {
        totalOrders++; // Race condition: non-atomic read-modify-write!
    }
}

// ✅ THREAD-SAFE: Using atomic concurrency primitives or stateless method variables
@Service
public class SafeOrderCounter {
    private final AtomicInteger totalOrders = new AtomicInteger(0);

    public void recordOrder() {
        totalOrders.incrementAndGet(); // Thread-safe lock-free CAS
    }

    public int getCount() {
        return totalOrders.get();
    }
}
```

---

### 6.3 Prototype Scope

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class DataExportSession {
    private final UUID sessionId = UUID.randomUUID();
    // Stateful tracking per export task
}
```

> [!WARNING]
> **Spring does NOT execute `@PreDestroy` or `DisposableBean.destroy()` for prototype beans!** Once the container instantiates and injects a prototype bean, it discards its reference. If prototype beans hold unmanaged OS resources (file handles, database sockets), manual cleanup must be performed by the caller.

---

### 6.4 Prototype Bean Injected into Singleton: 4 Solutions

#### The Concurrency Trap
When a Singleton bean injects a Prototype bean via standard constructor injection, **the prototype bean is only injected ONCE at application startup**. The Singleton retains that single instance forever, completely breaking the intended prototype behavior and corrupting concurrent user data!

```java
// ❌ CATASTROPHIC BUG: Only ONE cart is created at startup and shared among ALL users!
@Service
public class CheckoutService {
    private final UserShoppingCart cart; // @Scope("prototype")

    public CheckoutService(UserShoppingCart cart) {
        this.cart = cart; // Injected ONCE at startup!
    }
}
```

---

#### Solution 1: `ObjectProvider<T>` Injection (Recommended Idiomatic Approach)
- **Scenario**: Standard modern Spring Boot services requiring fresh prototype beans on demand.
- **Benefits**: Clean, strongly typed, zero bytecode magic, compile-time verified.

```java
@Service
public class CheckoutService {
    private final ObjectProvider<UserShoppingCart> cartProvider;

    public CheckoutService(ObjectProvider<UserShoppingCart> cartProvider) {
        this.cartProvider = cartProvider;
    }

    public void processOrder(OrderRequest request) {
        // Generates a brand-new UserShoppingCart prototype instance on every method call!
        UserShoppingCart cart = cartProvider.getObject();
        cart.loadCustomerItems(request.customerId());
        cart.calculateTaxes();
    }
}
```

---

#### Solution 2: Method Injection via `@Lookup`
- **Scenario**: Maintaining pure domain models without injecting Spring-specific provider interfaces.
- **Benefits**: Spring overrides the abstract method with a CGLIB subclass that queries the container.

```java
@Service
public abstract class CheckoutService {

    @Lookup
    public abstract UserShoppingCart getShoppingCart();

    public void processOrder(OrderRequest request) {
        UserShoppingCart cart = getShoppingCart(); // Fresh prototype instance
        cart.loadCustomerItems(request.customerId());
    }
}
```

---

#### Solution 3: Scoped Target Class Proxy (`ScopedProxyMode.TARGET_CLASS`)
- **Scenario**: Request or Prototype scoped beans injected into controllers or services where callers expect a standard class reference.
- **Benefits**: The singleton holds a CGLIB proxy; calling methods on the proxy delegates dynamically to the thread's active instance.

```java
@Component
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserShoppingCart {
    // Calling methods on this class routes through a proxy to the current thread's target
}
```

---

#### Solution 4: Direct `ApplicationContext.getBean()` Lookup (Service Locator)
- **Scenario**: Dynamic lookups where bean name or runtime arguments determine instantiation.
- **Trade-off**: Violates Dependency Inversion by coupling business logic directly to `ApplicationContext`.

```java
@Service
public class CheckoutService implements ApplicationContextAware {
    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext context) {
        this.context = context;
    }

    public void processOrder() {
        UserShoppingCart cart = context.getBean(UserShoppingCart.class);
    }
}
```

---

### Prototype Resolution Comparison

| Strategy | Spring Coupling | Reflection / Bytecode Overhead | Testability | Production Recommendation |
|---|---|---|---|---|
| **`ObjectProvider<T>`** | Low (`ObjectProvider` interface) | Ultra-low | High (can pass lambda mock) | 🌟 **Top Recommendation** |
| **`@Lookup`** | None in method signature | CGLIB subclassing | Moderate (must mock abstract method) | 🌟 **Recommended for Domain** |
| **`ScopedProxyMode`** | None | CGLIB Proxy on every invocation | Moderate | 🌟 **Standard for Request/Session** |
| **`ApplicationContext`** | High (Service Locator anti-pattern) | Reflection | Low (requires mocking context) | 🚫 **Avoid in business logic** |

---

### 6.5 Request and Session Scopes

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestCorrelationContext {
    private final String correlationId = UUID.randomUUID().toString();

    public String getCorrelationId() {
        return correlationId;
    }
}
```

---

## 7. @SpringBootApplication Composition

`@SpringBootApplication` is a **composed meta-annotation** combining three foundation annotations:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration     // 1. Specialization of @Configuration for Boot
@EnableAutoConfiguration     // 2. Triggers auto-configuration imports
@ComponentScan               // 3. Scans current package and sub-packages
public @interface SpringBootApplication {
    Class<?>[] exclude() default {};
    String[] scanBasePackages() default {};
}
```

```java
// Excluding unwanted auto-configurations programmatically:
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 8. Auto-Configuration Mechanics

### 8.1 How Auto-Configuration Works Internally

```
1. SpringApplication.run() boots container
               │
               ▼
2. @EnableAutoConfiguration activates AutoConfigurationImportSelector
               │
               ▼
3. Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   (Loaded ~150+ candidate auto-configuration classes)
               │
               ▼
4. Conditional Evaluation Phase:
   ├── @ConditionalOnClass (Is RedisClient on classpath?)
   ├── @ConditionalOnMissingBean (Has developer already defined a RedisTemplate?)
   └── @ConditionalOnProperty (Is spring.data.redis.host configured?)
               │
               ▼
5. Matching beans registered into ApplicationContext
```

### 8.2 Writing a Production Custom Auto-Configuration

```java
package com.enterprise.autoconfigure;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(MetricsCollector.class)
@EnableConfigurationProperties(MetricsProperties.class)
public class EnterpriseMetricsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean // Backs off if user defines their own MetricsCollector bean
    @ConditionalOnProperty(prefix = "enterprise.metrics", name = "enabled", havingValue = "true", matchIfMissing = true)
    public MetricsCollector metricsCollector(MetricsProperties properties) {
        return new PrometheusMetricsCollector(properties.getEndpoint());
    }
}
```

Registration file (`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`):
```properties
com.enterprise.autoconfigure.EnterpriseMetricsAutoConfiguration
```

---

## 9. SpringApplication.run() Pipeline

```
SpringApplication.run(Application.class, args)
  ├── 1. StopWatch started (measures bootstrap latency)
  ├── 2. BootstrapContext initialized
  ├── 3. ApplicationStartingEvent published to registered listeners
  ├── 4. Environment prepared (System properties, env vars, application.yml loaded)
  ├── 5. ApplicationEnvironmentPreparedEvent published
  ├── 6. Banner printed
  ├── 7. ApplicationContext created (Servlet/Reactive/None)
  ├── 8. Context prepared (Initializers invoked, ApplicationPreparedEvent fired)
  ├── 9. Context Refreshed (AbstractApplicationContext.refresh()):
  │       ├── invokeBeanFactoryPostProcessors() (@Configuration classes processed)
  │       ├── registerBeanPostProcessors() (BPPs registered)
  │       ├── onRefresh() -> Embedded Tomcat server started
  │       └── finishBeanFactoryInitialization() -> All singletons eagerly instantiated
  ├── 10. ContextRefreshedEvent & ApplicationStartedEvent published
  ├── 11. ApplicationRunner and CommandLineRunner beans executed in @Order
  └── 12. ApplicationReadyEvent published (Application is 100% operational)
```

---

## 10. Environment Profiles Activation (5 Approaches)

Environment profiles segregate configuration properties and beans across deployment environments (`dev`, `test`, `staging`, `prod`).

---

### Approach 1: Command-Line Arguments (`--spring.profiles.active`)
- **Scenario**: Local developer terminal launches or Docker container entrypoints.
- **Command**:
  ```bash
  java -jar order-service.jar --spring.profiles.active=prod
  ```
- **Benefits**: Highest command-line override precedence.

---

### Approach 2: Operating System Environment Variables
- **Scenario**: Modern Kubernetes Pod specifications or 12-factor cloud deployment engines.
- **Command / Env**:
  ```bash
  export SPRING_PROFILES_ACTIVE=prod
  java -jar order-service.jar
  ```
- **Benefits**: Cloud-native, zero JAR repackaging.

---

### Approach 3: Multi-Document YAML & Profile Groups
- **Scenario**: Enterprise microservices grouping multi-tier profiles together.
- **File** (`application.yml`):
  ```yaml
  spring:
    profiles:
      group:
        production:
          - prod
          - cloud-aws
          - db-postgres
          - metrics-datadog
  ```
- **Benefits**: Activating `--spring.profiles.active=production` automatically activates all 4 member profiles.

---

### Approach 4: Programmatic Activation via `SpringApplicationBuilder`
- **Scenario**: Custom launcher suites, desktop embedded utilities, or pre-flight initialization scripts.
- **Code**:
  ```java
  public class CustomLauncher {
      public static void main(String[] args) {
          new SpringApplicationBuilder(Application.class)
                  .profiles("prod", "cloud")
                  .run(args);
      }
  }
  ```

---

### Approach 5: Test Profile Slicing (`@ActiveProfiles`)
- **Scenario**: Integration testing with dedicated test databases (H2/Testcontainers).
- **Code**:
  ```java
  @SpringBootTest
  @ActiveProfiles("test")
  class OrderServiceIntegrationTest {
      // Loads application-test.yml automatically
  }
  ```

---

### Profile Activation Comparison Matrix

| Approach | Precedence | Mutability | Production Best Practice |
|---|---|---|---|
| **CLI Arguments** | Highest (Level 1) | Command level | Excellent for standalone container overrides |
| **OS Environment Variable** | High (Level 10) | Host / Pod level | 🌟 **Top Recommendation for Kubernetes / ECS** |
| **Profile Groups (YAML)** | Medium | Configuration level | 🌟 **Best for bundling multi-profile stacks** |
| **Programmatic Builder** | Bootstrap level | Code level | Useful for specialized bootstrap runners |
| **`@ActiveProfiles`** | Test harness | Test class level | 🌟 **Mandatory standard for integration tests** |

---

## 11. Configuration & Property Injection (4 Approaches)

---

### Approach 1: `@Value` Annotation
- **Scenario**: Injecting individual scalar properties or evaluating SpEL expressions.
- **Benefits**: Quick for single values with default fallbacks.
- **Limitations**: No type safety, no hierarchical grouping, no JSR-380 validation.

```java
@Service
public class PaymentGatewayClient {
    @Value("${payment.stripe.api-key}")
    private String apiKey;

    @Value("${payment.stripe.timeout:5000}") // Default 5000ms if missing
    private int timeout;

    @Value("#{T(java.lang.Math).min(${payment.max-retries:3}, 5)}") // SpEL evaluation
    private int maxRetries;
}
```

---

### Approach 2: `@ConfigurationProperties` with Mutable JavaBeans
- **Scenario**: Hierarchical configuration trees with complex nested maps, lists, and JSR-380 validation.
- **Benefits**: Type-safe, relaxed binding, IDE autocomplete metadata.

```java
@Component
@ConfigurationProperties(prefix = "app.security")
@Validated
public class SecurityProperties {
    @NotBlank
    private String jwtSecret;

    @Min(300)
    private int sessionTimeoutSeconds = 3600;

    private List<String> allowedOrigins = new ArrayList<>();

    // Getters and Setters...
    public String getJwtSecret() { return jwtSecret; }
    public void setJwtSecret(String jwtSecret) { this.jwtSecret = jwtSecret; }
    public int getSessionTimeoutSeconds() { return sessionTimeoutSeconds; }
    public void setSessionTimeoutSeconds(int s) { this.sessionTimeoutSeconds = s; }
    public List<String> getAllowedOrigins() { return allowedOrigins; }
    public void setAllowedOrigins(List<String> origins) { this.allowedOrigins = origins; }
}
```

---

### Approach 3: `@ConfigurationProperties` with Immutable Java 17+ Records (Modern Standard)
- **Scenario**: Modern immutable configurations in Spring Boot 3.x.
- **Benefits**: Fully immutable, compile-time null safety, zero boilerplate getters/setters.

```java
package com.enterprise.config;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
        @NotBlank String gatewayUrl,
        @Min(1000) @Max(60000) int timeoutMs,
        boolean enableRetry,
        Credentials credentials
) {
    public record Credentials(@NotBlank String apiKey, @NotBlank String secretKey) {}
}
```

Activation:
```java
@SpringBootApplication
@ConfigurationPropertiesScan("com.enterprise.config")
public class Application { ... }
```

---

### Approach 4: Programmatic `Environment` Abstraction Access
- **Scenario**: Dynamic property lookups where the property key is constructed dynamically at runtime.

```java
@Service
public class DynamicFeatureResolver {
    private final Environment env;

    public DynamicFeatureResolver(Environment env) {
        this.env = env;
    }

    public boolean isFeatureEnabled(String tenantId) {
        return env.getProperty("feature.tenant." + tenantId + ".enabled", Boolean.class, false);
    }
}
```

---

### Comparison Matrix of Property Approaches

| Feature | `@Value` | `@ConfigurationProperties` (Bean) | `@ConfigurationProperties` (Record) | `Environment` |
|---|---|---|---|---|
| **Type Safety** | ❌ String coercion only | ✅ Fully type-safe | 🌟 **Fully type-safe + Immutable** | ⚠️ Generic `getProperty(Class)` |
| **Validation Support** | ❌ None | ✅ JSR-380 (`@Validated`) | 🌟 **JSR-380 (`@Validated`)** | ❌ None |
| **Relaxed Binding** | ⚠️ Partial | ✅ Full (kebab, camel, env) | ✅ Full | ❌ Exact key only |
| **IDE Autocomplete** | ❌ No | ✅ With `spring-boot-configuration-processor` | ✅ With processor | ❌ No |
| **Best For** | Single scalar values | Legacy mutable POJOs | 🌟 **Production standard for Boot 3.x** | Dynamic runtime property keys |

---

## 12. Spring Boot Starters & Dependency BOM

### 12.1 Core Starters Reference
- `spring-boot-starter-web`: Spring MVC, Jackson JSON, and Embedded Tomcat.
- `spring-boot-starter-data-jpa`: Hibernate, Spring Data JPA repositories, HikariCP connection pool.
- `spring-boot-starter-validation`: Jakarta Bean Validation / Hibernate Validator.
- `spring-boot-starter-actuator`: Health check probes, Prometheus metrics, thread dumps.
- `spring-boot-starter-test`: JUnit 5, AssertJ, Mockito, JsonPath, and Spring Test context runner.

### 12.2 Swapping Embedded Tomcat for Undertow
```xml
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

---

## 13. ⚠️ High-Frequency Interview Traps & Gotchas

### Trap 1: Prototype Injected into Singleton
- **Problem**: Injected once at startup; loses prototype nature.
- **Fix**: Use `ObjectProvider<T>` or `@Lookup`.

### Trap 2: `@PostConstruct` and `@Transactional`
- **Problem**: Transactions fail silently or don't commit because CGLIB AOP proxies are created in Phase 6, *after* `@PostConstruct` in Phase 4.
- **Fix**: Use `ApplicationRunner` or `ApplicationListener<ApplicationReadyEvent>`.

### Trap 3: `@Autowired` on Static Fields
- **Problem**: Injected values remain `null` because Spring injects dependencies into object instances, never class objects.
- **Fix**: Use non-static constructor injection and instantiate utility services as managed Spring beans.

### Trap 4: `@Configuration(proxyBeanMethods = false)` (Lite Mode Pitfall)
- **Problem**: Calling another `@Bean` method directly produces a new unmanaged object instead of retrieving the container's singleton instance.
- **Fix**: Keep `proxyBeanMethods = true` if `@Bean` methods call each other, or inject dependencies as method arguments.

### Trap 5: Self-Invocation Bypasses AOP
- **Problem**: Calling a `@Transactional` or `@Async` method from another method in the *same class* bypasses the Spring AOP proxy (`this.method()`).
- **Fix**: Refactor to a separate collaborator service or inject self via `ObjectProvider<Self>`.

---

## 14. 🔍 Deep Dive — Internal Mechanics

### How `@Autowired` Disambiguation Algorithm Resolves Candidates:
1. Find all beans assignable to the requested injection target type.
2. If `0` matches -> throw `NoSuchBeanDefinitionException` (unless `required = false`).
3. If `1` match -> inject candidate directly.
4. If `2+` matches:
   - Check if exactly one candidate is marked `@Primary`. If yes -> inject it.
   - Check if `@Qualifier` matches a candidate's qualifier ID. If yes -> inject it.
   - Fall back to matching constructor/method parameter name against registered bean names.
   - If still ambiguous -> throw `NoUniqueBeanDefinitionException`.

---

## 15. 📝 Architecture Principle Checklist

- [ ] Use **constructor injection** with `final` fields for all mandatory dependencies.
- [ ] Use **`record` classes with `@ConfigurationProperties`** for configuration instead of loose `@Value` tags.
- [ ] Avoid state in Singleton beans; use `Atomic*` or `Concurrent*` primitives when shared mutable counters are required.
- [ ] Resolve Prototype-in-Singleton needs using `ObjectProvider<T>`.
- [ ] Execute transactional startup logic inside `ApplicationRunner`, never inside `@PostConstruct`.
- [ ] Structure event processing to use `@TransactionalEventListener(phase = AFTER_COMMIT)` for external side effects like Kafka and email.

---

## 16. ❓ Practice Questions & Answers

### Q1: Why does Spring Boot 3.x require Java 17 as an absolute baseline?
**Answer**: Spring Boot 3.x is built on Spring Framework 6.x and Jakarta EE 10. Jakarta EE 10 required the transition of all packages from `javax.*` to `jakarta.*`. Furthermore, Spring Framework 6 required modern JVM bytecode features including Java records, pattern matching, sealed classes, and pre-compilation Ahead-Of-Time (AOT) engine hooks used by GraalVM Native Image.

### Q2: How does `ObjectProvider<T>` prevent `NoSuchBeanDefinitionException`?
**Answer**: `ObjectProvider<T>` offers defensive retrieval methods such as `getIfAvailable()` and `getIfUnique()`. Instead of throwing a fatal startup exception if a collaborator is unconfigured, it returns `null` or a supplied default fallback, allowing graceful degradation.

### Q3: Explain why `@Transactional` does not roll back on checked exceptions by default.
**Answer**: Spring's transaction management defaults to rolling back on unchecked exceptions (`RuntimeException` and `Error`), following EJB 3.0 conventions where checked exceptions represent expected, recoverable business outcomes. To roll back on checked exceptions, declare `@Transactional(rollbackFor = Exception.class)`.

---

[🏠 Course Index](./00-course-index.md) | **Module 01** | [Next: 02 — Annotations →](./02-annotations.md)
