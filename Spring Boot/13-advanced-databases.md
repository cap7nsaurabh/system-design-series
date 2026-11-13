# 13 — Advanced Database Architectures in Spring Boot 3.x (Beyond Basic JPA)

> **Module Goal:** Master advanced enterprise data tier architectures in Spring Boot 3.x (Java 17/21). From multi-tenant isolation models, dynamic read/write replica routing via `AbstractRoutingDataSource`, modern fluent `JdbcClient` and high-throughput batching, to reactive non-blocking SQL via R2DBC, polyglot persistence, and distributed transaction patterns (SAGA, Outbox CDC, and Two-Phase Commit).

---

## Table of Contents

1. [Architectural Breaking Points of Basic ORM/JPA](#1-architectural-breaking-points-of-basic-ormjpa)
2. [Multi-Tenancy Isolation Architectures](#2-multi-tenancy-isolation-architectures)
   - [Approach 1: Database-per-Tenant (Physical Isolation)](#approach-1-database-per-tenant-physical-isolation)
   - [Approach 2: Schema-per-Tenant (Logical Schema Isolation)](#approach-2-schema-per-tenant-logical-schema-isolation)
   - [Approach 3: Shared Database with Discriminator Column (`@TenantId`)](#approach-3-shared-database-with-discriminator-column-tenantid)
3. [Read/Write Replica Splitting & Dynamic DataSource Routing](#3-readwrite-replica-splitting--dynamic-datasource-routing)
   - [Approach 1: `AbstractRoutingDataSource` with `@Transactional(readOnly = true)`](#approach-1-abstractroutingdatasource-with-transactionalreadonly--true)
   - [Approach 2: Multiple Explicit Configured DataSources & Repositories](#approach-2-multiple-explicit-configured-datasources--repositories)
   - [Approach 3: Cloud-Native Infrastructure Routing (AWS Aurora Reader Endpoints)](#approach-3-cloud-native-infrastructure-routing-aws-aurora-reader-endpoints)
4. [High-Performance SQL Data Access Paradigms](#4-high-performance-sql-data-access-paradigms)
   - [Approach 1: Modern Fluent `JdbcClient` (Spring Boot 3.2+)](#approach-1-modern-fluent-jdbcclient-spring-boot-32)
   - [Approach 2: Low-Level `JdbcTemplate` Bulk Batch Updates](#approach-2-low-level-jdbctemplate-bulk-batch-updates)
   - [Approach 3: Spring Data JDBC Aggregate Root Persistence](#approach-3-spring-data-jdbc-aggregate-root-persistence)
   - [Approach 4: Non-Blocking Reactive SQL via Spring Data R2DBC](#approach-4-non-blocking-reactive-sql-via-spring-data-r2dbc)
5. [Distributed Transaction Patterns Across Microservices](#5-distributed-transaction-patterns-across-microservices)
   - [Approach 1: The Transactional Outbox Pattern with Debezium CDC](#approach-1-the-transactional-outbox-pattern-with-debezium-cdc)
   - [Approach 2: SAGA Orchestration Workflow](#approach-2-saga-orchestration-workflow)
   - [Approach 3: Two-Phase Commit (XA Distributed Transactions)](#approach-3-two-phase-commit-xa-distributed-transactions)
6. [Polyglot Persistence Integration (Relational + Document + Cache)](#6-polyglot-persistence-integration-relational--document--cache)
7. [Comprehensive Configuration Reference](#7-comprehensive-configuration-reference)
8. [⚠️ Interview Traps & Production Gotchas](#️-interview-traps--production-gotchas)
9. [📝 Architectural Comparison Matrix](#-architectural-comparison-matrix)
10. [❓ Practice Questions & Real-World Scenarios](#-practice-questions--real-world-scenarios)

---

## 1. Architectural Breaking Points of Basic ORM/JPA

While Spring Data JPA with Hibernate is the default persistence layer in Spring Boot, high-throughput systems frequently hit architectural walls:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        JPA / Hibernate Bottlenecks                     │
├─────────────────────────┬──────────────────────────────────────────────┤
│ First-Level Cache Bloat │ Retains snapshots of all loaded entities     │
│ Dirty-Checking Overhead │ Compares 10,000 entity snapshots at commit   │
│ N+1 Query Traps         │ Lazy proxies trigger sequential round trips  │
│ Uncontrolled SQL Batch  │ IDENTITY generation disables JDBC batching   │
│ Blocking Servlet I/O    │ 1 connection held per slow HTTP request      │
└─────────────────────────┴──────────────────────────────────────────────┘
```

When building SaaS multi-tenancy, read/write splitting, high-performance streaming, or distributed microservice consensus, engineers must look beyond basic JPA.

---

## 2. Multi-Tenancy Isolation Architectures

In multi-tenant SaaS applications, isolating tenant data is critical for compliance, security, and scalability.

```
                    Multi-Tenancy Isolation Models
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
Approach 1: Separate DB     Approach 2: Separate Schema   Approach 3: Shared Table
- Complete physical isolation - Logical namespace isolation - Row-level tenant_id
- High infrastructure cost    - Moderate isolation cost    - Lowest infrastructure cost
- Dedicated connection pools  - Single connection pool     - Highest risk of data leak
```

---

### Approach 1: Database-per-Tenant (Physical Isolation)

#### Scenario
Enterprise B2B SaaS (fintech, healthcare, defense) where enterprise customers demand dedicated physical databases, individual encryption keys, and independent backup/restore capabilities to satisfy regulatory requirements (HIPAA, SOC 2).

#### Use-Case
High-value clients paying a premium for total data isolation and dedicated database compute resources.

#### Benefits Over Others
- **Zero Risk of Cross-Tenant Data Leaks:** Accidental omission of a `WHERE` clause cannot expose other tenants' data.
- **Independent Maintenance & Backups:** Restore, migrate, or snapshot Tenant A's database without touching Tenant B.
- **Dedicated Resource Sizing:** Allocate a larger RDS instance to a high-volume enterprise customer.

#### Production-Grade Code Example

```java
package com.example.advanceddb.multitenancy.dbpertenant;

import com.zaxxer.hikari.HikariDataSource;
import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.hibernate.cfg.AvailableSettings;
import org.hibernate.context.spi.CurrentTenantIdentifierResolver;
import org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider;
import org.springframework.boot.autoconfigure.orm.jpa.JpaProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

// 1. ThreadLocal Tenant Identifier Resolution
@Component
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = ThreadLocal.withInitial(() -> "default");

    public static void setTenantId(String tenantId) { CURRENT_TENANT.set(tenantId); }
    public static String getTenantId() { return CURRENT_TENANT.get(); }
    public static void clear() { CURRENT_TENANT.remove(); }
}

@Component
class HibernateTenantIdentifierResolver implements CurrentTenantIdentifierResolver {
    @Override
    public String resolveCurrentTenantIdentifier() {
        String tenant = TenantContext.getTenantId();
        return tenant != null ? tenant : "default";
    }

    @Override
    public boolean validateExistingCurrentSessions() { return true; }
}

// 2. Dynamic MultiTenantConnectionProvider managing distinct HikariCP pools
@Component
@Slf4j
class DynamicDatabaseMultiTenantConnectionProvider implements MultiTenantConnectionProvider {

    private final Map<String, DataSource> tenantDataSources = new ConcurrentHashMap<>();
    private DataSource defaultDataSource;

    @PostConstruct
    void init() {
        defaultDataSource = createPool("default_db");
        tenantDataSources.put("default", defaultDataSource);
        tenantDataSources.put("tenant_alpha", createPool("tenant_alpha_db"));
        tenantDataSources.put("tenant_beta", createPool("tenant_beta_db"));
    }

    private DataSource createPool(String dbName) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/" + dbName);
        ds.setUsername("postgres");
        ds.setPassword("secret");
        ds.setMaximumPoolSize(10);
        ds.setPoolName("HikariPool-" + dbName);
        return ds;
    }

    @Override
    public Connection getAnyConnection() throws SQLException {
        return defaultDataSource.getConnection();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public Connection getConnection(Object tenantIdentifier) throws SQLException {
        String tenantId = (String) tenantIdentifier;
        DataSource ds = tenantDataSources.getOrDefault(tenantId, defaultDataSource);
        log.debug("Acquired physical connection for tenant '{}' from pool", tenantId);
        return ds.getConnection();
    }

    @Override
    public void releaseConnection(Object tenantIdentifier, Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) { return false; }
    @Override
    public <T> T unwrap(Class<T> unwrapType) { return null; }
    @Override
    public boolean supportsAggressiveRelease() { return true; }
}

// 3. Wiring into EntityManagerFactory
@Configuration
class MultiTenantJpaConfig {

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            MultiTenantConnectionProvider multiTenantConnectionProvider,
            CurrentTenantIdentifierResolver tenantIdentifierResolver,
            JpaProperties jpaProperties) {

        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setPackagesToScan("com.example.advanceddb.domain");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());

        Map<String, Object> jpaPropsMap = new HashMap<>(jpaProperties.getProperties());
        jpaPropsMap.put(AvailableSettings.MULTI_TENANT_CONNECTION_PROVIDER, multiTenantConnectionProvider);
        jpaPropsMap.put(AvailableSettings.MULTI_TENANT_IDENTIFIER_RESOLVER, tenantIdentifierResolver);
        em.setJpaPropertyMap(jpaPropsMap);

        return em;
    }
}
```

---

### Approach 2: Schema-per-Tenant (Logical Schema Isolation)

#### Scenario
SaaS platforms with hundreds of mid-tier clients where creating 500 separate PostgreSQL databases would waste hardware memory (each database running separate buffer pools, WAL buffers, and catalog caches).

#### Use-Case
PostgreSQL or Oracle installations providing schema-level isolation (`tenant_101.orders`, `tenant_102.orders`) over a single physical database instance and a single shared connection pool.

#### Benefits Over Others
- **Connection Pool Efficiency:** All tenants share a single HikariCP pool.
- **Rapid Provisioning:** Creating a new tenant is as simple as executing `CREATE SCHEMA tenant_103;` and running migrations.
- **Moderate Data Isolation:** Protects against accidental cross-schema joins via database role permissions.

#### Production-Grade Code Example

```java
package com.example.advanceddb.multitenancy.schemapertenant;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

@Component
@RequiredArgsConstructor
@Slf4j
public class SchemaPerTenantConnectionProvider implements MultiTenantConnectionProvider {

    private final DataSource dataSource;

    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSource.getConnection();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public Connection getConnection(Object tenantIdentifier) throws SQLException {
        String schema = tenantIdentifier != null ? tenantIdentifier.toString() : "public";
        Connection connection = getAnyConnection();
        try {
            // Set the active search_path for PostgreSQL dynamically
            log.debug("Switching active schema to '{}'", schema);
            connection.createStatement().execute("SET search_path TO " + schema + ", public");
        } catch (SQLException e) {
            connection.close();
            throw new SQLException("Could not switch connection to schema: " + schema, e);
        }
        return connection;
    }

    @Override
    public void releaseConnection(Object tenantIdentifier, Connection connection) throws SQLException {
        try {
            // Reset schema back to default public before returning connection to HikariCP
            connection.createStatement().execute("SET search_path TO public");
        } finally {
            connection.close();
        }
    }

    @Override public boolean supportsAggressiveRelease() { return true; }
    @Override public boolean isUnwrappableAs(Class<?> unwrapType) { return false; }
    @Override public <T> T unwrap(Class<T> unwrapType) { return null; }
}
```

---

### Approach 3: Shared Database with Discriminator Column (`@TenantId`)

#### Scenario
Mass-market consumer or micro-business SaaS supporting 100,000+ tenants where managing separate databases or schemas is operationally impractical.

#### Use-Case
Low-cost SaaS tiers where every table includes a `tenant_id` column, and Hibernate 6+ automatically injects tenant filters into every SQL query.

#### Benefits Over Others
- **Minimal Infrastructure Cost:** Operates on a single database schema and table structure.
- **Automated Filter Injection:** Hibernate 6's native `@TenantId` automatically appends `AND tenant_id = ?` to all JPQL, Criteria, and raw entity updates, preventing developer omissions.

#### Production-Grade Code Example

```java
package com.example.advanceddb.multitenancy.discriminator;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.hibernate.annotations.TenantId;

@Entity
@Table(name = "invoices")
@Getter
@Setter
public class Invoice {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // Hibernate 6+ native discriminator annotation
    // Automatically populated from CurrentTenantIdentifierResolver!
    // Injects "WHERE tenant_id = ?" into EVERY SELECT and INSERT!
    @TenantId
    @Column(name = "tenant_id", nullable = false, updatable = false)
    private String tenantId;

    @Column(nullable = false)
    private String invoiceNumber;

    @Column(nullable = false)
    private Double amount;
}
```

---

## 3. Read/Write Replica Splitting & Dynamic DataSource Routing

High-volume relational workloads typically feature an 80:20 read-to-write ratio. Routing read queries to read-replicas offloads the primary database, reserving it for write transactions.

```
                           Incoming Web Request
                                     │
                                     ▼
                      Spring Transaction Interceptor
                                     │
                 ┌───────────────────┴───────────────────┐
                 ▼                                       ▼
    @Transactional(readOnly = false)         @Transactional(readOnly = true)
                 │                                       │
                 ▼                                       ▼
     Tenant/Routing Context: MASTER          Tenant/Routing Context: REPLICA
                 │                                       │
                 └───────────────────┬───────────────────┘
                                     ▼
                         AbstractRoutingDataSource
                                     │
                 ┌───────────────────┴───────────────────┐
                 ▼                                       ▼
        Primary DataSource (Write)              Replica DataSource (Read)
        - INSERT, UPDATE, DELETE                - SELECT Queries
```

---

### Approach 1: `AbstractRoutingDataSource` with `@Transactional(readOnly = true)`

#### Scenario
Applications connecting to a traditional database cluster (PostgreSQL with read-replicas, MySQL Primary/Replica) where read queries should automatically execute against read-only replicas.

#### Use-Case
Transparent read/write splitting at the application layer without changing existing Spring Data JPA repository code.

#### Benefits Over Others
- **Zero Query Rewrites:** Existing repository queries automatically route to the replica if called inside a `@Transactional(readOnly = true)` method.
- **Connection Isolation:** Master and replica pools are sized and managed independently.

#### Production-Grade Code Example

```java
package com.example.advanceddb.routing;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

// 1. The Dynamic Routing Router
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {

    public enum DataSourceType { MASTER, REPLICA }

    @Override
    protected Object determineCurrentLookupKey() {
        // Inspect Spring's TransactionSynchronizationManager
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        return isReadOnly ? DataSourceType.REPLICA : DataSourceType.MASTER;
    }
}

// 2. Multi-DataSource Bean Configuration
@Configuration
public class RoutingDataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource masterDataSource,
            @Qualifier("replicaDataSource") DataSource replicaDataSource) {

        ReadWriteRoutingDataSource routingDataSource = new ReadWriteRoutingDataSource();

        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(ReadWriteRoutingDataSource.DataSourceType.MASTER, masterDataSource);
        targetDataSources.put(ReadWriteRoutingDataSource.DataSourceType.REPLICA, replicaDataSource);

        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource); // Fallback to master
        return routingDataSource;
    }
}
```

---

### Approach 2: Multiple Explicit Configured DataSources & Repositories

#### Scenario
An enterprise application that connects to two completely different physical databases simultaneously (e.g., a primary PostgreSQL database for core banking, and a secondary MySQL database for legacy audit reporting).

#### Use-Case
Microservices orchestrating transactions across multiple distinct data repositories.

#### Benefits Over Others
- **Complete Domain Segregation:** Entities and repositories are partitioned into separate Java packages.
- **Distinct Dialects:** Supports different database engines (e.g., PostgreSQL for DB 1, Oracle for DB 2).

#### Production-Grade Code Example

```java
package com.example.advanceddb.multipleds;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.boot.orm.jpa.EntityManagerFactoryBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;

@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.advanceddb.repository.order",
        entityManagerFactoryRef = "orderEntityManagerFactory",
        transactionManagerRef = "orderTransactionManager"
)
public class OrderDataSourceConfig {

    @Primary
    @Bean(name = "orderDataSource")
    @ConfigurationProperties(prefix = "app.datasource.orders")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Primary
    @Bean(name = "orderEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean orderEntityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("orderDataSource") DataSource dataSource) {
        return builder
                .dataSource(dataSource)
                .packages("com.example.advanceddb.domain.order")
                .persistenceUnit("orders")
                .build();
    }

    @Primary
    @Bean(name = "orderTransactionManager")
    public PlatformTransactionManager orderTransactionManager(
            @Qualifier("orderEntityManagerFactory") LocalContainerEntityManagerFactoryBean factory) {
        return new JpaTransactionManager(factory.getObject());
    }
}
```

---

### Approach 3: Cloud-Native Infrastructure Routing (AWS Aurora Reader Endpoints)

#### Scenario
Modern cloud deployments running on AWS Aurora PostgreSQL or MySQL clusters.

#### Use-Case
Using the cloud platform's built-in reader and writer DNS endpoints instead of managing routing logic inside Java code.

#### Benefits Over Others
- **Zero Custom Java Routing Code:** Spring Boot connects to the writer endpoint for writes and the reader endpoint for reads.
- **Automated Failover:** If the Aurora primary node crashes, AWS promotes a replica and re-points the DNS writer record automatically.

---

## 4. High-Performance SQL Data Access Paradigms

When query performance and memory footprint are paramount, alternatives to basic JPA provide significant benefits.

```
┌────────────────────────────────────────────────────────────────────────┐
│               Spring Boot Data Access Performance Matrix               │
├─────────────────┬──────────────┬───────────────┬───────────────────────┤
│ Framework       │ Memory Usage │ Control       │ Best Used For         │
├─────────────────┼──────────────┼───────────────┼───────────────────────┤
│ Spring Data JPA │ High         │ Low/Generated │ Complex domain graphs │
│ Fluent          │ Low          │ High/Raw SQL  │ Modern Spring Boot 3.2│
│ JdbcClient      │              │               │ microservices         │
│ JdbcTemplate    │ Low          │ High/Raw SQL  │ Legacy & bulk batch   │
│ Spring Data     │ Low          │ Moderate      │ Pure DDD aggregates   │
│ JDBC            │              │               │ without proxies       │
│ Spring Data     │ Minimal      │ High/Reactive │ WebFlux non-blocking  │
│ R2DBC           │              │               │ high concurrency      │
└─────────────────┴──────────────┴───────────────┴───────────────────────┘
```

---

### Approach 1: Modern Fluent `JdbcClient` (Spring Boot 3.2+)

#### Scenario
Spring Boot 3.2+ applications requiring high-performance SQL execution and clean DTO mapping without the boilerplate of `RowMapper` implementations.

#### Use-Case
The modern replacement for `JdbcTemplate` and `NamedParameterJdbcTemplate` in Spring Boot 3.2+.

#### Benefits Over Others
- **Fluent API:** Clean, readable method chaining (`client.sql(...).param(...).query(...).list()`).
- **Automatic Record Class Mapping:** Automatically maps SQL columns directly to Java 21 record components.
- **Zero Hibernate Overhead:** Bypasses entity lifecycle, proxy generation, and dirty-checking.

#### Production-Grade Code Example

```java
package com.example.advanceddb.jdbcclient;

import lombok.RequiredArgsConstructor;
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

public record CustomerAnalyticsDto(Long id, String name, String tier, BigDecimal lifetimeValue) {}

@Repository
@RequiredArgsConstructor
public class HighPerformanceAnalyticsRepository {

    // Spring Boot 3.2+ Auto-Configured Fluent Client
    private final JdbcClient jdbcClient;

    public Optional<CustomerAnalyticsDto> findTopCustomerByRegion(String region, LocalDate since) {
        String sql = """
            SELECT c.id, c.name, c.tier, SUM(o.total_amount) AS lifetimeValue
            FROM customers c
            JOIN orders o ON c.id = o.customer_id
            WHERE c.region = :region AND o.order_date >= :since
            GROUP BY c.id, c.name, c.tier
            ORDER BY lifetimeValue DESC
            LIMIT 1
        """;

        return jdbcClient.sql(sql)
                .param("region", region)
                .param("since", since)
                .query(CustomerAnalyticsDto.class) // Automatically maps directly to Java record!
                .optional();
    }

    public List<CustomerAnalyticsDto> listVipCustomers(BigDecimal minSpend) {
        return jdbcClient.sql("SELECT id, name, tier, balance AS lifetimeValue FROM customers WHERE balance > ?")
                .param(minSpend)
                .query(CustomerAnalyticsDto.class)
                .list();
    }
}
```

---

### Approach 2: Low-Level `JdbcTemplate` Bulk Batch Updates

#### Scenario
Inserting or updating 50,000+ records during ETL jobs, CSV imports, or end-of-day reconciliation.

#### Use-Case
High-throughput batch operations where minimizing network round trips is critical.

#### Benefits Over Others
- **Single Network Round-Trip:** Groups thousands of insert statements into a single wire packet.
- **Database Driver Optimization:** Combining `batchUpdate()` with `rewriteBatchedStatements=true` allows MySQL and PostgreSQL drivers to rewrite statements into high-efficiency multi-row inserts (`INSERT INTO t VALUES (...), (...), (...)`).

#### Production-Grade Code Example

```java
package com.example.advanceddb.batch;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.jdbc.core.BatchPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;

public record StockTick(String symbol, double price, long volume, long epochMillis) {}

@Service
@RequiredArgsConstructor
@Slf4j
public class BulkIngestionService {

    private final JdbcTemplate jdbcTemplate;

    @Transactional
    public void ingestTicksInBatches(List<StockTick> ticks, int batchSize) {
        String sql = "INSERT INTO market_ticks (symbol, price, volume, tick_time) VALUES (?, ?, ?, ?)";

        long start = System.currentTimeMillis();

        for (int i = 0; i < ticks.size(); i += batchSize) {
            List<StockTick> batch = ticks.subList(i, Math.min(i + batchSize, ticks.size()));

            jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement ps, int index) throws SQLException {
                    StockTick tick = batch.get(index);
                    ps.setString(1, tick.symbol());
                    ps.setDouble(2, tick.price());
                    ps.setLong(3, tick.volume());
                    ps.setLong(4, tick.epochMillis());
                }

                @Override
                public int getBatchSize() {
                    return batch.size();
                }
            });
        }

        log.info("Ingested {} market ticks in {} ms", ticks.size(), (System.currentTimeMillis() - start));
    }
}
```

---

### Approach 3: Spring Data JDBC Aggregate Root Persistence

#### Scenario
Domain-Driven Design (DDD) projects rejecting JPA's complexity, lazy loading traps, and proxy behaviors.

#### Use-Case
DDD applications where entities are modeled strictly as cohesive Aggregate Roots.

#### Benefits Over Others
- **No Lazy Loading Traps:** Relationships are loaded eagerly without proxy generation.
- **Explicit Saves:** No unexpected updates from automatic dirty-checking.

#### Production-Grade Code Example

```java
package com.example.advanceddb.datajdbc;

import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.MappedCollection;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.repository.CrudRepository;

import java.util.Set;

@Table("orders")
public record OrderAggregate(
        @Id Long id,
        String customerEmail,
        @MappedCollection(idColumn = "order_id") Set<OrderItem> items
) {}

@Table("order_items")
public record OrderItem(String productSku, int quantity, double price) {}

public interface OrderAggregateRepository extends CrudRepository<OrderAggregate, Long> {}
```

---

### Approach 4: Non-Blocking Reactive SQL via Spring Data R2DBC

#### Scenario
High-concurrency streaming applications built on Spring WebFlux, Netty, and reactive streams.

#### Use-Case
Reactive microservices handling tens of thousands of concurrent connections using a small, fixed thread pool.

#### Benefits Over Others
- **Zero Blocking I/O:** Worker threads are never blocked waiting for database network responses.
- **Resource Efficiency:** Handles high concurrency with a minimal memory footprint.

#### Production-Grade Code Example

```java
package com.example.advanceddb.r2dbc;

import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Table("device_telemetry")
public record DeviceTelemetry(@Id Long id, String deviceSerial, Double temperature, Long timestamp) {}

public interface ReactiveTelemetryRepository extends ReactiveCrudRepository<DeviceTelemetry, Long> {
    Flux<DeviceTelemetry> findByDeviceSerial(String deviceSerial);
}

@Service
public class ReactiveTelemetryService {

    private final ReactiveTelemetryRepository repository;

    public ReactiveTelemetryService(ReactiveTelemetryRepository repository) {
        this.repository = repository;
    }

    public Flux<DeviceTelemetry> streamTelemetry(String serial) {
        // Returns a non-blocking reactive stream directly to the client
        return repository.findByDeviceSerial(serial);
    }
}
```

---

## 5. Distributed Transaction Patterns Across Microservices

In distributed microservice architectures, traditional single-database ACID transactions are not possible across service boundaries.

```
Distributed Transaction Architectural Choices
                     │
    ┌────────────────┼────────────────┐
    ▼                ▼                ▼
Transactional     SAGA Pattern       Two-Phase Commit (XA)
Outbox + CDC     - Orchestrated      - Distributed Lock
- Guaranteed       compensation      - Heavy coordinator
  at-least-once  - Eventual          - Avoid in modern cloud
  event delivery   consistency
```

---

### Approach 1: The Transactional Outbox Pattern with Debezium CDC

#### Scenario
Microservices that must atomically mutate a database record and publish an event to Apache Kafka without risk of dual-write discrepancies.

#### Use-Case
The industry-standard architectural pattern for reliable event publishing in distributed systems.

#### Benefits Over Others
- **ACID Atomic Reliability:** The business mutation and the outbox event record are committed together in the **same local database transaction**.
- **Zero Dual-Write Failures:** If the transaction rolls back, no message is published.
- **Non-Blocking CDC Extraction:** Debezium reads the database transaction log directly, imposing near-zero overhead on the application.

#### Production-Grade Code Example

```java
package com.example.advanceddb.outbox;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.UUID;

// 1. The Outbox Table Entity
@Entity
@Table(name = "outbox_events")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String eventType;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String payload;

    @Column(nullable = false)
    private Instant createdAt;
}

// 2. Service performing atomic dual-state write in a single local transaction
@Service
@RequiredArgsConstructor
public class OrderServiceWithOutbox {

    private final EntityManager entityManager;
    private final ObjectMapper objectMapper;

    @Transactional
    public String createOrder(String customerId, double total) throws Exception {
        String orderId = UUID.randomUUID().toString();

        // 1. Write business entity
        entityManager.createNativeQuery(
                "INSERT INTO orders (id, customer_id, total_amount) VALUES (?, ?, ?)")
                .setParameter(1, orderId)
                .setParameter(2, customerId)
                .setParameter(3, total)
                .executeUpdate();

        // 2. Write event to outbox in the SAME local transaction
        String eventPayload = objectMapper.writeValueAsString(Map.of(
                "orderId", orderId,
                "customerId", customerId,
                "amount", total
        ));

        OutboxEvent outbox = OutboxEvent.builder()
                .aggregateType("Order")
                .aggregateId(orderId)
                .eventType("ORDER_CREATED")
                .payload(eventPayload)
                .createdAt(Instant.now())
                .build();

        entityManager.persist(outbox);

        // Debezium captures this row via WAL and streams it to Apache Kafka!
        return orderId;
    }
}
```

---

### Approach 2: SAGA Orchestration Workflow

#### Scenario
Multi-step distributed business processes across multiple microservices (e.g., Order Service -> Payment Service -> Inventory Service -> Shipping Service) requiring rollback capabilities without two-phase commit.

#### Use-Case
E-commerce checkout workflows where each step can succeed or fail, and previous steps must be reversed using **compensating transactions**.

#### Benefits Over Others
- **High Concurrency:** Services do not hold database locks across network boundaries.
- **Eventual Consistency:** System eventually reaches a consistent state through compensating actions.

#### Production-Grade Code Example

```java
package com.example.advanceddb.saga;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class CheckoutSagaOrchestrator {

    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;

    public boolean executeOrderSaga(String orderId, double amount, String itemSku) {
        log.info("Initiating checkout SAGA for order: {}", orderId);

        // Step 1: Authorize payment
        boolean paymentAuthorized = paymentClient.authorizePayment(orderId, amount);
        if (!paymentAuthorized) {
            log.error("SAGA Step 1 failed: Payment rejected. Aborting.");
            return false;
        }

        // Step 2: Reserve inventory
        boolean inventoryReserved = inventoryClient.reserveInventory(orderId, itemSku);
        if (!inventoryReserved) {
            log.warn("SAGA Step 2 failed: Inventory out of stock! Executing COMPENSATING transaction...");
            // Execute compensation: Refund payment!
            paymentClient.refundPayment(orderId, amount);
            return false;
        }

        log.info("Checkout SAGA completed successfully for order: {}", orderId);
        return true;
    }
}

interface PaymentClient {
    boolean authorizePayment(String orderId, double amount);
    void refundPayment(String orderId, double amount);
}

interface InventoryClient {
    boolean reserveInventory(String orderId, String sku);
}
```

---

### Approach 3: Two-Phase Commit (XA Distributed Transactions)

#### Scenario
Legacy mainframe integrations or strict compliance architectures where eventual consistency is unacceptable.

#### Use-Case
Distributed transactions spanning multiple relational databases or JMS queues that must commit or abort atomically via a coordinator (e.g., Narayana, Atomikos).

#### Trade-offs & Limitations
- **Heavy Performance Cost:** Two-phase locking across network boundaries significantly reduces throughput.
- **Single Point of Failure:** If the transaction coordinator crashes during phase two, resources remain locked indefinitely.

---

## 6. Polyglot Persistence Integration (Relational + Document + Cache)

Modern systems combine multiple database technologies to match specific access patterns:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Polyglot Persistence Layer                        │
├───────────────────┬──────────────────┬─────────────────────────────────┤
│ Engine            │ Role             │ Use-Case                        │
├───────────────────┼──────────────────┼─────────────────────────────────┤
│ PostgreSQL        │ Relational OLTP  │ Users, Orders, Billing, ACID    │
│ MongoDB           │ Document Store   │ Product Catalog, Dynamic Schema │
│ Redis             │ Key-Value Store  │ Distributed Locks, Caching      │
└───────────────────┴──────────────────┴─────────────────────────────────┘
```

---

## 7. Comprehensive Configuration Reference

```yaml
spring:
  # Master Read/Write Routing DataSource
  datasource:
    master:
      jdbc-url: jdbc:postgresql://db-primary.prod.internal:5432/core_db
      username: app_user
      password: ${DB_PASSWORD}
      driver-class-name: org.postgresql.Driver
      hikari:
        maximum-pool-size: 20
        connection-timeout: 10000
    replica:
      jdbc-url: jdbc:postgresql://db-replica.prod.internal:5432/core_db?readOnly=true
      username: app_user
      password: ${DB_PASSWORD}
      driver-class-name: org.postgresql.Driver
      hikari:
        maximum-pool-size: 40
        connection-timeout: 5000

  jpa:
    open-in-view: false # CRITICAL: Disable OSIV in production!
    properties:
      hibernate:
        jdbc:
          batch_size: 50 # Enable JDBC batching
          order_inserts: true
          order_updates: true
```

---

## 8. ⚠️ Interview Traps & Production Gotchas

### Trap 1: `AbstractRoutingDataSource` Binds Connection on Transaction Start

> [!CAUTION]
> If a `@Transactional` method starts as read-write (`readOnly = false`), Spring acquires a connection from the master pool immediately. Calling a secondary `@Transactional(readOnly = true)` method within that same execution will **not** switch to the replica pool because the transaction is already bound to the master connection.

---

### Trap 2: Mixing JPA and `JdbcTemplate` in the Same Transaction

If you mutate an entity with JPA and then execute a raw `JdbcTemplate` query in the same transaction, the raw SQL query may not see the JPA mutations. Hibernate delays `INSERT` and `UPDATE` statements until flush time. Always call `entityManager.flush()` before executing raw SQL queries within the same transaction.

---

### Trap 3: PostgreSQL Batch Insert Driver URL Missing

Calling `jdbcTemplate.batchUpdate()` on PostgreSQL without setting `reWriteBatchedInserts=true` in the JDBC URL causes the driver to execute individual `INSERT` statements sequentially behind the scenes.

---

## 9. 📝 Architectural Comparison Matrix

| Pattern | Isolation Level | Complexity | Scalability | Primary Failure Mode |
|---|---|---|---|---|
| **Database-per-Tenant** | Total Physical | High | Low (Pool limits) | Connection pool exhaustion |
| **Schema-per-Tenant** | Moderate Logical | Medium | High | Schema migration lag |
| **Shared `@TenantId`** | Low (Logical) | Low | Very High | Accidental filter omission |
| **Read/Write Splitting** | Eventual Read | Medium | High | Replication lag dirty reads |
| **Outbox + CDC** | Eventual Event | Medium | Extreme | Debezium pipe backlog |
| **SAGA Orchestrator** | Eventual State | High | High | Compensation failure |

---

## 10. ❓ Practice Questions & Real-World Scenarios

1. **How does `AbstractRoutingDataSource` determine which database pool to use at runtime?**
   - *Answer:* It overrides `determineCurrentLookupKey()`, which inspects a `ThreadLocal` or Spring's `TransactionSynchronizationManager.isCurrentTransactionReadOnly()` to return the appropriate lookup key (e.g., `MASTER` vs `REPLICA`).

2. **Why does `@GeneratedValue(strategy = GenerationType.IDENTITY)` disable Hibernate batch inserts?**
   - *Answer:* `IDENTITY` generation requires the database to generate the primary key upon row insertion. Hibernate must execute each `INSERT` immediately to retrieve the generated ID, preventing it from batching multiple statements together.

3. **How does the Transactional Outbox pattern prevent dual-write inconsistencies?**
   - *Answer:* It writes both the domain entity mutation and the event notification record into the same local database within a single ACID transaction. If the transaction fails, both writes roll back. A separate CDC engine (Debezium) or poller reads the outbox table to publish events reliably.
