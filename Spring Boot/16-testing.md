# 16 — Testing Spring Boot 3.x Applications: The Enterprise Guide

> **Module Goal:** Master modern enterprise testing in Spring Boot 3.x (Java 17/21). From unit testing domain logic without Spring overhead, to sliced web tests (`@WebMvcTest`), real container integration via Testcontainers and `@ServiceConnection`, mocking external HTTP boundaries with WireMock and `MockRestServiceServer`, testing asynchronous pipelines with Awaitility, and enforcing architectural integrity with ArchUnit.

---

## Table of Contents

1. [The Modern Testing Pyramid in Spring Boot 3.x](#1-the-modern-testing-pyramid-in-spring-boot-3x)
2. [Web Layer Testing Approaches](#2-web-layer-testing-approaches)
   - [Approach 1: Sliced Controller Testing via `@WebMvcTest` & `MockMvc`](#approach-1-sliced-controller-testing-via-webmvctest--mockmvc)
   - [Approach 2: Full-Stack Integration Testing via `@SpringBootTest(webEnvironment = RANDOM_PORT)`](#approach-2-full-stack-integration-testing-via-springboottestwebenvironment--random_port)
   - [Approach 3: Non-Blocking Reactive Web Testing via `WebTestClient`](#approach-3-non-blocking-reactive-web-testing-via-webtestclient)
   - [Approach 4: Pure Isolated Unit Testing (`@ExtendWith(MockitoExtension.class)`)](#approach-4-pure-isolated-unit-testing-extendwithmockitoextensionclass)
3. [Database & Persistence Tier Testing Approaches](#3-database--persistence-tier-testing-approaches)
   - [Approach 1: Real Container Testing via Testcontainers & Spring Boot 3.1+ `@ServiceConnection`](#approach-1-real-container-testing-via-testcontainers--spring-boot-31-serviceconnection)
   - [Approach 2: Sliced Persistence Testing via `@DataJpaTest`](#approach-2-sliced-persistence-testing-via-datajpatest)
   - [Approach 3: Dynamic Property Injection via `@DynamicPropertySource`](#approach-3-dynamic-property-injection-via-dynamicpropertysource)
   - [Approach 4: Declarative Dataset Execution via `@Sql` & `@SqlMergeMode`](#approach-4-declarative-dataset-execution-via-sql--sqlmergemode)
4. [External HTTP & Downstream Service Mocking Strategies](#4-external-http--downstream-service-mocking-strategies)
   - [Approach 1: Network-Level Mocking via WireMock (`@WireMockTest`)](#approach-1-network-level-mocking-via-wiremock-wiremocktest)
   - [Approach 2: Client-Level Mocking via `@RestClientTest` & `MockRestServiceServer`](#approach-2-client-level-mocking-via-restclienttest--mockrestserviceserver)
   - [Approach 3: Consumer-Driven Contract Testing via Spring Cloud Contract](#approach-3-consumer-driven-contract-testing-via-spring-cloud-contract)
   - [Approach 4: Context-Level Mocking via `@MockBean` & `@SpyBean`](#approach-4-context-level-mocking-via-mockbean--spybean)
5. [Asynchronous, Event-Driven & Reactive Testing Strategies](#5-asynchronous-event-driven--reactive-testing-strategies)
   - [Approach 1: Polling-Based Asynchronous Assertion via Awaitility](#approach-1-polling-based-asynchronous-assertion-via-awaitility)
   - [Approach 2: Synchronous Test Executor Replacement (`SyncTaskExecutor`)](#approach-2-synchronous-test-executor-replacement-synctaskexecutor)
   - [Approach 3: Reactive Stream Verification via `StepVerifier`](#approach-3-reactive-stream-verification-via-stepverifier)
   - [Approach 4: Virtual Time Manipulation for Delays & Backoff](#approach-4-virtual-time-manipulation-for-delays--backoff)
6. [Architectural Governance & Test Data Management](#6-architectural-governance--test-data-management)
   - [Approach 1: Architectural Constraint Enforcement via ArchUnit](#approach-1-architectural-constraint-enforcement-via-archunit)
   - [Approach 2: Test Data Builder & ObjectMother Patterns](#approach-2-test-data-builder--objectmother-patterns)
7. [Comprehensive Configuration Reference](#7-comprehensive-configuration-reference)
8. [⚠️ Interview Traps & Production Gotchas](#️-interview-traps--production-gotchas)
9. [📝 Architectural Comparison Matrix](#-architectural-comparison-matrix)
10. [❓ Practice Questions & Real-World Scenarios](#-practice-questions--real-world-scenarios)

---

## 1. The Modern Testing Pyramid in Spring Boot 3.x

A high-performing test suite follows the **Testing Pyramid**: fast, isolated unit tests at the base, focused slice integration tests in the middle, and minimal full-context tests at the top.

```
                     ▲
                    / \
                   /   \     E2E Tests (5%)
                  / E2E \    - @SpringBootTest(RANDOM_PORT)
                 /-------\   - Full context + real Docker containers
                /         \
               /Integration\ Integration Tests (25%)
              /  (Slices)   \ - @WebMvcTest, @DataJpaTest, @RestClientTest
             /---------------\ - Testcontainers, WireMock
            /                 \
           /     Unit Tests    \ Unit Tests (70%)
          /     (Pure Java)     \ - JUnit 5 + MockitoExtension
         /                       \ - Zero Spring Context, execution in milliseconds
        └─────────────────────────┘
```

---

## 2. Web Layer Testing Approaches

Testing REST APIs requires choosing between mocking the servlet container (`MockMvc`), starting an embedded server, or bypassing Spring completely.

---

### Approach 1: Sliced Controller Testing via `@WebMvcTest` & `MockMvc`

#### Scenario
Verifying HTTP status codes, JSON serialization/deserialization, request header parsing, URL parameter bindings, and validation rules (`@Valid`) on controller endpoints without loading the entire service and persistence tier.

#### Use-Case
Unit-testing the web layer in isolation. Fast execution while still exercising Spring MVC infrastructure (filters, converters, interceptors, `@ControllerAdvice`).

#### Benefits Over Others
- **Fast Startup:** Only loads controllers and web components (`@ControllerAdvice`, `Filter`); ignores `@Service` and `@Repository` beans.
- **Mocked Business Logic:** Forces clean isolation by requiring `@MockBean` for service dependencies.
- **In-Memory Dispatch:** No real HTTP socket overhead or port allocation.

#### Production-Grade Code Example

```java
package com.example.testing.web.sliced;

import com.example.testing.domain.UserDto;
import com.example.testing.domain.UserRegistrationRequest;
import com.example.testing.service.UserService;
import com.example.testing.web.UserController;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerSlicedMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private UserService userService; // Mocked: Service layer is not loaded in this slice!

    @Test
    @DisplayName("GET /api/v1/users/{id} - Returns 200 OK with User JSON")
    @WithMockUser(username = "admin", roles = {"USER"})
    void shouldReturnUserWhenExists() throws Exception {
        UserDto mockUser = new UserDto("USR-101", "alice@example.com", "Alice Smith");
        when(userService.getUserById("USR-101")).thenReturn(mockUser);

        mockMvc.perform(get("/api/v1/users/USR-101")
                .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.userId").value("USR-101"))
                .andExpect(jsonPath("$.email").value("alice@example.com"))
                .andExpect(jsonPath("$.fullName").value("Alice Smith"));
    }

    @Test
    @DisplayName("POST /api/v1/users - Rejects invalid payload with 400 Bad Request")
    @WithMockUser
    void shouldRejectInvalidRegistrationPayload() throws Exception {
        UserRegistrationRequest invalidRequest = new UserRegistrationRequest("", "invalid-email");

        mockMvc.perform(post("/api/v1/users")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidRequest)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.title").value("Bad Request"))
                .andExpect(jsonPath("$.status").value(400));
    }
}
```

---

### Approach 2: Full-Stack Integration Testing via `@SpringBootTest(webEnvironment = RANDOM_PORT)`

#### Scenario
End-to-end integration tests verifying the full path from the HTTP network interface through security filters, controllers, transactional services, to a real database.

#### Use-Case
High-confidence acceptance tests validating that all layers function together before promoting an artifact to staging or production.

#### Benefits Over Others
- **Real Network Socket:** Boots embedded Tomcat on a random available OS port (`RANDOM_PORT`), exercising real network I/O.
- **Validates Entire Pipeline:** Verifies custom security filters, Jackson deserializers, database transactions, and JPA relationships work together.

#### Production-Grade Code Example

```java
package com.example.testing.web.full;

import com.example.testing.domain.OrderCreationRequest;
import com.example.testing.domain.OrderResponse;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderFullStackIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("POST /api/v1/orders - End-to-end order processing over real HTTP")
    void shouldExecuteFullOrderLifecycle() {
        OrderCreationRequest request = new OrderCreationRequest("SKU-9921", 2, "CUST-404");

        ResponseEntity<OrderResponse> response = restTemplate
                .withBasicAuth("testuser", "password")
                .postForEntity("/api/v1/orders", request, OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().orderId()).startsWith("ORD-");
        assertThat(response.getBody().status()).isEqualTo("CONFIRMED");
    }
}
```

---

### Approach 3: Non-Blocking Reactive Web Testing via `WebTestClient`

#### Scenario
Spring WebFlux applications or standard Spring MVC endpoints tested using a modern, fluent client that supports non-blocking assertions and response body extraction.

#### Use-Case
Testing reactive endpoints returning `Mono<T>` or `Flux<T>` streams, or as a modern alternative to `MockMvc` for imperative controllers.

#### Benefits Over Others
- **Universal Client:** Works against reactive WebFlux pipelines, MockMvc bindings, or live HTTP servers over the network.
- **Fluent Assertions:** Built-in JSON path assertion methods (`expectBody().jsonPath(...)`).

#### Production-Grade Code Example

```java
package com.example.testing.web.reactive;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class ReactiveProductEndpointTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldStreamCatalogItems() {
        webTestClient.get()
                .uri("/api/v1/reactive/products")
                .exchange()
                .expectStatus().isOk()
                .expectHeader().contentType("application/json")
                .expectBodyList(ProductDto.class)
                .hasSize(5)
                .consumeWith(response -> {
                    var list = response.getResponseBody();
                    org.assertj.core.api.Assertions.assertThat(list.get(0).sku()).isNotBlank();
                });
    }

    record ProductDto(String sku, String title, double price) {}
}
```

---

### Approach 4: Pure Isolated Unit Testing (`@ExtendWith(MockitoExtension.class)`)

#### Scenario
Unit-testing business calculation algorithms, domain state machines, or service methods containing complex business logic.

#### Use-Case
The base of the testing pyramid. Tests run in 2 to 5 milliseconds without starting any Spring ApplicationContext.

#### Benefits Over Others
- **Sub-Second Suite Execution:** Hundreds of tests execute in a few seconds.
- **Pinpoint Error Identification:** A failure points directly to a bug in that specific class, rather than a Spring wiring or configuration issue.

#### Production-Grade Code Example

```java
package com.example.testing.unit;

import com.example.testing.domain.Order;
import com.example.testing.domain.TaxCalculationResult;
import com.example.testing.repository.TaxRateRepository;
import com.example.testing.service.TaxCalculationService;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class) // Zero Spring context! Pure Java unit test!
class TaxCalculationServiceUnitTest {

    @Mock
    private TaxRateRepository taxRateRepository;

    @InjectMocks
    private TaxCalculationService taxService;

    @Test
    @DisplayName("Computes sales tax correctly when regional rate exists")
    void shouldCalculateTaxAccurately() {
        when(taxRateRepository.findByStateCode("CA")).thenReturn(Optional.of(new BigDecimal("0.0725")));

        TaxCalculationResult result = taxService.calculateTax("CA", new BigDecimal("100.00"));

        assertThat(result.taxAmount()).isEqualByComparingTo(new BigDecimal("7.25"));
        assertThat(result.totalWithTax()).isEqualByComparingTo(new BigDecimal("107.25"));
        verify(taxRateRepository, times(1)).findByStateCode("CA");
    }

    @Test
    @DisplayName("Throws IllegalArgumentException for invalid jurisdiction")
    void shouldThrowWhenStateNotFound() {
        when(taxRateRepository.findByStateCode("UNKNOWN")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> taxService.calculateTax("UNKNOWN", new BigDecimal("100.00")))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Unsupported tax jurisdiction: UNKNOWN");
    }
}
```

---

## 3. Database & Persistence Tier Testing Approaches

Testing repositories against in-memory H2 often introduces false positives because H2 has different SQL dialects, locks, and constraints compared to production PostgreSQL/MySQL.

---

### Approach 1: Real Container Testing via Testcontainers & Spring Boot 3.1+ `@ServiceConnection`

#### Scenario
Testing Spring Data JPA repositories, native SQL queries, window functions, and database constraints against the **exact** database engine used in production.

#### Use-Case
The modern standard for integration testing in Spring Boot 3.1+. Replaces H2 with a real Dockerized database.

#### Benefits Over Others
- **Zero Dialect Discrepancies:** Tests execute against real PostgreSQL, MySQL, or Oracle engines.
- **Zero Manual Configuration:** `@ServiceConnection` dynamically wires JDBC URL, username, and password into Spring Boot's `DataSource` without `@DynamicPropertySource`.

#### Production-Grade Code Example

```java
package com.example.testing.persistence.testcontainers;

import com.example.testing.domain.CustomerEntity;
import com.example.testing.repository.CustomerRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // Do NOT replace with H2!
class CustomerRepositoryTestcontainersTest {

    // Spring Boot 3.1+ ServiceConnection wires this container to HikariCP automatically!
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private CustomerRepository customerRepository;

    @Test
    @DisplayName("Saves and queries customer using PostgreSQL native constraints")
    void shouldPersistAndRetrieveCustomer() {
        CustomerEntity customer = new CustomerEntity("CUST-901", "Acme Corp", "enterprise");
        customerRepository.save(customer);

        var retrieved = customerRepository.findById("CUST-901");

        assertThat(retrieved).isPresent();
        assertThat(retrieved.get().getCompanyName()).isEqualTo("Acme Corp");
    }
}
```

---

### Approach 2: Sliced Persistence Testing via `@DataJpaTest`

#### Scenario
Testing repository query methods, derived queries, and Spring Data Specifications where the test suite must execute rapidly and embedded H2 is acceptable for basic syntax validation.

#### Use-Case
Fast developer feedback loops when writing complex JPQL or Criteria queries.

#### Benefits Over Others
- **Automatic Rollback:** Every test method executes inside a transaction that rolls back automatically at the end of the method.
- **Focused Slice:** Starts only JPA-related beans; ignores web and messaging tiers.

#### Production-Grade Code Example

```java
package com.example.testing.persistence.sliced;

import com.example.testing.domain.InvoiceEntity;
import com.example.testing.repository.InvoiceRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
class InvoiceRepositorySlicedJpaTest {

    @Autowired
    private TestEntityManager entityManager; // Convenient helper for persisting test fixtures

    @Autowired
    private InvoiceRepository invoiceRepository;

    @Test
    void shouldFindUnpaidInvoicesOverThreshold() {
        // Given: Seed test data via TestEntityManager
        entityManager.persist(new InvoiceEntity("INV-1", new BigDecimal("500.00"), false));
        entityManager.persist(new InvoiceEntity("INV-2", new BigDecimal("1200.00"), false));
        entityManager.persist(new InvoiceEntity("INV-3", new BigDecimal("800.00"), true));
        entityManager.flush();

        // When
        List<InvoiceEntity> overdue = invoiceRepository.findUnpaidOverThreshold(new BigDecimal("1000.00"));

        // Then
        assertThat(overdue).hasSize(1);
        assertThat(overdue.get(0).getInvoiceNumber()).isEqualTo("INV-2");
    }
}
```

---

### Approach 3: Dynamic Property Injection via `@DynamicPropertySource`

#### Scenario
Connecting integration tests to third-party Testcontainers (Kafka, LocalStack S3, HashiCorp Vault) where Spring Boot does not have a dedicated `@ServiceConnection` adapter.

#### Use-Case
Extracting dynamically allocated ports and hostnames from running Docker containers and injecting them into Spring's `Environment` before the application context starts.

#### Production-Grade Code Example

```java
package com.example.testing.persistence.dynamicprops;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class CustomContainerIntegrationTest {

    @Container
    static GenericContainer<?> customStorage = new GenericContainer<>("minio/minio:RELEASE.2024-01-16T16-07-38Z")
            .withExposedPorts(9000)
            .withEnv("MINIO_ROOT_USER", "admin")
            .withEnv("MINIO_ROOT_PASSWORD", "password123")
            .withCommand("server /data");

    @DynamicPropertySource
    static void configureDynamicStorageProperties(DynamicPropertyRegistry registry) {
        // Dynamically wire the random ephemeral port into application properties
        registry.add("app.storage.s3.endpoint",
                () -> "http://" + customStorage.getHost() + ":" + customStorage.getMappedPort(9000));
        registry.add("app.storage.s3.access-key", () -> "admin");
        registry.add("app.storage.s3.secret-key", () -> "password123");
    }

    @Test
    void contextLoadsWithDynamicProperties() {
        // ApplicationContext boots with the dynamically configured MinIO endpoint!
    }
}
```

---

### Approach 4: Declarative Dataset Execution via `@Sql` & `@SqlMergeMode`

#### Scenario
Integration tests requiring complex baseline database state (50 relational tables with foreign keys and reference data) that would require 200 lines of Java builder code to set up.

#### Use-Case
Loading SQL seed files before test methods and cleaning up data afterward.

#### Benefits Over Others
- **Separation of Test Data:** Keeps large database fixtures in SQL files rather than cluttering Java test code.
- **Composable Datasets:** Combine class-level reference data with method-level test fixtures using `@SqlMergeMode(MERGE)`.

#### Production-Grade Code Example

```java
package com.example.testing.persistence.sql;

import com.example.testing.repository.LedgerRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.context.jdbc.SqlMergeMode;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.context.jdbc.Sql.ExecutionPhase.AFTER_TEST_METHOD;
import static org.springframework.test.context.jdbc.Sql.ExecutionPhase.BEFORE_TEST_METHOD;
import static org.springframework.test.context.jdbc.SqlMergeMode.MergeMode.MERGE;

@DataJpaTest
@Sql(scripts = "/db/schema-seed-reference-data.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/db/cleanup-ledger.sql", executionPhase = AFTER_TEST_METHOD)
@SqlMergeMode(MERGE)
class LedgerRepositorySqlTest {

    @Autowired
    private LedgerRepository ledgerRepository;

    @Test
    @Sql(scripts = "/db/overdue-invoices-fixture.sql") // Appends to class-level seed script
    void shouldCalculateTotalOutstandingBalance() {
        double balance = ledgerRepository.calculateTotalOutstanding("CORP-99");
        assertThat(balance).isGreaterThan(5000.00);
    }
}
```

---

## 4. External HTTP & Downstream Service Mocking Strategies

When an application calls external REST APIs (payment gateways, shipping carriers, notification services), testing must simulate external service responses, timeouts, and error payloads.

---

### Approach 1: Network-Level Mocking via WireMock (`@WireMockTest`)

#### Scenario
Testing an HTTP client (e.g., `RestClient`, `RestTemplate`, or OpenFeign) where tests must exercise real network sockets, HTTP status codes, headers, and timeout behaviors.

#### Use-Case
The industry standard for mocking downstream HTTP services. Runs a real HTTP server on an ephemeral port.

#### Benefits Over Others
- **Tests the Real HTTP Stack:** Validates custom request interceptors, authorization headers, error decoders, and connection timeouts.
- **Rich Verification:** Assert that exact HTTP calls, headers, and JSON bodies were transmitted over the network.

#### Production-Grade Code Example

```java
package com.example.testing.external.wiremock;

import com.example.testing.client.PaymentGatewayClient;
import com.example.testing.domain.PaymentResult;
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@WireMockTest(httpPort = 8089)
class WireMockPaymentGatewayIntegrationTest {

    @Autowired
    private PaymentGatewayClient paymentClient;

    @DynamicPropertySource
    static void configureWireMockEndpoint(DynamicPropertyRegistry registry) {
        registry.add("external.payment.url", () -> "http://localhost:8089");
    }

    @Test
    void shouldHandleSuccessfulPaymentSettlement() {
        // Stub external payment provider response
        stubFor(post(urlEqualTo("/v1/charge"))
                .withHeader("Authorization", containing("Bearer secret-api-token"))
                .withRequestBody(matchingJsonPath("$.amount", equalTo("150.00")))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            {"chargeId": "CHG-9821", "status": "SETTLED"}
                        """)));

        PaymentResult result = paymentClient.settlePayment("ORD-1", 150.00);

        assertThat(result.isSuccess()).isTrue();
        assertThat(result.chargeId()).isEqualTo("CHG-9821");

        // Verify the exact HTTP request was made
        verify(1, postRequestedFor(urlEqualTo("/v1/charge")));
    }
}
```

---

### Approach 2: Client-Level Mocking via `@RestClientTest` & `MockRestServiceServer`

#### Scenario
Testing a service that uses Spring's `RestClient` or `RestTemplate` without starting an external server.

#### Use-Case
Focused unit/slice testing of client wrapper classes, validating URL templating, header construction, and response deserialization.

#### Benefits Over Others
- **Fast In-Memory Interception:** `MockRestServiceServer` intercepts HTTP client calls directly in memory without socket overhead.
- **Focused Slice:** Loads only client beans and JSON converters; avoids starting web or database components.

#### Production-Grade Code Example

```java
package com.example.testing.external.restclienttest;

import com.example.testing.client.ShippingClient;
import com.example.testing.domain.ShippingQuote;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.client.RestClientTest;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.*;
import static org.springframework.test.web.client.response.MockRestResponseCreators.withSuccess;

@RestClientTest(ShippingClient.class)
class ShippingClientSliceTest {

    @Autowired
    private ShippingClient shippingClient;

    @Autowired
    private MockRestServiceServer mockServer;

    @Test
    void shouldFetchShippingQuote() {
        mockServer.expect(requestTo("https://api.shipping.com/v1/rates?weightKg=5.5"))
                .andExpect(method(HttpMethod.GET))
                .andRespond(withSuccess("""
                    {"cost": 24.50, "estimatedDays": 2}
                """, MediaType.APPLICATION_JSON));

        ShippingQuote quote = shippingClient.getRate(5.5);

        assertThat(quote.cost()).isEqualTo(24.50);
        assertThat(quote.estimatedDays()).isEqualTo(2);

        mockServer.verify();
    }
}
```

---

### Approach 3: Consumer-Driven Contract Testing via Spring Cloud Contract

#### Scenario
Microservice architectures where changes to a provider service's API contract must not break downstream consumer services.

#### Use-Case
Validating contracts between consumer and provider teams to catch breaking API changes at build time.

#### Benefits Over Others
- **Contract as Source of Truth:** A machine-readable contract generates both producer-side acceptance tests and consumer-side WireMock stubs automatically.

---

### Approach 4: Context-Level Mocking via `@MockBean` & `@SpyBean`

#### Scenario
Replacing a specific internal Spring bean (e.g., `EmailNotificationService`) with a Mockito mock inside an integration test.

#### Use-Case
Mocking out components with external side effects (sending real emails, charging credit cards) during integration tests.

#### Trade-offs & Limitations
- **Context Pollution:** Adding `@MockBean` alters the `ApplicationContext` cache key, forcing Spring to create a new context and slowing down test suites.
- **Mitigation:** Define shared `@MockBean` dependencies in a common abstract base class to preserve context caching.

---

## 5. Asynchronous, Event-Driven & Reactive Testing Strategies

Testing asynchronous processes (`@Async`, scheduled tasks, Kafka listeners, reactive streams) requires waiting for completion without brittle `Thread.sleep()` calls.

---

### Approach 1: Polling-Based Asynchronous Assertion via Awaitility

#### Scenario
Testing methods with asynchronous side effects (e.g., publishing a Kafka event that updates a database row in the background).

#### Use-Case
Waiting for eventual consistency conditions without fixed delays.

#### Benefits Over Others
- **Zero Brittle `Thread.sleep()`:** Polls with a configurable interval and completes as soon as the condition is satisfied.
- **Clear Timeouts:** Throws informative exceptions if the condition is not met within the timeout window.

#### Production-Grade Code Example

```java
package com.example.testing.async.awaitility;

import com.example.testing.domain.AuditEvent;
import com.example.testing.repository.AuditRepository;
import com.example.testing.service.AsyncAuditingService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@SpringBootTest
class AsyncAuditingIntegrationTest {

    @Autowired
    private AsyncAuditingService auditingService;

    @Autowired
    private AuditRepository auditRepository;

    @Test
    void shouldPersistAuditRecordAsynchronously() {
        auditingService.recordLoginAsync("USR-9021", "SUCCESS");

        // Poll every 100ms for up to 3 seconds until the database contains the audit row
        await().atMost(Duration.ofSeconds(3))
                .pollInterval(Duration.ofMillis(100))
                .untilAsserted(() -> {
                    var events = auditRepository.findByUserId("USR-9021");
                    assertThat(events).hasSize(1);
                    assertThat(events.get(0).getStatus()).isEqualTo("SUCCESS");
                });
    }
}
```

---

### Approach 2: Synchronous Test Executor Replacement (`SyncTaskExecutor`)

#### Scenario
Testing classes with `@Async` methods where running the task synchronously on the calling thread simplifies assertions and eliminates concurrency concerns.

#### Use-Case
Unit testing services containing `@Async` methods without multi-threading overhead.

#### Benefits Over Others
- **Deterministic Execution:** The task completes before the method returns, allowing immediate assertions without polling.

#### Production-Grade Code Example

```java
package com.example.testing.async.syncexecutor;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.core.task.SyncTaskExecutor;
import org.springframework.scheduling.annotation.EnableAsync;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class SynchronousExecutorReplacementTest {

    @TestConfiguration
    @EnableAsync
    static class TestExecutorOverrideConfig {
        // Replaces the async thread pool with an executor that runs tasks on the caller thread
        @Bean("taskExecutor")
        public Executor taskExecutor() {
            return new SyncTaskExecutor();
        }
    }

    @Autowired
    private AsyncReportService reportService;

    @Test
    void shouldExecuteReportImmediatelyOnCurrentThread() throws Exception {
        CompletableFuture<String> future = reportService.generateReportAsync("Q4-SUMMARY");

        // With SyncTaskExecutor, the future is already completed upon method return!
        assertThat(future.isDone()).isTrue();
        assertThat(future.get()).isEqualTo("REPORT-DATA-Q4-SUMMARY");
    }
}

class AsyncReportService {
    @org.springframework.scheduling.annotation.Async("taskExecutor")
    public CompletableFuture<String> generateReportAsync(String name) {
        return CompletableFuture.completedFuture("REPORT-DATA-" + name);
    }
}
```

---

### Approach 3: Reactive Stream Verification via `StepVerifier`

#### Scenario
Testing Spring WebFlux or Project Reactor code returning `Mono<T>` or `Flux<T>` streams.

#### Use-Case
Asserting element sequences, completion signals, and error states emitted by reactive publishers.

#### Benefits Over Others
- **Deterministic Reactive Assertion:** Subscribes to publishers and asserts signals sequentially.

#### Production-Grade Code Example

```java
package com.example.testing.async.reactive;

import org.junit.jupiter.api.Test;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.test.StepVerifier;

class ReactiveStreamPipelineTest {

    @Test
    void shouldVerifyReactiveFluxStream() {
        Flux<String> stream = Flux.just("item1", "item2", "item3")
                .map(String::toUpperCase);

        StepVerifier.create(stream)
                .expectNext("ITEM1")
                .expectNext("ITEM2")
                .expectNext("ITEM3")
                .verifyComplete(); // Asserts stream completes without errors
    }

    @Test
    void shouldVerifyErrorSignal() {
        Mono<Object> failingMono = Mono.error(new IllegalStateException("Stream failed"));

        StepVerifier.create(failingMono)
                .expectError(IllegalStateException.class)
                .verify();
    }
}
```

---

### Approach 4: Virtual Time Manipulation for Delays & Backoff

#### Scenario
Testing reactive streams with long time delays (e.g., a periodic ticker emitting every 10 minutes, or exponential retry backoffs).

#### Use-Case
Fast-forwarding virtual time clocks using `StepVerifier.withVirtualTime()`, testing a 10-minute delay in 5 milliseconds.

#### Benefits Over Others
- **Zero Real-Time Delays:** Advances virtual clocks programmatically.

#### Production-Grade Code Example

```java
package com.example.testing.async.virtualtime;

import org.junit.jupiter.api.Test;
import reactor.core.publisher.Flux;
import reactor.test.StepVerifier;

import java.time.Duration;

class VirtualTimeManipulationTest {

    @Test
    void shouldFastForwardPeriodicIntervals() {
        // withVirtualTime takes a supplier to install the VirtualTimeScheduler before stream creation
        StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofMinutes(1)).take(3))
                .expectSubscription()
                .thenAwait(Duration.ofMinutes(1)) // Fast-forward 1 minute instantly!
                .expectNext(0L)
                .thenAwait(Duration.ofMinutes(2)) // Fast-forward 2 minutes instantly!
                .expectNext(1L, 2L)
                .verifyComplete();
    }
}
```

---

## 6. Architectural Governance & Test Data Management

Automated tests can also enforce architectural rules across team codebases and simplify test fixture creation.

---

### Approach 1: Architectural Constraint Enforcement via ArchUnit

#### Scenario
Enforcing architectural layering rules across large development teams to prevent architectural drift (e.g., ensuring controllers never inject repositories directly, and services never depend on web controllers).

#### Use-Case
Automated architectural compliance testing running as part of the standard JUnit build pipeline.

#### Benefits Over Others
- **Fails the Build on Architectural Violations:** Prevents forbidden dependencies before code merges.
- **Executable Architecture Documentation:** Expresses architecture rules as executable Java code.

#### Production-Grade Code Example

```java
package com.example.testing.architecture;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import org.springframework.stereotype.Service;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "com.example.testing")
class CleanArchitectureEnforcementTest {

    @ArchTest
    static final ArchRule enforceLayerBoundaries = layeredArchitecture()
            .consideringOnlyDependenciesInLayers()
            .layer("Web").definedBy("..web..")
            .layer("Service").definedBy("..service..")
            .layer("Repository").definedBy("..repository..")
            .whereLayer("Web").mayNotBeAccessedByAnyLayer()
            .whereLayer("Service").mayOnlyBeAccessedByLayers("Web")
            .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service");

    @ArchTest
    static final ArchRule controllersMustNotAccessRepositoriesDirectly = noClasses()
            .that().resideInAPackage("..web..")
            .should().dependOnClassesThat().resideInAPackage("..repository..")
            .because("Controllers must delegate to services and never access persistence directly");

    @ArchTest
    static final ArchRule serviceClassesMustBeAnnotatedWithService = classes()
            .that().haveSimpleNameEndingWith("Service")
            .should().beAnnotatedWith(Service.class);
}
```

---

### Approach 2: Test Data Builder & ObjectMother Patterns

#### Scenario
Constructing complex domain entity graphs in tests without repetitive constructor boilerplate.

#### Use-Case
Creating test fixtures with sensible defaults while allowing individual tests to override only the fields relevant to their scenario.

#### Benefits Over Others
- **Reduced Test Brittleness:** Adding a new field to an entity only requires updating the builder default; individual tests remain unchanged.

#### Production-Grade Code Example

```java
package com.example.testing.patterns;

import lombok.Builder;
import lombok.Getter;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Getter
@Builder
class TestOrder {
    private String orderId;
    private String customerId;
    private BigDecimal amount;
    private String status;
    private Instant createdAt;
}

// 1. Fluent Test Data Builder
class TestOrderBuilder {
    private String orderId = "ORD-" + UUID.randomUUID();
    private String customerId = "CUST-DEFAULT";
    private BigDecimal amount = new BigDecimal("99.99");
    private String status = "PENDING";
    private Instant createdAt = Instant.now();

    public static TestOrderBuilder anOrder() { return new TestOrderBuilder(); }
    public TestOrderBuilder withAmount(BigDecimal amount) { this.amount = amount; return this; }
    public TestOrderBuilder withStatus(String status) { this.status = status; return this; }
    public TestOrder build() {
        return new TestOrder(orderId, customerId, amount, status, createdAt);
    }
}

// 2. ObjectMother providing named, common test fixtures
class OrderMother {
    public static TestOrder standardPendingOrder() {
        return TestOrderBuilder.anOrder().build();
    }

    public static TestOrder paidVipOrder() {
        return TestOrderBuilder.anOrder()
                .withStatus("PAID")
                .withAmount(new BigDecimal("1500.00"))
                .build();
    }
}
```

---

## 7. Comprehensive Configuration Reference

```yaml
# src/test/resources/application-test.yml
spring:
  main:
    allow-bean-definition-overriding: true
  jpa:
    hibernate:
      ddl-auto: validate # Validate schemas against Flyway/Testcontainers
    open-in-view: false
  flyway:
    enabled: true
  task:
    scheduling:
      pool:
        size: 1
  cache:
    type: none # Disable caching in tests to prevent cross-test state leakage
```

---

## 8. ⚠️ Interview Traps & Production Gotchas

### Trap 1: Context Pollution Caused by Random `@MockBean` Usage

> [!CAUTION]
> Every unique combination of `@MockBean` across test classes modifies the Spring `TestContext` cache key. If 30 test classes each mock different services, Spring boots 30 separate application contexts, turning a 20-second test run into a 15-minute bottleneck.
> **Solution:** Group common `@MockBean` definitions into a shared abstract base test class.

---

### Trap 2: `@Transactional` in Tests Masks `LazyInitializationException`

Annotating integration test methods with `@Transactional` keeps the Hibernate session open for the entire duration of the test method. A call to a lazy-loaded relationship succeeds in the test, but fails in production because the real service transaction closes before the web serialization layer runs.

---

### Trap 3: Thread-Safety Issues with Shared WireMock Instances

When running tests in parallel, sharing a static WireMock port across test classes causes race conditions where one test's stubs override another's. Always use dynamic ephemeral ports or separate WireMock instances per test class.

---

## 9. 📝 Architectural Comparison Matrix

| Testing Technique | Startup Speed | Fidelity to Production | Scope | Best Used For |
|---|---|---|---|---|
| **JUnit + Mockito** | Instant (2ms) | Low | Unit | Pure domain and business logic |
| **`@WebMvcTest`** | Fast (500ms) | Medium | Web Slice | Routing, security, validation |
| **`@DataJpaTest`** | Moderate (1s) | Medium | Data Slice | Custom query method assertions |
| **Testcontainers** | Moderate (3-5s) | **Identical** | Persistence | True DB queries & constraints |
| **`@SpringBootTest`** | Slow (5-10s) | High | Full Stack | Critical end-to-end integration flows |

---

## 10. ❓ Practice Questions & Real-World Scenarios

1. **How does `@ServiceConnection` in Spring Boot 3.1+ eliminate the need for `@DynamicPropertySource`?**
   - *Answer:* `@ServiceConnection` inspects the running Testcontainer type (e.g., `PostgreSQLContainer`) and registers the corresponding `ConnectionDetails` bean (e.g., `JdbcConnectionDetails`) into the application context, auto-configuring connection properties without manual property mapping.

2. **Why can `@Transactional` on test methods produce false positive test results?**
   - *Answer:* It extends the persistence session across the entire test method. Lazily loaded entity relationships that would throw `LazyInitializationException` in production (where the transaction ends at the service boundary) succeed in the test because the session remains open.

3. **How does ArchUnit help teams maintain architectural integrity over time?**
   - *Answer:* It parses compiled bytecode and evaluates user-defined rules (e.g., preventing web controllers from accessing repositories directly), failing the build if developers introduce forbidden architectural dependencies.
