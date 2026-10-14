# Testing Spring Boot Applications: A Definitive Guide

Testing in Spring Boot is a massive topic. A robust testing strategy ensures that your application behaves exactly as expected in production, refactoring is safe, and regressions are caught before they reach users. Spring Boot provides a comprehensive testing framework that seamlessly integrates with JUnit, Mockito, Testcontainers, and other testing libraries.

In this deep dive, we will cover everything from the Testing Pyramid to advanced Testcontainers configurations, mocking strategies, and common interview traps that catch even senior engineers off guard.

---

## 1. The Testing Pyramid in Spring Boot

The Testing Pyramid is a conceptual framework that guides developers on how to structure their test suite. The fundamental idea is to have a large number of fast, isolated unit tests at the base, a moderate number of integration tests in the middle, and a few slow, brittle End-to-End (E2E) tests at the top.

### 1.1 Unit Tests
Unit tests verify the behavior of individual components (usually classes or methods) in isolation. In Spring Boot, unit tests **should not** start the Spring Application Context. They should use standard JUnit and Mockito.

- **Speed:** Extremely fast (milliseconds).
- **Scope:** Single class/method.
- **Dependencies:** All external dependencies (other classes, databases, APIs) are mocked.

### 1.2 Integration Tests
Integration tests verify that different components of your application work correctly together. In Spring, this often involves starting a partial or full Application Context to test how beans interact, how data is persisted to a database, or how the web layer handles HTTP requests.

- **Speed:** Slower (seconds).
- **Scope:** Multiple layers or external system interactions.
- **Dependencies:** Uses real databases (via Testcontainers) or wire-mocked external APIs.

### 1.3 End-to-End (E2E) Tests
E2E tests verify the entire application flow from the user's perspective, running the application in an environment that closely mirrors production.

- **Speed:** Very slow (minutes).
- **Scope:** The entire system, including UI (if applicable), backend, database, and message brokers.
- **Dependencies:** All real infrastructure.

### 1.4 When to Use Which

| Test Type | Spring Boot Approach | Tools | Execution Time | Context Started? |
| :--- | :--- | :--- | :--- | :--- |
| **Unit** | Plain JUnit5 + Mockito. No Spring annotations. | `@ExtendWith(MockitoExtension.class)` | 🚀 Milliseconds | ❌ No |
| **Integration** | Sliced Contexts or Full Context with Mocked Boundaries. | `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` | 🐢 Seconds | ✅ Yes (Full/Partial) |
| **E2E** | Full Context with real infrastructure (Testcontainers). | `@SpringBootTest(webEnvironment = RANDOM_PORT)`, RestAssured | 🐌 Minutes | ✅ Yes (Full) |

> [!IMPORTANT]
> The biggest mistake developers make is using `@SpringBootTest` for everything. This leads to massive test execution times and brittle tests. Use pure unit tests for domain logic and isolated component behavior.

---

## 2. Spring Boot Test Slices

Spring Boot provides "Test Slices"—specialized annotations that load only a subset of the Application Context. This is a crucial optimization technique. Instead of loading the entire application (which might include hundreds of beans, database connections, and external API clients), a slice loads only what is necessary for a specific layer.

### 2.1 The Almighty `@SpringBootTest`

`@SpringBootTest` is the heavy hitter. It tells Spring Boot to search for your main configuration class (the one with `@SpringBootApplication`) and start the **entire** Application Context.

#### When to use:
- When you genuinely need the entire application to be wired up.
- For broad integration tests where you test the flow from the Controller down to the Repository.

#### Why it's slow:
Starting a full Spring Context involves classpath scanning, auto-configuration evaluation, instantiating singletons, establishing database connection pools, and starting embedded servers (like Tomcat). Doing this repeatedly across hundreds of test classes is a performance nightmare.

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApplicationE2ETest {

    @Autowired
    private org.springframework.boot.test.web.client.TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrderSuccessfully() {
        // Given
        OrderRequest request = new OrderRequest("SKU-123", 5);

        // When
        var response = restTemplate.postForEntity("/api/v1/orders", request, OrderResponse.class);

        // Then
        assertThat(response.getStatusCode().is2xxSuccessful()).isTrue();
        assertThat(response.getBody().getOrderId()).isNotNull();
    }
}
```

> [!WARNING]
> **Context Caching:** Spring Test framework caches the Application Context based on the configuration of the test. If two test classes use the exact same setup (same profiles, same mocks, same configuration), Spring reuses the context. If you alter the context (e.g., using `@MockBean` in one test but not the other), Spring is forced to create a **new** context, drastically slowing down your test suite.

### 2.2 `@WebMvcTest`: Testing Controllers

`@WebMvcTest` auto-configures the Spring MVC infrastructure and limits scanned beans to `@Controller`, `@ControllerAdvice`, `@JsonComponent`, `Converter`, `GenericConverter`, `Filter`, `WebSecurityConfigurerAdapter`, and `HandlerMethodArgumentResolver`. 

It **does not** load `@Service`, `@Repository`, or `@Component` beans. You must mock your services using `@MockBean`.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class) // Only load UserController and MVC components
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc; // Entry point for MVC testing

    @MockBean
    private UserService userService; // We must mock the service layer!

    @Test
    void shouldReturnUserWhenFound() throws Exception {
        UserDto mockUser = new UserDto(1L, "saurabh", "saurabh@example.com");
        when(userService.getUserById(1L)).thenReturn(mockUser);

        mockMvc.perform(get("/api/users/1")
                .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.username").value("saurabh"))
                .andExpect(jsonPath("$.email").value("saurabh@example.com"));
    }
    
    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.getUserById(99L)).thenThrow(new UserNotFoundException(99L));

        mockMvc.perform(get("/api/users/99"))
                .andExpect(status().isNotFound()); // Relies on @ControllerAdvice
    }
}
```

#### Security with `@WebMvcTest`
If you have Spring Security on your classpath, `@WebMvcTest` will also scan security configuration. You often need to bypass security for simple controller tests, which you can do using `@WithMockUser`:

```java
@Test
@WithMockUser(username = "admin", roles = {"ADMIN"})
void shouldAllowAdminToDeleteUser() throws Exception {
    mockMvc.perform(delete("/api/users/1").with(csrf()))
            .andExpect(status().isNoContent());
}
```

### 2.3 `@DataJpaTest`: Testing Repositories

`@DataJpaTest` focuses entirely on JPA components. It configures an in-memory database (if available on the classpath like H2), configures Hibernate, Spring Data, and the `DataSource`, and enables SQL logging. It also turns on `@Transactional` by default, meaning every test rolls back its transactions at the end.

> [!TIP]
> By default, `@DataJpaTest` replaces your configured `DataSource` with an embedded database. If you want to use a real database (e.g., a PostgreSQL container via Testcontainers), you must explicitly disable this replacement using `@AutoConfigureTestDatabase(replace = Replace.NONE)`.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // Don't use H2! Use the DB provided by Testcontainers
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindUserByEmail() {
        // Given
        User user = new User("saurabh", "saurabh@example.com");
        userRepository.save(user);

        // When
        var foundUser = userRepository.findByEmail("saurabh@example.com");

        // Then
        assertThat(foundUser).isPresent();
        assertThat(foundUser.get().getUsername()).isEqualTo("saurabh");
    }
}
```

### 2.4 `@JsonTest`: Testing Serialization/Deserialization

Sometimes, you have complex JSON serialization rules (e.g., `@JsonFormat`, custom Serializers, `@JsonView`). `@JsonTest` auto-configures Jackson (or Gson) and provides `JacksonTester` utility classes to assert JSON structures.

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.boot.test.json.JacksonTester;
import java.time.LocalDate;
import static org.assertj.core.api.Assertions.assertThat;

@JsonTest
class EmployeeJsonTest {

    @Autowired
    private JacksonTester<Employee> json;

    @Test
    void serializeEmployee() throws Exception {
        Employee employee = new Employee(1L, "John Doe", LocalDate.of(2023, 1, 15));
        
        // Assert serialization logic
        var result = json.write(employee);
        
        assertThat(result).hasJsonPathNumberValue("$.id");
        assertThat(result).extractingJsonPathStringValue("$.name").isEqualTo("John Doe");
        // Verify custom date formatting
        assertThat(result).extractingJsonPathStringValue("$.joinedDate").isEqualTo("01-15-2023"); 
    }
    
    @Test
    void deserializeEmployee() throws Exception {
        String content = "{\"id\":1,\"name\":\"John Doe\",\"joinedDate\":\"01-15-2023\"}";
        
        Employee employee = json.parse(content).getObject();
        
        assertThat(employee.getName()).isEqualTo("John Doe");
        assertThat(employee.getJoinedDate()).isEqualTo(LocalDate.of(2023, 1, 15));
    }
}
```

---

## 3. Mocking & Stubbing

In Spring Boot, managing dependencies inside the Application Context during tests requires understanding the difference between standard Mockito mocks and Spring's Mockito integration.

### 3.1 `@MockBean` vs `@SpyBean` vs Mockito `@Mock`

| Annotation | Origin | Description | Context Effect | When to Use |
| :--- | :--- | :--- | :--- | :--- |
| `@Mock` | Mockito | Creates a standard mock object. Unaware of Spring. | None | Pure unit tests without Spring context. |
| `@MockBean` | Spring Boot | Adds a Mockito mock to the Spring Application Context, replacing any existing bean of the same type. | **Modifies Context** (Creates a new context cache key). | Integration tests where a specific downstream bean needs mocking. |
| `@SpyBean` | Spring Boot | Wraps an *existing* bean in the context with a Mockito spy. Real methods are called unless explicitly stubbed. | **Modifies Context** | When you want to verify calls on a real bean or stub only specific methods of a real bean. |

### 3.2 Context Pollution with `@MockBean`

> [!CAUTION]
> **Context Pollution:** Every time you use `@MockBean` or `@SpyBean` on a specific field in a test class, Spring Test computes a new context cache key. 
> 
> Example: 
> `TestA` uses `@SpringBootTest`. 
> `TestB` uses `@SpringBootTest` and `@MockBean EmailService`.
> 
> Spring cannot reuse `TestA`'s context for `TestB` because `TestB` requires a modified context (with a mocked `EmailService`). If you have 50 integration tests, and they mock random beans in different combinations, Spring will boot up the context 50 times. This is the #1 cause of extremely slow test suites.

**Solution:** 
1. Push `@MockBean` definitions to a common base class for integration tests.
2. Use `@TestConfiguration` to define global mocks.
3. Better yet, avoid `@MockBean` in broad integration tests and rely on WireMock or Testcontainers.

### 3.3 Advanced MockMvc

`MockMvc` provides a fluent API to perform HTTP requests against your controllers without starting a real HTTP server (like Tomcat).

```java
mockMvc.perform(post("/api/orders")
        .contentType(MediaType.APPLICATION_JSON)
        .content("{\"sku\":\"LAPTOP\",\"quantity\":2}")
        .header("X-API-KEY", "secret-key"))
        .andExpect(status().isCreated())
        .andExpect(header().string("Location", "http://localhost/api/orders/123"))
        .andExpect(jsonPath("$.orderId").value("123"))
        .andDo(print()); // Prints the request and response to console for debugging
```

### 3.4 JsonPath Magic

Spring MVC tests rely heavily on Jayway JsonPath for assertions.

- `$.id` - Access root field 'id'
- `$.items[0].name` - Access array elements
- `$.items.length()` - Array size
- `$.[?(@.price > 100)]` - Filter operations

```java
mockMvc.perform(get("/api/products"))
       .andExpect(jsonPath("$.content", hasSize(5))) // Requires org.hamcrest.Matchers.hasSize
       .andExpect(jsonPath("$.content[0].name").value("iPhone 15"))
       .andExpect(jsonPath("$.content[*].id", containsInAnyOrder(1, 2, 3, 4, 5)));
```

---

## 4. External Service Testing

Modern microservices rarely live in isolation; they talk to other HTTP APIs. Mocking the service layer that calls the API (using `@MockBean`) tests your application, but it *fails* to test the HTTP client mapping (RestTemplate, WebClient, or RestClient).

### 4.1 MockRestServiceServer (for RestTemplate/RestClient)

If you use Spring's native clients, `MockRestServiceServer` lets you mock the HTTP responses directly at the client level, verifying URL paths, headers, and payloads.

```java
import org.springframework.test.web.client.MockRestServiceServer;
import org.springframework.test.web.client.match.MockRestRequestMatchers;
import org.springframework.test.web.client.response.MockRestResponseCreators;
// ... setup in a @RestClientTest or @SpringBootTest

@Autowired
private RestTemplate restTemplate;

private MockRestServiceServer mockServer;

@BeforeEach
void setUp() {
    mockServer = MockRestServiceServer.createServer(restTemplate);
}

@Test
void shouldFetchWeatherFromExternalApi() {
    // Stub the external HTTP call
    mockServer.expect(MockRestRequestMatchers.requestTo("https://api.weather.com/v1/current?city=London"))
            .andExpect(MockRestRequestMatchers.method(HttpMethod.GET))
            .andRespond(MockRestResponseCreators.withSuccess("{\"temp\": 15.5, \"condition\": \"Rain\"}", MediaType.APPLICATION_JSON));

    // Call your service which uses the RestTemplate
    Weather result = weatherService.getWeatherForCity("London");

    assertThat(result.getTemperature()).isEqualTo(15.5);
    
    // Verify that the external API was actually called
    mockServer.verify();
}
```

### 4.2 WireMock

For broader integration testing, WireMock is the industry standard. It starts a lightweight, real HTTP server during tests. This tests your entire HTTP stack, including custom interceptors, error decoders, and timeouts.

**Setup with Spring Boot (using WireMock Spring Boot module):**

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.x.x</version>
    <scope>test</scope>
</dependency>
```

```java
import com.github.tomakehurst.wiremock.client.WireMock;
import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import org.junit.jupiter.api.Test;

@SpringBootTest
@WireMockTest(httpPort = 8089) // Starts WireMock on port 8089
class PaymentIntegrationTest {

    @Test
    void shouldProcessPaymentSuccessfully() {
        // Stub the external Stripe/PayPal API
        WireMock.stubFor(WireMock.post(WireMock.urlEqualTo("/v1/charges"))
                .withHeader("Authorization", WireMock.containing("Bearer "))
                .withRequestBody(WireMock.matchingJsonPath("$.amount", WireMock.equalTo("1000")))
                .willReturn(WireMock.aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("{\"id\": \"ch_123\", \"status\": \"succeeded\"}")));

        // Trigger your application logic...
        PaymentStatus status = paymentService.charge(1000, "token_abc");

        assertThat(status).isEqualTo(PaymentStatus.SUCCESS);
        
        // Verify exactly one POST request was made
        WireMock.verify(1, WireMock.postRequestedFor(WireMock.urlEqualTo("/v1/charges")));
    }
}
```
*Note: You would configure your application's external API base URL property in `application-test.yml` to point to `http://localhost:8089`.*

---

## 5. Testcontainers: The Modern Way to Integration Test

### 5.1 Why in-memory H2 is an anti-pattern for integration tests

Historically, developers used H2 or HSQLDB for `@DataJpaTest` and integration tests.
**The Problem:** H2 is not PostgreSQL, MySQL, or Oracle. 
- It has different syntax (e.g., JSONB queries in Postgres will fail in H2).
- It handles constraints differently.
- It has different transaction isolation semantics.
- You might write a query that works in H2 but fails in production, defeating the entire purpose of an integration test.

**The Solution:** Testcontainers. It uses Docker to spin up the *exact* same database version you use in production, runs your tests against it, and tears it down automatically.

### 5.2 Setting up Testcontainers for PostgreSQL

Add dependencies:
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### 5.3 `@ServiceConnection` (Spring Boot 3.1+ Magic)

Before Spring Boot 3.1, you had to use `@DynamicPropertySource` to manually extract the dynamic IP and port from the container and inject it into the Spring Environment.

**The Old Way (Spring Boot 2.x / 3.0):**
```java
@Testcontainers
@SpringBootTest
class OldWayTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    // ...
}
```

**The New Way (Spring Boot 3.1+):**
Spring Boot 3.1 introduced the `@ServiceConnection` annotation. It automatically discovers the container type (Postgres, Redis, Kafka, RabbitMQ) and creates the necessary `ConnectionDetails` beans to auto-configure the application!

```java
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class ModernIntegrationTest {

    @Container
    @ServiceConnection // Magic happens here! No DynamicPropertySource needed.
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @Test
    void contextLoads() {
        // The Spring application is automatically wired to the Postgres Docker container!
    }
}
```

### 5.4 Testcontainers with Redis

The exact same paradigm applies to Redis, Kafka, and others.

```java
@Container
@ServiceConnection
static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
```
With `@ServiceConnection`, Spring Boot automatically configures the `RedisConnectionFactory`.

---

## 6. Test Configuration & Best Practices

### 6.1 `@TestConfiguration`

Sometimes you need to define beans specifically for testing (e.g., a dummy implementation of an interface, or overriding a specific property). 

```java
@TestConfiguration
public class CustomTestConfig {
    
    @Bean
    @Primary // Override the primary bean in the context
    public S3Uploader mockS3Uploader() {
        return new DummyS3Uploader();
    }
}
```
To use this, import it into your test: `@Import(CustomTestConfig.class)`.

> [!NOTE]
> If a class is annotated with `@TestConfiguration`, Spring Boot's component scanning will **ignore** it in the main `src/main/java` code. It is designed to be used safely in `src/test/java`.

### 6.2 `@ActiveProfiles("test")`

Always use a specific profile for tests. This allows you to maintain an `application-test.yml` file where you can disable scheduled jobs, disable caching, or tweak retry mechanisms to speed up tests.

```yaml
# src/test/resources/application-test.yml
spring:
  flyway:
    enabled: true
  task:
    scheduling:
      pool:
        size: 1 # Reduce overhead
  cache:
    type: none # Disable caching to prevent state bleed between tests
```

Use `@ActiveProfiles("test")` on your test classes or base test class.

### 6.3 Resetting Database State Between Tests

In integration tests, mutating database state can cause subsequent tests to fail if they expect a clean slate. How do we clean the database?

1. **`@Transactional` on Test Methods (Default in `@DataJpaTest`):** 
   Spring wraps the test in a transaction and rolls it back at the end. 
   *Pros:* Extremely fast. 
   *Cons:* Can mask lazy-loading exceptions and transaction boundary issues (see Interview Traps).

2. **`@Sql` annotation:**
   Executes SQL scripts before or after a test.
   ```java
   @Test
   @Sql(scripts = "/test-data.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
   @Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
   void testWithData() { ... }
   ```

3. **Database Rider / Testcontainers JDBC `TC_INITSCRIPT`:**
   Declarative dataset management.

4. **Manual Teardown:**
   Injecting the `UserRepository` and calling `repository.deleteAll()` in an `@AfterEach` method. Safest, but slow if you have many tables with foreign key constraints.

---

## 7. ⚠️ Interview Traps & Gotchas

### 7.1 `@Transactional` in Tests Hiding `LazyInitializationException`

**The Trap:** You write an integration test. You fetch an entity, and assert on a lazily loaded collection. The test passes! You deploy to production. Production crashes with `org.hibernate.LazyInitializationException`.

**Why?**
When you put `@Transactional` on a JUnit test method, the *entire test method* runs inside a single transaction. The Hibernate Session remains open for the duration of the test. When you access a lazy collection, Hibernate happily fetches it.
In production, the transaction usually ends at the Service layer. When the Controller tries to access the lazy collection to serialize it to JSON, the Session is closed, throwing the exception.

**The Fix:**
Avoid `@Transactional` on integration tests that test end-to-end flows. Instead, clean up your data manually in `@AfterEach` or use a dedicated database clearing utility. If you must use `@Transactional` in tests, manually flush and clear the entity manager (`entityManager.flush(); entityManager.clear();`) immediately after the setup phase to detach entities and simulate the transaction boundary.

### 7.2 Context Caching and the `@DirtiesContext` Nightmare

**The Trap:** "My tests fail randomly depending on the execution order." 
Developer adds `@DirtiesContext` to the test class to "fix" it.

**Why it's bad:**
`@DirtiesContext` tells Spring: "This test modified the Application Context (e.g., changed a singleton bean's state). Destroy the context and rebuild it from scratch for the next test."
If you have 50 tests with `@DirtiesContext`, Spring starts 50 Application Contexts. A 10-second test suite becomes a 10-minute test suite.

**The Fix:**
Never use `@DirtiesContext` unless you are testing failure scenarios of context loading. 
Instead, find the stateful bean that is bleeding state (e.g., a `static` variable, a singleton cache, or data in the DB) and reset its state in an `@AfterEach` method.

### 7.3 `@WebMvcTest` Doesn't Load Everything

**The Trap:** You use `@WebMvcTest` and wonder why your `OncePerRequestFilter`, global CORS configuration (defined in a `@Configuration` class), or AOP aspects aren't running.

**Why?**
`@WebMvcTest` is a *slice*. It specifically excludes generic `@Component` and `@Configuration` classes unless they are strictly related to MVC infrastructure.
**The Fix:** You must explicitly import your filter or configuration using `@Import(MySecurityFilter.class)` on the test class.

### 7.4 Testing `@Async` Methods

**The Trap:** You call a method annotated with `@Async`. Your JUnit assertion immediately runs and fails because the async thread hasn't finished yet.

**The Fix:** Use Awaitility. Awaitility is a DSL for testing asynchronous systems.

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import static org.awaitility.Awaitility.await;
import java.time.Duration;

@Test
void shouldProcessAsyncEvent() {
    eventPublisher.publish(new UserRegisteredEvent(1L));

    // Wait at most 5 seconds until the repository indicates the user is active
    await().atMost(Duration.ofSeconds(5))
           .pollInterval(Duration.ofMillis(200))
           .untilAsserted(() -> {
               User user = userRepository.findById(1L).orElseThrow();
               assertThat(user.getStatus()).isEqualTo(UserStatus.ACTIVE);
           });
}
```

---

## 8. 🔍 Deep Dive: How `@MockBean` breaks Context Caching

Spring’s `TestContextManager` maintains a `ContextCache` (a static cache of ApplicationContexts). The key for this cache is formed by combining several factors of your test class:
1. `locations` (from `@ContextConfiguration`)
2. `classes` (from `@ContextConfiguration` / `@SpringBootTest`)
3. `contextInitializerClasses`
4. `contextCustomizers` (This is where `@MockBean` hooks in!)
5. `activeProfiles`

When you add `@MockBean` to a test class, Spring registers a `MockitoContextCustomizer`. This customizer alters the bean definition registry to replace the real bean definition with a mock factory. 

Because `TestClassA` has no customizers, and `TestClassB` has a `MockitoContextCustomizer` for `UserService`, they generate *different* cache keys. Spring detects a cache miss for `TestClassB` and starts a brand new context. 

To optimize:
Create a base class for all integration tests:
```java
@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
public abstract class BaseIntegrationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    
    // Define ANY widely used mock here so it becomes part of the shared context key
    @MockBean
    protected ThirdPartyBillingClient billingClient;
}
```
If all your tests extend `BaseIntegrationTest`, they will share the exact same configuration, customizers, and cache key. Spring will boot the context exactly **once**.

---

## 9. 📝 Summary & Key Takeaways

1. **Follow the Pyramid:** Write many isolated Unit Tests, fewer sliced/Testcontainer Integration Tests, and rare full E2E tests.
2. **Slicing is powerful:** Use `@WebMvcTest`, `@DataJpaTest`, and `@JsonTest` to test specific layers rapidly without booting the entire monolith.
3. **Beware Context Pollution:** `@MockBean` and `@SpyBean` generate new context cache keys. Push them to base classes if used broadly, or avoid them in favor of wire-mocking.
4. **Testcontainers > In-Memory DBs:** Never use H2 to test Postgres repositories. Use Testcontainers with Spring Boot 3.1's `@ServiceConnection` for effortless real-database testing.
5. **Mock the Network, Not the Client:** Use `WireMock` or `MockRestServiceServer` to test external HTTP integrations rather than mocking the HTTP client interface.
6. **Watch out for `@Transactional`:** It modifies the transaction boundaries of your application during tests and can mask critical bugs like `LazyInitializationException`.

---

## 10. ❓ Practice Questions

1. Explain why using `@SpringBootTest` for every test class in a large enterprise application is a bad practice.
2. You have a Controller that relies on a `JwtUtils` component for parsing tokens. When you write a `@WebMvcTest` for this controller, Spring throws a `NoSuchBeanDefinitionException` for `JwtUtils`. Why does this happen and how do you fix it?
3. Describe the mechanism by which `@MockBean` affects Spring's context caching.
4. What is the difference between `@Mock` and `@MockBean`?
5. Why is H2 considered an anti-pattern for integration testing a Spring Boot application configured for MySQL? What is the modern alternative?
6. You have an `@Async` method that updates a database record. Your test asserts the record is updated, but it randomly fails. How do you rewrite the test to be deterministic?
7. Explain how `@ServiceConnection` works in Spring Boot 3.1+ and how it simplifies Testcontainers setup compared to `@DynamicPropertySource`.
