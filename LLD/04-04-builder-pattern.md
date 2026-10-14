# Builder Pattern: Production Architecture & Low-Level Design in Java

> **Category:** [Creational Design Patterns](./04-creational-design-patterns.md) | **Navigation:** [Back to Creational Index](./04-creational-design-patterns.md) | [Master LLD Index](./00-index.md)

---


### 4.1 Intent & Problem Context
The **Builder Pattern** separates the construction of a complex object from its representation, allowing the same construction process to create various representations.

#### The Anti-Patterns Builder Resolves
1. **Telescoping Constructor Anti-Pattern**: Providing overloaded constructors with increasing parameter lists (`new Config(a)`, `new Config(a, b)`, `new Config(a, b, c, d, e)`). This leads to parameter transposition bugs (passing an integer timeout into an integer port parameter).
2. **JavaBeans Mutability Anti-Pattern**: Calling a zero-arg constructor followed by setters (`config.setHost(...)`, `config.setPort(...)`). This leaves objects in an **inconsistent, partially initialized state** mid-construction and precludes immutability, causing thread-safety hazards.

---

### 4.2 Production Implementation: Thread-Safe Database Connection Pool Config

```java
package com.systemdesign.creational.builder;

import java.time.Duration;
import java.util.*;

/**
 * Immutable, thread-safe configuration object.
 */
public final class DatabasePoolConfig {

    private final String jdbcUrl;
    private final String username;
    private final String password;
    private final int minIdleConnections;
    private final int maxPoolSize;
    private final Duration connectionTimeout;
    private final Duration idleTimeout;
    private final Map<String, String> connectionProperties;
    private final List<String> initSqlStatements;

    // Private constructor: instantiated exclusively by Builder
    private DatabasePoolConfig(Builder builder) {
        this.jdbcUrl = builder.jdbcUrl;
        this.username = builder.username;
        this.password = builder.password;
        this.minIdleConnections = builder.minIdleConnections;
        this.maxPoolSize = builder.maxPoolSize;
        this.connectionTimeout = builder.connectionTimeout;
        this.idleTimeout = builder.idleTimeout;
        // Defensive copying to ensure complete immutability
        this.connectionProperties = Collections.unmodifiableMap(new HashMap<>(builder.connectionProperties));
        this.initSqlStatements = Collections.unmodifiableList(new ArrayList<>(builder.initSqlStatements));
    }

    public static Builder newBuilder() {
        return new Builder();
    }

    // Getters
    public String getJdbcUrl() { return jdbcUrl; }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public int getMinIdleConnections() { return minIdleConnections; }
    public int getMaxPoolSize() { return maxPoolSize; }
    public Duration getConnectionTimeout() { return connectionTimeout; }
    public Duration getIdleTimeout() { return idleTimeout; }
    public Map<String, String> getConnectionProperties() { return connectionProperties; }
    public List<String> getInitSqlStatements() { return initSqlStatements; }

    public static final class Builder {
        private String jdbcUrl;
        private String username;
        private String password;
        private int minIdleConnections = 5;
        private int maxPoolSize = 20;
        private Duration connectionTimeout = Duration.ofSeconds(30);
        private Duration idleTimeout = Duration.ofMinutes(10);
        private final Map<String, String> connectionProperties = new HashMap<>();
        private final List<String> initSqlStatements = new ArrayList<>();

        private Builder() {}

        public Builder jdbcUrl(String jdbcUrl) {
            this.jdbcUrl = Objects.requireNonNull(jdbcUrl, "JDBC URL cannot be null");
            return this;
        }

        public Builder credentials(String username, String password) {
            this.username = Objects.requireNonNull(username, "Username cannot be null");
            this.password = Objects.requireNonNull(password, "Password cannot be null");
            return this;
        }

        public Builder poolLimits(int minIdle, int maxPool) {
            this.minIdleConnections = minIdle;
            this.maxPoolSize = maxPool;
            return this;
        }

        public Builder connectionTimeout(Duration timeout) {
            this.connectionTimeout = Objects.requireNonNull(timeout);
            return this;
        }

        public Builder idleTimeout(Duration timeout) {
            this.idleTimeout = Objects.requireNonNull(timeout);
            return this;
        }

        public Builder addProperty(String key, String value) {
            this.connectionProperties.put(Objects.requireNonNull(key), Objects.requireNonNull(value));
            return this;
        }

        public Builder addInitSql(String sql) {
            this.initSqlStatements.add(Objects.requireNonNull(sql));
            return this;
        }

        /**
         * Validates all cross-field constraints before returning the immutable instance.
         */
        public DatabasePoolConfig build() {
            // Validation invariants
            if (jdbcUrl == null || !jdbcUrl.startsWith("jdbc:")) {
                throw new IllegalStateException("Invalid JDBC URL format: " + jdbcUrl);
            }
            if (username == null || username.isBlank()) {
                throw new IllegalStateException("Database username is required");
            }
            if (minIdleConnections < 0) {
                throw new IllegalStateException("minIdleConnections cannot be negative");
            }
            if (maxPoolSize <= 0 || maxPoolSize < minIdleConnections) {
                throw new IllegalStateException(String.format(
                        "Invalid pool bounds: minIdle=%d, maxPool=%d", minIdleConnections, maxPoolSize));
            }
            if (connectionTimeout.isNegative() || connectionTimeout.isZero()) {
                throw new IllegalStateException("connectionTimeout must be strictly positive");
            }

            return new DatabasePoolConfig(this);
        }
    }
}
```

---

### 4.3 Staged Builder (Wizard / Directed Acyclic Graph Builder)
A **Staged Builder** uses typed interfaces to enforce the provision of mandatory fields at compile-time before optional configurations or `build()` can be called.

```java
package com.systemdesign.creational.builder;

import java.net.URI;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Type-safe Staged Builder enforcing: Method -> Url -> Payload/Headers -> Build
 */
public final class ImmutableHttpRequest {

    private final String method;
    private final URI uri;
    private final Map<String, String> headers;
    private final String body;

    private ImmutableHttpRequest(String method, URI uri, Map<String, String> headers, String body) {
        this.method = method;
        this.uri = uri;
        this.headers = Map.copyOf(headers);
        this.body = body;
    }

    public String getMethod() { return method; }
    public URI getUri() { return uri; }
    public Map<String, String> getHeaders() { return headers; }
    public String getBody() { return body; }

    // Entry point for the Stage Wizard
    public static MethodStage create() {
        return new RequestBuilder();
    }

    // Step 1: Mandatory HTTP Method
    public interface MethodStage {
        UrlStage get();
        UrlStage post();
        UrlStage put();
    }

    // Step 2: Mandatory URI
    public interface UrlStage {
        OptionalStage url(URI uri);
    }

    // Step 3: Optional Configurations + Final Build
    public interface OptionalStage {
        OptionalStage addHeader(String key, String value);
        OptionalStage body(String payload);
        ImmutableHttpRequest build();
    }

    private static final class RequestBuilder implements MethodStage, UrlStage, OptionalStage {
        private String method;
        private URI uri;
        private final Map<String, String> headers = new HashMap<>();
        private String body;

        @Override
        public UrlStage get() {
            this.method = "GET";
            return this;
        }

        @Override
        public UrlStage post() {
            this.method = "POST";
            return this;
        }

        @Override
        public UrlStage put() {
            this.method = "PUT";
            return this;
        }

        @Override
        public OptionalStage url(URI uri) {
            this.uri = Objects.requireNonNull(uri, "URI must not be null");
            return this;
        }

        @Override
        public OptionalStage addHeader(String key, String value) {
            this.headers.put(Objects.requireNonNull(key), Objects.requireNonNull(value));
            return this;
        }

        @Override
        public OptionalStage body(String payload) {
            this.body = payload;
            return this;
        }

        @Override
        public ImmutableHttpRequest build() {
            return new ImmutableHttpRequest(method, uri, headers, body);
        }
    }
}
```

* **Compilation Guarantee**: A caller literally **cannot** call `.build()` without supplying `.post()` and `.url(...)` first:
```java
// Compile-time enforced:
ImmutableHttpRequest req = ImmutableHttpRequest.create()
        .post()
        .url(URI.create("https://api.gateway.internal/v1/charge"))
        .addHeader("Content-Type", "application/json")
        .body("{\"amount\": 1000}")
        .build();
```

---

---

## Navigation & Related Patterns
* [Back to Creational Patterns Index](./04-creational-design-patterns.md)
* [Master LLD Index](./00-index.md)
* [Previous: 04-03-abstract-factory-pattern.md](./04-03-abstract-factory-pattern.md)
* [Next: 04-05-prototype-pattern.md](./04-05-prototype-pattern.md)
