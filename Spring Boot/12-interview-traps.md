# 12 — Spring Boot Interview Traps & Deep Architectural Gotchas

> **Module Goal:** Master the treacherous edge cases, subtle concurrency traps, AOP proxy bypasses, transactional rollbacks, and bean scope misconfigurations in Spring Boot 3.x (Java 17/21). For every classic architectural failure, this guide provides the exact root cause, broken anti-pattern code, and all viable production-grade resolution approaches structured with **Scenario**, **Use-Case**, **Benefits Over Others**, and **Fully Documented Code Examples**.

---

## Table of Contents

1. [AOP Proxy Architecture & Self-Invocation Traps](#1-aop-proxy-architecture--self-invocation-traps)
   - [The Trap: Self-Invocation Bypassing `@Transactional`, `@Async`, and `@Cacheable`](#the-trap-self-invocation-bypassing-transactional-async-and-cacheable)
   - [Approach 1: Refactor to Dedicated Collaborator Service (Architectural Separation)](#approach-1-refactor-to-dedicated-collaborator-service-architectural-separation)
   - [Approach 2: Self-Injection via `@Lazy` / `ObjectProvider<T>`](#approach-2-self-injection-via-lazy--objectprovidert)
   - [Approach 3: Programmatic Execution (`TransactionTemplate`, `TaskExecutor`, `CacheManager`)](#approach-3-programmatic-execution-transactiontemplate-taskexecutor-cachemanager)
   - [Approach 4: `AopContext.currentProxy()` with Expose-Proxy Configuration](#approach-4-aopcontextcurrentproxy-with-expose-proxy-configuration)
   - [Approach 5: Compile-Time / Load-Time AspectJ Bytecode Weaving](#approach-5-compile-time--load-time-aspectj-bytecode-weaving)
2. [Bean Scope & Dependency Injection Traps](#2-bean-scope--dependency-injection-traps)
   - [The Trap: Prototype Scope State Mutation Inside Singleton Beans](#the-trap-prototype-scope-state-mutation-inside-singleton-beans)
   - [Approach 1: Method Injection via `@Lookup`](#approach-1-method-injection-via-lookup)
   - [Approach 2: Programmatic Bean Retrieval via `ObjectProvider<T>`](#approach-2-programmatic-bean-retrieval-via-objectprovidert)
   - [Approach 3: Target Class Scoped Proxies (`ScopedProxyMode.TARGET_CLASS`)](#approach-3-target-class-scoped-proxies-scopedproxymodetarget_class)
   - [Approach 4: Dedicated Factory Bean Pattern](#approach-4-dedicated-factory-bean-pattern)
3. [Transactional Boundary & Rollback Traps](#3-transactional-boundary--rollback-traps)
   - [The Trap: Checked Exceptions Committing Dirty State Silently](#the-trap-checked-exceptions-committing-dirty-state-silently)
   - [Approach 1: Explicit Declarative Rollback (`rollbackFor = Exception.class`)](#approach-1-explicit-declarative-rollback-rollbackfor--exceptionclass)
   - [Approach 2: Programmatic Rollback via `TransactionAspectSupport`](#approach-2-programmatic-rollback-via-transactionaspectsupport)
   - [Approach 3: Custom Domain Unchecked Exception Wrapping](#approach-3-custom-domain-unchecked-exception-wrapping)
   - [Approach 4: Programmatic `TransactionTemplate` with Custom Callback](#approach-4-programmatic-transactiontemplate-with-custom-callback)
4. [JPA & Persistence Context Traps](#4-jpa--persistence-context-traps)
   - [The Trap: `LazyInitializationException` Beyond the Session Boundary](#the-trap-lazyinitializationexception-beyond-the-session-boundary)
   - [Approach 1: JPQL Explicit `JOIN FETCH`](#approach-1-jpql-explicit-join-fetch)
   - [Approach 2: Dynamic JPA `@EntityGraph`](#approach-2-dynamic-jpa-entitygraph)
   - [Approach 3: DTO Record Constructor Projection (Zero Entity Overhead)](#approach-3-dto-record-constructor-projection-zero-entity-overhead)
   - [Approach 4: Programmatic `TransactionTemplate` Stack-Confined Loading](#approach-4-programmatic-transactiontemplate-stack-confined-loading)
5. [Concurrency, Async & ThreadContext Traps](#5-concurrency-async--threadcontext-traps)
   - [The Trap: `SimpleAsyncTaskExecutor` Memory Starvation & ThreadContext Loss](#the-trap-simpleasynctaskexecutor-memory-starvation--threadcontext-loss)
   - [Approach 1: Bounded `ThreadPoolTaskExecutor` with `TaskDecorator`](#approach-1-bounded-threadpooltaskexecutor-with-taskdecorator)
   - [Approach 2: Java 21 Virtual Threads (`VirtualThreadTaskExecutor`)](#approach-2-java-21-virtual-threads-virtualthreadtaskexecutor)
   - [Approach 3: Micrometer Context Propagation Pipeline](#approach-3-micrometer-context-propagation-pipeline)
6. [Web Layer & Servlet Filter Traps](#6-web-layer--servlet-filter-traps)
   - [The Trap: Servlet Filter Exceptions Bypassing `@RestControllerAdvice`](#the-trap-servlet-filter-exceptions-bypassing-restcontrolleradvice)
   - [Approach 1: Delegating to `HandlerExceptionResolver` Inside Filter](#approach-1-delegating-to-handlerexceptionresolver-inside-filter)
   - [Approach 2: Direct RFC 7807 `ProblemDetail` Writing via `ObjectMapper`](#approach-2-direct-rfc-7807-problemdetail-writing-via-objectmapper)
   - [Approach 3: Forwarding to Container Error Endpoint via `RequestDispatcher`](#approach-3-forwarding-to-container-error-endpoint-via-requestdispatcher)
7. [Comprehensive Trap Matrix & Senior Engineer Cheat Sheet](#7-comprehensive-trap-matrix--senior-engineer-cheat-sheet)

---

## 1. AOP Proxy Architecture & Self-Invocation Traps

### The Trap: Self-Invocation Bypassing `@Transactional`, `@Async`, and `@Cacheable`

#### The Failure Scenario
A senior developer creates an `OrderProcessingService`. The public entrypoint `processOrder()` executes preliminary validations and then invokes `executePaymentTransaction()` and `sendOrderConfirmationEmailAsync()` defined within the **same** class.

```java
// ❌ BROKEN CODE: Self-invocation completely bypasses Spring AOP proxies!
@Service
@Slf4j
public class OrderProcessingService {

    @Autowired
    private OrderRepository orderRepository;

    public void processOrder(Long orderId) {
        log.info("Processing order {}", orderId);
        // Direct method call on 'this' pointer!
        // The CGLIB proxy is NEVER invoked!
        this.executePaymentTransaction(orderId);
        this.sendOrderConfirmationEmailAsync(orderId);
    }

    @Transactional // ⚠️ TRAP: IGNORED! Runs in calling thread without transaction!
    public void executePaymentTransaction(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(OrderStatus.PAID);
        orderRepository.save(order);
        if (true) throw new RuntimeException("Payment gateway crashed!"); 
        // ⚠️ NEVER ROLLS BACK: The database row remains PAID!
    }

    @Async // ⚠️ TRAP: IGNORED! Runs synchronously on Tomcat HTTP request thread!
    public void sendOrderConfirmationEmailAsync(Long orderId) {
        // Blocks the HTTP response for 5 seconds while sending SMTP mail!
        sendEmailOverSmtp(orderId);
    }
}
```

#### Why It Fails Internally
Spring implements declarative behaviors using **Dynamic AOP Proxies** (CGLIB subclasses by default in Spring Boot 2.x and 3.x). When an external client calls `orderProcessingService.processOrder()`, it invokes the CGLIB proxy wrapper.

```
Caller (e.g. Controller)
       │
       ▼
┌──────────────────────────────────────┐
│       OrderProcessingService Proxy   │  <--- CGLIB Subclass
│  Intercepts processOrder()           │
└──────────────────┬───────────────────┘
                   │ Delegates
                   ▼
┌──────────────────────────────────────┐
│     Target Instance (Actual Object)  │
│  void processOrder() {               │
│      this.executePaymentTransaction()│  <--- DIRECT 'this' INVOCATION!
│  }                                   │       Proxy is completely bypassed!
│  void executePaymentTransaction() {} │       No TransactionInterceptor!
└──────────────────────────────────────┘
```

When `processOrder()` calls `this.executePaymentTransaction()`, it invokes the method directly on the internal JVM object reference `this`. It does **not** route through the CGLIB interceptor chain (`TransactionInterceptor`, `AsyncExecutionInterceptor`, `CacheInterceptor`).

---

### Approach 1: Refactor to Dedicated Collaborator Service (Architectural Separation)

#### Scenario
Complex domain services where business workflows encompass multiple transactional, asynchronous, or cached operations.

#### Use-Case
Recommended default design pattern adhering strictly to the **Single Responsibility Principle (SRP)**. Business workflows belong in an orchestrator, while transactional mutations belong in focused collaborator services.

#### Benefits Over Others
- **Zero Framework Magic:** Works universally without special proxy flags, reflection, or bytecode agents.
- **Superior Testability:** Collaborators can be cleanly mocked in unit tests without configuring Spring AOP test runners.
- **Clean Architecture:** Eliminates cyclic dependencies and bloated 1,000-line services.

#### Production-Grade Code Example

```java
package com.example.traps.aop.solution1;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderOrchestratorService {

    private final OrderPaymentService paymentService;
    private final OrderNotificationService notificationService;

    public void processOrder(Long orderId) {
        log.info("Orchestrating order processing for ID: {}", orderId);

        // Invokes CGLIB proxy of OrderPaymentService -> @Transactional is fully active!
        paymentService.executePaymentTransaction(orderId);

        // Invokes CGLIB proxy of OrderNotificationService -> @Async executes on thread pool!
        notificationService.sendOrderConfirmationEmailAsync(orderId);
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
class OrderPaymentService {

    @Transactional
    public void executePaymentTransaction(Long orderId) {
        log.info("Executing payment transaction inside active transaction. Thread: {}", Thread.currentThread().getName());
        // Database mutation code here...
    }
}

@Service
@Slf4j
class OrderNotificationService {

    @Async("taskExecutor")
    public void sendOrderConfirmationEmailAsync(Long orderId) {
        log.info("Dispatching email notification asynchronously on thread: {}", Thread.currentThread().getName());
        // SMTP dispatch here...
    }
}
```

---

### Approach 2: Self-Injection via `@Lazy` / `ObjectProvider<T>`

#### Scenario
Legacy services or monolithic domain classes where extracting dozens of methods into separate classes would create an explosion of tiny, artificial service classes.

#### Use-Case
Adding `@Async` or `@Transactional` caching to an internal utility method without undertaking a full architectural refactoring.

#### Benefits Over Others
- **Low Refactoring Overhead:** Requires modifying only the existing class without creating new files.
- **Guaranteed Proxy Invocation:** Route internal method calls through the injected self-proxy reference.
- **Cycle Prevention:** Modern Spring Boot prohibits circular dependencies by default; injecting with `@Lazy` or `ObjectProvider<T>` breaks the startup construction cycle cleanly.

#### Production-Grade Code Example

```java
package com.example.traps.aop.solution2;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.context.annotation.Lazy;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Slf4j
public class SelfInjectedReportService {

    // Option A: Field/Setter self-injection with @Lazy (Breaks circular reference check)
    private final ObjectProvider<SelfInjectedReportService> selfProvider;

    public SelfInjectedReportService(ObjectProvider<SelfInjectedReportService> selfProvider) {
        this.selfProvider = selfProvider;
    }

    public void generateMonthlyReport(String reportType) {
        log.info("Starting monthly report generation for type: {}", reportType);

        // Fetch the CGLIB proxy from the ObjectProvider
        SelfInjectedReportService proxy = selfProvider.getObject();

        // Calling proxy.fetchCachedData() invokes CacheInterceptor!
        String data = proxy.fetchCachedData(reportType);

        // Calling proxy.persistAuditAsync() submits to ThreadPoolTaskExecutor!
        proxy.persistAuditAsync(reportType, data.length());
    }

    @Cacheable(value = "report-data", key = "#reportType")
    public String fetchCachedData(String reportType) {
        log.info("Cache miss! Querying expensive database report data for: {}", reportType);
        return "Comprehensive financial data for " + reportType;
    }

    @Async("fileProcessingExecutor")
    public void persistAuditAsync(String reportType, int bytes) {
        log.info("Persisting audit log asynchronously on thread: {}", Thread.currentThread().getName());
    }
}
```

---

### Approach 3: Programmatic Execution (`TransactionTemplate`, `TaskExecutor`, `CacheManager`)

#### Scenario
High-throughput services requiring precise programmatic boundaries where declarative annotations lack granular control (e.g., catching exceptions inside a sub-transaction without marking the outer transaction rollback-only).

#### Use-Case
Complex transactional flows where only 3 lines of a 100-line method need database locks, or where caching must be conditionally checked without dynamic SpEL evaluations.

#### Benefits Over Others
- **Complete AOP Independence:** 100% immune to self-invocation proxy bypass; operates on pure object composition.
- **Fine-Grained Exception Handling:** Catch checked exceptions directly within transaction callbacks and return custom domain results.
- **Connection Efficiency:** Keeps the database connection open only for the 10 milliseconds of the database write, rather than the entire 5 seconds of the outer business method.

#### Production-Grade Code Example

```java
package com.example.traps.aop.solution3;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.task.TaskExecutor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.support.TransactionTemplate;

@Service
@RequiredArgsConstructor
@Slf4j
public class ProgrammaticOrderService {

    private final TransactionTemplate transactionTemplate;
    private final TaskExecutor taskExecutor;

    public void processOrderPipeline(Long orderId) {
        log.info("Initiating order pipeline for: {}", orderId);

        // 1. Non-transactional compute / validation (No DB connection held!)
        validateClientAccount(orderId);

        // 2. Programmatic Transaction Boundary (Zero AOP proxy dependency!)
        boolean paymentSuccessful = Boolean.TRUE.equals(transactionTemplate.execute(status -> {
            try {
                // DB mutations inside active transaction
                log.info("Executing transactional database write. Active: {}", status.hasSavepoint());
                persistPaymentRecord(orderId);
                return true;
            } catch (Exception e) {
                log.error("Payment failed, rolling back programmatic transaction", e);
                status.setRollbackOnly(); // Explicit programmatic rollback
                return false;
            }
        }));

        if (!paymentSuccessful) {
            throw new IllegalStateException("Order processing halted due to payment failure");
        }

        // 3. Programmatic Asynchronous Execution (Zero @Async proxy dependency!)
        taskExecutor.execute(() -> {
            log.info("Running background dispatch on worker thread: {}", Thread.currentThread().getName());
            dispatchWarehouseNotification(orderId);
        });
    }

    private void validateClientAccount(Long orderId) {}
    private void persistPaymentRecord(Long orderId) {}
    private void dispatchWarehouseNotification(Long orderId) {}
}
```

---

### Approach 4: `AopContext.currentProxy()` with Expose-Proxy Configuration

#### Scenario
Legacy monolithic codebases where refactoring classes or injecting collaborator objects is prohibited by project constraints.

#### Use-Case
Quickly fixing self-invocation in existing classes by explicitly retrieving the active proxy from the Spring `ThreadLocal` context.

#### Benefits Over Others
- **Zero Code Structural Changes:** No class refactoring or constructor injection required.
- **Preserves Existing Method Signatures:** Directly resolves the current executing proxy.

#### Trade-offs & Limitations
- **Couples Code to Spring AOP:** Pollutes domain code with framework-internal APIs (`AopContext`).
- **Requires Global Flag:** Throws `IllegalStateException: Cannot find current proxy` if `@EnableAspectJAutoProxy(exposeProxy = true)` is omitted.

#### Production-Grade Code Example

```java
package com.example.traps.aop.solution4;

import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.framework.AopContext;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Configuration
@EnableAspectJAutoProxy(exposeProxy = true) // ⚠️ CRITICAL: Exposes proxy to ThreadLocal
class AopExposureConfig {}

@Service
@Slf4j
public class ExposedProxyService {

    public void outerEntrypoint() {
        log.info("Executing outer entrypoint on thread: {}", Thread.currentThread().getName());

        // Retrieve the current Spring AOP proxy from ThreadLocal
        ExposedProxyService currentProxy = (ExposedProxyService) AopContext.currentProxy();

        // Invoke through the retrieved proxy wrapper -> Interceptors execute!
        currentProxy.transactionalInnerMethod();
    }

    @Transactional
    public void transactionalInnerMethod() {
        log.info("Executing transactional inner method with active interceptor!");
    }
}
```

---

### Approach 5: Compile-Time / Load-Time AspectJ Bytecode Weaving

#### Scenario
High-performance trading systems or deeply layered domain models where methods on `private`, `final`, or `package-private` scopes must participate in transactions, or where non-Spring-managed entities (instantiated via `new`) require dependency injection and transaction boundaries.

#### Use-Case
Complex enterprise systems discarding Spring's proxy model entirely in favor of true AspectJ bytecode weaving (CTW via `ajc` compiler plugin or LTW via JVM `-javaagent`).

#### Benefits Over Others
- **Interception of `this` Calls:** Bytecode is rewritten directly inside the `.class` file; internal `this.method()` calls are intercepted natively.
- **Works on `private` and `final` Methods:** Proxies can only override non-final public/protected methods; AspectJ weaves directly into bytecode instructions.
- **Superior Execution Performance:** Eliminates the method-call overhead of proxy interceptor chains.

#### Production-Grade Code Example

```xml
<!-- Maven pom.xml configuration for AspectJ Compile-Time Weaving -->
<plugin>
    <groupId>dev.aspectj</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>1.14</version>
    <configuration>
        <complianceLevel>21</complianceLevel>
        <source>21</source>
        <target>21</target>
        <aspectLibraries>
            <aspectLibrary>
                <groupId>org.springframework</groupId>
                <artifactId>spring-aspects</artifactId>
            </aspectLibrary>
        </aspectLibraries>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```java
package com.example.traps.aop.solution5;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableLoadTimeWeaving;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.stereotype.Service;

@Configuration
@EnableTransactionManagement(mode = org.springframework.context.annotation.AdviceMode.ASPECTJ)
class AspectJWeavingConfig {}

@Service
public class WeavedDomainService {

    public void process() {
        // With AspectJ weaving, this direct internal call IS intercepted!
        this.internalTransactionalWork();
    }

    @Transactional // Works EVEN on private methods with AspectJ!
    private void internalTransactionalWork() {
        // Database logic woven directly at bytecode level
    }
}
```

---

## 2. Bean Scope & Dependency Injection Traps

### The Trap: Prototype Scope State Mutation Inside Singleton Beans

#### The Failure Scenario
A developer defines a stateful user session validator as `@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)`. They inject this prototype bean directly into a singleton `@Service` via constructor injection.

```java
// ❌ THE SCOPE MISMATCH TRAP
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class UserSessionContext {
    private String userId; // Stateful user context
    public void setUserId(String userId) { this.userId = userId; }
    public String getUserId() { return userId; }
}

@Service
public class OrderService {
    // ⚠️ TRAP: The prototype is injected ONCE when OrderService is instantiated at startup!
    // Every subsequent request and concurrent thread shares this IDENTICAL instance!
    private final UserSessionContext sessionContext;

    public OrderService(UserSessionContext sessionContext) {
        this.sessionContext = sessionContext;
    }

    public void process(String currentUserId) {
        sessionContext.setUserId(currentUserId);
        // Under concurrent load, Thread B overwrites Thread A's userId, corrupting customer data!
    }
}
```

---

### Approach 1: Method Injection via `@Lookup`

#### Scenario
A singleton bean needs a fresh prototype bean instance on every method invocation without polluting the class with `ApplicationContext` dependencies.

#### Use-Case
Clean object-oriented code where Spring dynamically overrides an abstract or stub method using CGLIB bytecode generation.

#### Benefits Over Others
- **Clean Domain Code:** No references to Spring container interfaces (`BeanFactory`, `ApplicationContext`).
- **Framework Managed:** Spring generates a CGLIB subclass at runtime that queries the `BeanFactory` for a fresh prototype bean on each invocation.

#### Production-Grade Code Example

```java
package com.example.traps.di.lookup;

import org.springframework.beans.factory.annotation.Lookup;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class StatefulWorkerTask {
    private String correlationId;
    public void execute(String id) {
        this.correlationId = id;
        // Stateful task logic...
    }
}

@Service
public abstract class DispatcherService {

    public void dispatch(String customerId) {
        // Invoking getWorker() dynamically delegates to Spring's CGLIB BeanFactory lookup
        StatefulWorkerTask worker = getWorkerTask();
        worker.execute(customerId);
    }

    // Spring CGLIB overrides this method to return a brand new prototype instance every time!
    @Lookup
    public abstract StatefulWorkerTask getWorkerTask();
}
```

---

### Approach 2: Programmatic Bean Retrieval via `ObjectProvider<T>`

#### Scenario
Singleton services that need on-demand creation of prototype beans with optional construction parameters or streaming support.

#### Use-Case
The modern (Spring 4.3+) standard for programmatic, type-safe prototype resolution.

#### Benefits Over Others
- **Supports Concrete Classes:** Does not require abstract classes or methods.
- **Factory Parameterization:** Supports passing dynamic arguments to prototype constructors via `provider.getObject(arg1, arg2)`.

#### Production-Grade Code Example

```java
package com.example.traps.di.objectprovider;

import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class ProcessingContext {
    private final String tenantId;
    public ProcessingContext(String tenantId) { this.tenantId = tenantId; }
    public String getTenantId() { return tenantId; }
}

@Service
@RequiredArgsConstructor
public class MultiTenantJobCoordinator {

    // ObjectProvider retrieves a new instance on demand
    private final ObjectProvider<ProcessingContext> contextProvider;

    public void executeJob(String tenantId) {
        // Supplies dynamic constructor arguments to the prototype bean
        ProcessingContext context = contextProvider.getObject(tenantId);
        // Execute tenant-isolated work safely...
    }
}
```

---

### Approach 3: Target Class Scoped Proxies (`ScopedProxyMode.TARGET_CLASS`)

#### Scenario
Web applications injecting request-scoped or session-scoped beans (e.g., active user identity, shopping cart) directly into singleton controllers or services.

#### Use-Case
Transparent dependency injection where the singleton holds a stable CGLIB proxy reference, and the proxy dynamically routes method calls to the active thread's request or session context.

#### Benefits Over Others
- **Standard Direct Field Injection:** The singleton simply calls `userSession.getUserId()` as if it were a normal field.
- **Thread & Context Isolation:** The proxy queries `RequestContextHolder` underneath, ensuring each thread accesses its own scoped instance.

#### Production-Grade Code Example

```java
package com.example.traps.di.scopedproxy;

import lombok.Getter;
import lombok.Setter;
import org.springframework.context.annotation.Scope;
import org.springframework.context.annotation.ScopedProxyMode;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;
import org.springframework.web.context.WebApplicationContext;

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
@Getter
@Setter
class RequestUserContext {
    private String currentUserId;
    private String jwtToken;
}

@Service
public class BillingService {

    // Safe to inject into Singleton! Spring injects a CGLIB scoped proxy!
    private final RequestUserContext requestUserContext;

    public BillingService(RequestUserContext requestUserContext) {
        this.requestUserContext = requestUserContext;
    }

    public void processPayment() {
        // The proxy delegates to the current thread's HTTP request context
        String userId = requestUserContext.getCurrentUserId();
        // Execute billing for current user...
    }
}
```

---

### Approach 4: Dedicated Factory Bean Pattern

#### Scenario
Complex prototype initialization requiring database queries, dynamic builder construction, or external client authentication before returning the instance.

#### Use-Case
Complex domain objects whose creation involves complex lifecycle steps outside standard Spring XML or annotation scopes.

#### Benefits Over Others
- **Complete Encapsulation:** Isolates instantiation and resource configuration inside a dedicated factory component.
- **Deterministic Teardown:** Can manage destruction lifecycle callbacks explicitly.

#### Production-Grade Code Example

```java
package com.example.traps.di.factory;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

public class SftpConnectionClient {
    private final String sessionToken;
    public SftpConnectionClient(String sessionToken) { this.sessionToken = sessionToken; }
    public void upload(byte[] data) {}
}

@Component
public class SftpClientFactory {
    public SftpConnectionClient createNewClient(String userToken) {
        // Perform handshakes, allocate sockets
        return new SftpConnectionClient(userToken);
    }
}

@Component
@RequiredArgsConstructor
public class DataExportService {
    private final SftpClientFactory clientFactory;

    public void exportData(String userToken, byte[] payload) {
        SftpConnectionClient client = clientFactory.createNewClient(userToken);
        client.upload(payload);
    }
}
```

---

## 3. Transactional Boundary & Rollback Traps

### The Trap: Checked Exceptions Committing Dirty State Silently

#### The Failure Scenario
A developer marks a financial transfer method with `@Transactional`. During execution, an `IOException` or custom `PaymentDeclinedException` (extending `java.lang.Exception`) is thrown. 

```java
// ❌ THE CHECKED EXCEPTION TRAP
@Service
public class BankingService {

    @Autowired
    private AccountRepository accountRepository;

    @Transactional // ⚠️ TRAP: By default, Spring ONLY rolls back on RuntimeException and Error!
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) throws InsufficientFundsException {
        Account from = accountRepository.findById(fromId).orElseThrow();
        from.debit(amount);
        accountRepository.save(from);

        if (someExternalCheckFails()) {
            // Checked exception thrown!
            // ⚠️ SPRING COMMITS THE TRANSACTION! The debit is permanently written to the database!
            throw new InsufficientFundsException("Transfer blocked by compliance");
        }
    }
}
```

#### Why It Fails Internally
Spring's `TransactionAspectSupport` implements default rollback rules dating back to EJB container specifications:
- `RuntimeException` & `Error` -> **ROLLBACK**
- Checked `Exception` -> **COMMIT** (assumed to be a recoverable business outcome)

---

### Approach 1: Explicit Declarative Rollback (`rollbackFor = Exception.class`)

#### Scenario
Standard enterprise services using declarative annotations where all unhandled exceptions must trigger an immediate rollback.

#### Use-Case
Default production best practice for almost every enterprise `@Transactional` annotation.

#### Benefits Over Others
- **Universal Safety:** Ensures any thrown exception (checked or unchecked) marks the database transaction for immediate rollback.
- **Zero Boilerplate:** Single configuration property.

#### Production-Grade Code Example

```java
package com.example.traps.tx.solution1;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SecureBankingService {

    @Transactional(rollbackFor = Exception.class) // Ensures rollback for ALL checked and unchecked exceptions
    public void executeTransfer(Long fromId, Long toId, double amount) throws Exception {
        // Database mutations...
        if (amount > 10000) {
            throw new Exception("Compliance review required"); // Transaction ROLLS BACK cleanly!
        }
    }
}
```

---

### Approach 2: Programmatic Rollback via `TransactionAspectSupport`

#### Scenario
Methods that catch checked exceptions to log audit events, notify telemetry systems, or construct a custom response payload, while ensuring the database transaction is marked for rollback.

#### Use-Case
Suppressing exceptions from escaping the service layer while preventing dirty state from committing to the database.

#### Benefits Over Others
- **Safe Exception Absorption:** The method can catch the exception, return an HTTP error DTO, and still guarantee that no partial database writes are committed.

#### Production-Grade Code Example

```java
package com.example.traps.tx.solution2;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.interceptor.TransactionAspectSupport;

import java.io.IOException;

@Service
@Slf4j
public class SuppressedRollbackService {

    @Transactional
    public boolean performImport(byte[] data) {
        try {
            // Write batch records...
            readNetworkBatch(data);
            return true;
        } catch (IOException e) {
            log.error("Network IO failed during batch import, marking transaction rollback-only", e);
            // Explicitly notify the Spring transaction interceptor to rollback
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
            return false; // Return failure cleanly without bubbling exception to client
        }
    }

    private void readNetworkBatch(byte[] data) throws IOException {
        throw new IOException("Connection dropped");
    }
}
```

---

### Approach 3: Custom Domain Unchecked Exception Wrapping

#### Scenario
Modern Clean Architecture / Domain-Driven Design (DDD) where domain business exceptions extend `RuntimeException`.

#### Use-Case
Eliminating checked exceptions from domain service method signatures entirely, adhering to modern Java best practices.

#### Benefits Over Others
- **Clean Method Signatures:** No ugly `throws Exception` clauses polluting code.
- **Automatic Spring Rollback:** Natively triggers Spring's default rollback mechanism.

#### Production-Grade Code Example

```java
package com.example.traps.tx.solution3;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

// Domain exception extends RuntimeException
class InsufficientLiquidityException extends RuntimeException {
    public InsufficientLiquidityException(String message) { super(message); }
}

@Service
public class DomainTransferService {

    @Transactional // Rolls back automatically because InsufficientLiquidityException is unchecked!
    public void executeDebit(Long accountId, double amount) {
        if (amount > 500) {
            throw new InsufficientLiquidityException("Requested debit exceeds overdraft limits");
        }
    }
}
```

---

### Approach 4: Programmatic `TransactionTemplate` with Custom Callback

#### Scenario
Microservices needing fine-grained transaction boundaries without relying on method-level annotations.

#### Use-Case
Complex batch workflows where transactions are started, committed, or rolled back iteratively within loops.

#### Benefits Over Others
- **Loop-Level Transactions:** Commit or rollback individual records inside a loop without creating separate classes or methods.

#### Production-Grade Code Example

```java
package com.example.traps.tx.solution4;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.support.TransactionTemplate;

import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class BatchItemProcessor {

    private final TransactionTemplate transactionTemplate;

    public void processBatch(List<String> items) {
        for (String item : items) {
            // Each iteration executes inside its own independent, isolated physical transaction
            transactionTemplate.execute(status -> {
                try {
                    persistItem(item);
                    return true;
                } catch (Exception e) {
                    log.error("Failed item '{}', rolling back only this record", item, e);
                    status.setRollbackOnly();
                    return false;
                }
            });
        }
    }

    private void persistItem(String item) {}
}
```

---

## 4. JPA & Persistence Context Traps

### The Trap: `LazyInitializationException` Beyond the Session Boundary

#### The Failure Scenario
A controller calls a `@Transactional` service to load an `Author` entity. The service returns the entity, the transaction commits, and Hibernate closes the `EntityManager`. When Jackson serializes `author.getBooks()` in the web layer, Hibernate throws `LazyInitializationException: could not initialize proxy - no Session`.

```
Service Layer (Active Session)         Web / Controller Layer (Session Closed)
┌───────────────────────────────┐     ┌───────────────────────────────────────┐
│ @Transactional                │     │ Jackson ObjectMapper                  │
│ Author author = repo.get(1);  │ ──▶ │ author.getBooks()                     │
│ (Books collection is a proxy) │     │ ⚠️ LazyInitializationException:       │
│ Transaction commits & closes! │     │ "could not initialize proxy -         │
└───────────────────────────────┘     │  no Session"                          │
                                      └───────────────────────────────────────┘
```

> [!CAUTION]
> **Never enable `spring.jpa.open-in-view=true` (OSIV) as a workaround!** OSIV keeps database connections open during the entire HTTP request lifecycle, including view rendering and network I/O, quickly exhausting the HikariCP connection pool under moderate traffic.

---

### Approach 1: JPQL Explicit `JOIN FETCH`

#### Scenario
Known read workflows where the parent entity and its child collection are always required together.

#### Use-Case
Standard repository query methods loading parent and child entities in a single SQL query.

#### Benefits Over Others
- **Guaranteed Single Query:** Fetches both entities in a single `JOIN` query, eliminating the N+1 select problem.

#### Production-Grade Code Example

```java
package com.example.traps.jpa.joinfetch;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Set;

@Entity
@Table(name = "authors")
class Author {
    @Id @GeneratedValue private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private Set<Book> books;
}

@Entity
@Table(name = "books")
class Book {
    @Id @GeneratedValue private Long id;
    private String title;
    @ManyToOne(fetch = FetchType.LAZY) private Author author;
}

interface AuthorRepository extends JpaRepository<Author, Long> {

    // Eagerly fetches books in the initial SELECT query via SQL JOIN
    @Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books WHERE a.id = :id")
    Author findByIdWithBooksEagerly(@Param("id") Long id);
}
```

---

### Approach 2: Dynamic JPA `@EntityGraph`

#### Scenario
A repository method where different use cases require different fetch plans (e.g., Use Case A needs `Author.books`, while Use Case B needs `Author.books` and `Author.publisher`).

#### Use-Case
Declarative, dynamic fetching without rewriting queries.

#### Benefits Over Others
- **Composable Fetch Plans:** Keeps repository query definitions clean.
- **Cartesian Product Defense:** Can be configured to fetch subgraphs without hand-crafted SQL strings.

#### Production-Grade Code Example

```java
package com.example.traps.jpa.entitygraph;

import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

interface EntityGraphAuthorRepository extends JpaRepository<Author, Long> {

    // Dynamically overrides FetchType.LAZY for the 'books' attribute
    @EntityGraph(attributePaths = {"books"})
    Optional<Author> findDetailedById(Long id);
}
```

---

### Approach 3: DTO Record Constructor Projection (Zero Entity Overhead)

#### Scenario
Read-only queries serving REST APIs or dashboards where Hibernate entity dirty tracking and proxy management add unnecessary overhead.

#### Use-Case
High-performance read queries. This is the **most scalable, production-recommended solution**.

#### Benefits Over Others
- **Zero Hibernate Proxies:** Bypasses entities and proxies entirely; maps SQL columns directly to Java record constructors.
- **Zero `LazyInitializationException` Risk:** Plain Java records have no persistence session dependency.
- **Minimal Memory Footprint:** Bypasses Hibernate first-level cache dirty-checking snapshots.

#### Production-Grade Code Example

```java
package com.example.traps.jpa.dto;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public record AuthorSummaryDto(Long authorId, String authorName, Long totalBooks) {}

interface DtoAuthorRepository extends JpaRepository<Author, Long> {

    @Query("""
        SELECT new com.example.traps.jpa.dto.AuthorSummaryDto(a.id, a.name, COUNT(b.id))
        FROM Author a LEFT JOIN a.books b
        WHERE a.id = :id
        GROUP BY a.id, a.name
    """)
    AuthorSummaryDto findSummaryById(@Param("id") Long id);
}
```

---

### Approach 4: Programmatic `TransactionTemplate` Stack-Confined Loading

#### Scenario
Use cases where legacy service boundaries cannot be modified, but a specific controller or worker needs to load a lazy collection safely.

#### Use-Case
Initializing lazy collections within an explicit transaction boundary.

#### Benefits Over Others
- **Explicit Scope:** Isolates the database connection to the exact block of code that accesses the lazy relationship.

#### Production-Grade Code Example

```java
package com.example.traps.jpa.template;

import lombok.RequiredArgsConstructor;
import org.hibernate.Hibernate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.support.TransactionTemplate;

@Service
@RequiredArgsConstructor
public class LazyLoadingCoordinator {

    private final AuthorRepository authorRepository;
    private final TransactionTemplate transactionTemplate;

    public int getBookCountSafely(Long authorId) {
        return Boolean.TRUE.equals(transactionTemplate.execute(status -> {
            Author author = authorRepository.findById(authorId).orElseThrow();
            // Force initialization of lazy collection while the Hibernate session is active
            Hibernate.initialize(author.getBooks());
            return author.getBooks().size();
        }));
    }
}
```

---

## 5. Concurrency, Async & ThreadContext Traps

### The Trap: `SimpleAsyncTaskExecutor` Memory Starvation & ThreadContext Loss

#### The Failure Scenario
A developer adds `@EnableAsync` without defining a custom `TaskExecutor` bean. Under peak load, Spring defaults to `SimpleAsyncTaskExecutor`. This executor **does not pool threads**; it spawns an unbounded new OS thread for every single asynchronous task.

Furthermore, context stored in `ThreadLocal` (e.g., Spring Security's `SecurityContext`, Logback MDC trace IDs) is completely lost in the async thread.

```
Request Thread (Thread-1)                    Async Worker Thread (Thread-2)
┌──────────────────────────────────────┐     ┌──────────────────────────────────────┐
│ ThreadLocal Variables:               │     │ ThreadLocal Variables:               │
│ - SecurityContext: User="Alice"      │ ──X │ - SecurityContext: NULL (Anonymous!) │
│ - MDC: TraceId="TR-9821"             │     │ - MDC: NULL (Logs untraceable!)      │
└──────────────────────────────────────┘     └──────────────────────────────────────┘
```

---

### Approach 1: Bounded `ThreadPoolTaskExecutor` with `TaskDecorator`

#### Scenario
Production applications processing asynchronous tasks on platform worker threads while propagating MDC tracing IDs and tenant identifiers.

#### Use-Case
Standard production setup for `@Async` in Spring Boot 3.x.

#### Benefits Over Others
- **Thread Pool Protection:** Bounded pool size and queue capacity prevent thread exhaustion and out-of-memory errors.
- **Context Preservation:** Safely captures and restores `ThreadLocal` state across thread boundaries.

#### Production-Grade Code Example

```java
package com.example.traps.async.pool;

import org.slf4j.MDC;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskDecorator;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.Map;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

@Configuration
@EnableAsync
public class SafeAsyncPoolConfiguration implements AsyncConfigurer {

    @Bean(name = "applicationTaskExecutor")
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(32);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("app-async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // Propagate MDC logging context across thread boundaries
        executor.setTaskDecorator(new MdcContextPropagatingDecorator());
        executor.initialize();
        return executor;
    }
}

class MdcContextPropagatingDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        // Capture context from calling thread
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                // Restore context on worker thread
                if (contextMap != null) MDC.setContextMap(contextMap);
                runnable.run();
            } finally {
                // Always clean up worker thread state
                MDC.clear();
            }
        };
    }
}
```

---

### Approach 2: Java 21 Virtual Threads (`VirtualThreadTaskExecutor`)

#### Scenario
High-concurrency microservices performing high-volume I/O operations (HTTP requests, database calls, file transfers) in background tasks.

#### Use-Case
Modern Spring Boot 3.2+ applications running on Java 21+.

#### Benefits Over Others
- **Near-Infinite Concurrency:** Spawns millions of lightweight virtual threads with minimal memory overhead.
- **Zero Pool Tuning:** Eliminates complex thread pool sizing and queue capacity configuration.

#### Production-Grade Code Example

```java
package com.example.traps.async.virtual;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.web.embedded.tomcat.TomcatProtocolHandlerCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.AsyncTaskExecutor;
import org.springframework.core.task.support.TaskExecutorAdapter;

import java.util.concurrent.Executors;

@Configuration
public class VirtualThreadAsyncConfiguration {

    @Bean(name = "taskExecutor")
    public AsyncTaskExecutor virtualThreadTaskExecutor() {
        // Leverages Java 21 Virtual Threads for all @Async tasks
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }

    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadCustomizer() {
        // Leverages Virtual Threads for Tomcat HTTP connector requests
        return protocolHandler -> protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

---

### Approach 3: Micrometer Context Propagation Pipeline

#### Scenario
Distributed microservices utilizing OpenTelemetry, distributed tracing, and reactive pipelines.

#### Use-Case
The official Spring Boot 3.x mechanism for propagating observability and security context across threads.

#### Benefits Over Others
- **Unified Standard:** Propagates tracing, metrics, and security context uniformly across both imperative and reactive threads.

#### Production-Grade Code Example

```java
package com.example.traps.async.micrometer;

import io.micrometer.context.ContextRegistry;
import io.micrometer.context.ContextSnapshot;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.concurrent.Executor;

@Configuration
public class MicrometerContextConfiguration {

    @PostConstruct
    public void registerContextAccessors() {
        // Register SecurityContext propagation with Micrometer
        ContextRegistry.getInstance().registerThreadLocalAccessor(
                "security-context",
                SecurityContextHolder::getContext,
                SecurityContextHolder::setContext,
                SecurityContextHolder::clearContext
        );
    }

    public static Runnable wrapWithContext(Runnable task) {
        // Captures all registered ThreadLocals and restores them on execution
        return ContextSnapshot.captureAll().wrap(task);
    }
}
```

---

## 6. Web Layer & Servlet Filter Traps

### The Trap: Servlet Filter Exceptions Bypassing `@RestControllerAdvice`

#### The Failure Scenario
A developer builds an authentication filter (e.g., `JwtAuthenticationFilter`) that validates bearer tokens. When a malformed token is received, the filter throws an `InvalidTokenException`. 

The developer has a `@RestControllerAdvice` configured to handle `InvalidTokenException` and return a clean JSON error response. However, the client receives an ugly default Tomcat 500 HTML error page instead.

#### Why It Fails Internally
`@RestControllerAdvice` is integrated into Spring's `DispatcherServlet`. Servlet `Filter`s reside in the Servlet Container chain **outside and before** the `DispatcherServlet`. When an exception is thrown in a Filter, the `DispatcherServlet` is never reached.

```
Client HTTP Request
       │
       ▼
┌──────────────────────────────────────────────┐
│  Servlet Container Filter Chain               │
│  ┌────────────────────────────────────────┐  │
│  │ JwtAuthenticationFilter                │  │
│  │ ⚠️ Throws InvalidTokenException!       │  │
│  └──────────────────┬─────────────────────┘  │
└─────────────────────┼────────────────────────┘
                      │ ❌ Never reaches DispatcherServlet!
                      ▼
┌──────────────────────────────────────────────┐
│  Spring DispatcherServlet                    │
│  (Wraps @RestControllerAdvice)               │
└──────────────────────────────────────────────┘
```

---

### Approach 1: Delegating to `HandlerExceptionResolver` Inside Filter

#### Scenario
Propagating filter exceptions into the standard Spring MVC `@RestControllerAdvice` exception pipeline.

#### Use-Case
The cleanest, most idiomatic solution for handling filter errors in Spring Boot.

#### Benefits Over Others
- **Centralized Exception Handling:** Reuses your existing `@RestControllerAdvice` and `@ExceptionHandler` methods for both filter and controller errors.

#### Production-Grade Code Example

```java
package com.example.traps.filter.solution1;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.servlet.HandlerExceptionResolver;

import java.io.IOException;

@Component
@Slf4j
public class SafeJwtAuthenticationFilter extends OncePerRequestFilter {

    private final HandlerExceptionResolver resolver;

    // Inject Spring's primary HandlerExceptionResolver
    public SafeJwtAuthenticationFilter(@Qualifier("handlerExceptionResolver") HandlerExceptionResolver resolver) {
        this.resolver = resolver;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        try {
            validateBearerToken(request);
            filterChain.doFilter(request, response);
        } catch (Exception ex) {
            log.warn("Filter encountered error, delegating to DispatcherServlet HandlerExceptionResolver", ex);
            // Forward the exception into Spring's @RestControllerAdvice pipeline!
            resolver.resolveException(request, response, null, ex);
        }
    }

    private void validateBearerToken(HttpServletRequest request) {
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.equals("Bearer MALFORMED")) {
            throw new IllegalArgumentException("Invalid JWT signature");
        }
    }
}
```

---

### Approach 2: Direct RFC 7807 `ProblemDetail` Writing via `ObjectMapper`

#### Scenario
Lightweight security filters or edge gateways that need to reject invalid requests immediately without invoking the Spring MVC dispatcher pipeline.

#### Use-Case
High-security firewalls, rate limiters, and IP blocking filters.

#### Benefits Over Others
- **Zero MVC Dependency:** Generates the RFC 7807 JSON response directly from the filter.
- **Fail-Fast Efficiency:** Aborts request processing immediately at the container edge.

#### Production-Grade Code Example

```java
package com.example.traps.filter.solution2;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.FilterChain;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.net.URI;
import java.time.Instant;

@Component
public class DirectErrorWritingFilter extends OncePerRequestFilter {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws IOException {
        String apiKey = request.getHeader("X-API-KEY");

        if (apiKey == null || !apiKey.equals("SECRET-KEY")) {
            // Construct RFC 7807 Problem Detail response directly
            ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                    HttpStatus.UNAUTHORIZED,
                    "Missing or invalid API authorization key."
            );
            problem.setTitle("Unauthorized Access");
            problem.setType(URI.create("https://api.example.com/errors/unauthorized"));
            problem.setProperty("timestamp", Instant.now());

            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
            response.getWriter().write(objectMapper.writeValueAsString(problem));
            return; // Terminate filter chain execution
        }

        try {
            filterChain.doFilter(request, response);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

### Approach 3: Forwarding to Container Error Endpoint via `RequestDispatcher`

#### Scenario
Applications that already have a dedicated error controller mapping `/error` to handle container-level errors.

#### Use-Case
Reusing Spring Boot's built-in `BasicErrorController` for filter exceptions.

#### Benefits Over Others
- **Standard Servlet Compliance:** Relies entirely on the standard Java Servlet specification request dispatch mechanism.

#### Production-Grade Code Example

```java
package com.example.traps.filter.solution3;

import jakarta.servlet.FilterChain;
import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class RequestDispatcherForwardingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws IOException {
        try {
            filterChain.doFilter(request, response);
        } catch (Exception ex) {
            // Attach exception to standard request attributes
            request.setAttribute(RequestDispatcher.ERROR_STATUS_CODE, HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ex);
            request.setAttribute(RequestDispatcher.ERROR_MESSAGE, ex.getMessage());

            // Forward to Spring Boot's standard /error controller
            request.getRequestDispatcher("/error").forward(request, response);
        }
    }
}
```

---

## 7. Comprehensive Trap Matrix & Senior Engineer Cheat Sheet

| Category | The Trap | Root Cause | Primary Fix Approach | Alternative Fix Approach |
|---|---|---|---|---|
| **AOP** | Self-invocation ignores `@Transactional` | Calling `this.method()` bypasses CGLIB proxy wrapper | Separate into dedicated collaborator service | Self-injection via `@Lazy ObjectProvider<T>` |
| **AOP** | `@Async` runs synchronously on caller thread | Calling `this.asyncMethod()` bypasses AOP thread submission | Refactor method to external collaborator bean | Programmatic `TaskExecutor.execute()` |
| **DI** | Prototype bean behaves as a singleton | Injected once into singleton constructor at startup | Method injection via `@Lookup` | Programmatic retrieval via `ObjectProvider<T>` |
| **Tx** | Checked exceptions commit dirty state | Spring transactions only roll back on unchecked exceptions by default | Add `rollbackFor = Exception.class` | Wrap in domain `RuntimeException` |
| **JPA** | `LazyInitializationException` in web layer | Hibernate session closes at the service transaction boundary | DTO Record Constructor Projection | Repository `JOIN FETCH` / `@EntityGraph` |
| **Async** | Unbounded threads cause OutOfMemoryError | `@EnableAsync` defaults to `SimpleAsyncTaskExecutor` | Configure bounded `ThreadPoolTaskExecutor` | Java 21 Virtual Threads |
| **Web** | Filter exceptions return raw HTML 500 | Filters sit outside `DispatcherServlet` and `@RestControllerAdvice` | Delegate to `HandlerExceptionResolver` in filter | Write RFC 7807 `ProblemDetail` via `ObjectMapper` |
| **Sec** | Security context lost in async worker threads | SecurityContext is bound to caller's `ThreadLocal` | Configure `DelegatingSecurityContextAsyncTaskExecutor` | Micrometer `ContextSnapshot.wrap()` |
| **JPA** | Batch inserts disabled silently | `@GeneratedValue(strategy = GenerationType.IDENTITY)` forces immediate insert to get ID | Use `GenerationType.SEQUENCE` | Spring `JdbcClient.batch()` |
| **Web** | Memory exhaustion on large file uploads | Calling `file.getBytes()` loads entire binary into heap | Stream directly via NIO or `transferTo(Path)` | Asynchronous `StreamingResponseBody` |
