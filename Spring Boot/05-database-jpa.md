# Module 05: Database & JPA/Hibernate Deep Dive in Spring Boot
[🏠 Course Index](./00-course-index.md) | [← 04 — Exception Handling](./04-exception-handling.md) | **Module 05** | [Next: 06 — Concurrency →](./06-concurrency.md)

---

> **Series:** Spring Boot Mastery | **Level:** Intermediate → Advanced  
> **Spring Boot Version:** 3.x / Spring Framework 6.x | **Java Baseline:** Java 17/21+ (Jakarta EE 10 / Hibernate 6.x)  
> **Focus:** High-throughput database interaction, multi-approach query formulation, N+1 query elimination, transaction boundaries, and HikariCP connection pooling.

---

## Table of Contents

1. [Spring Data JPA & Hibernate Architecture](#1-spring-data-jpa--hibernate-architecture)
   - [1.1 The Layer Cake: JDBC vs JPA vs Hibernate vs Spring Data JPA](#11-the-layer-cake-jdbc-vs-jpa-vs-hibernate-vs-spring-data-jpa)
   - [1.2 Spring Data Repository Hierarchy](#12-spring-data-repository-hierarchy)
2. [Query Formulation Strategies (6 Approaches)](#2-query-formulation-strategies-6-approaches)
   - [2.1 Approach 1: Derived Query Methods](#21-approach-1-derived-query-methods)
   - [2.2 Approach 2: Explicit JPQL Queries (`@Query`)](#22-approach-2-explicit-jpql-queries-query)
   - [2.3 Approach 3: Native SQL Queries (`nativeQuery = true`)](#23-approach-3-native-sql-queries-nativequery--true)
   - [2.4 Approach 4: Spring Data JPA Specifications (Criteria API)](#24-approach-4-spring-data-jpa-specifications-criteria-api)
   - [2.5 Approach 5: QueryDSL (Type-Safe Fluent Queries)](#25-approach-5-querydsl-type-safe-fluent-queries)
   - [2.6 Approach 6: Fluent `JdbcClient` (Spring Boot 3.2+ High Performance)](#26-approach-6-fluent-jdbcclient-spring-boot-32-high-performance)
   - [2.7 Query Formulation Comparison Matrix](#27-query-formulation-comparison-matrix)
3. [Entity Mapping & Schema Modeling](#3-entity-mapping--schema-modeling)
   - [3.1 Core Entity Annotations & Mapping](#31-core-entity-annotations--mapping)
   - [3.2 Primary Key Generation Strategies & JDBC Batching Impact](#32-primary-key-generation-strategies--jdbc-batching-impact)
   - [3.3 Composite Keys (`@Embeddable` & `@EmbeddedId`)](#33-composite-keys-embeddable--embeddedid)
   - [3.4 Base Entity Inheritance (`@MappedSuperclass`)](#34-base-entity-inheritance-mappedsuperclass)
4. [Entity Relationships & Performance Gotchas](#4-entity-relationships--performance-gotchas)
   - [4.1 Owning vs Inverse Sides of Relationships](#41-owning-vs-inverse-sides-of-relationships)
   - [4.2 Bidirectional `@OneToMany` & Synchronization Helpers](#42-bidirectional-onetomany--synchronization-helpers)
   - [4.3 Cascade Types & The `@ManyToMany` REMOVE Hazard](#43-cascade-types--the-manytomany-remove-hazard)
   - [4.4 FetchType: LAZY vs EAGER Defaults](#44-fetchtype-lazy-vs-eager-defaults)
5. [The N+1 Select Problem & 4 Architectural Solutions](#5-the-n1-select-problem--4-architectural-solutions)
   - [5.1 The Root Cause & Detection Strategies](#51-the-root-cause--detection-strategies)
   - [5.2 Solution 1: JPQL `JOIN FETCH`](#52-solution-1-jpql-join-fetch)
   - [5.3 Solution 2: Declarative `@EntityGraph`](#53-solution-2-declarative-entitygraph)
   - [5.4 Solution 3: Hibernate Batch Fetching (`@BatchSize`)](#54-solution-3-hibernate-batch-fetching-batchsize)
   - [5.5 Solution 4: DTO Constructor Projections](#55-solution-4-dto-constructor-projections)
   - [5.6 N+1 Resolution Comparison Matrix](#56-n1-resolution-comparison-matrix)
6. [Transaction Management Deep Dive (3 Approaches)](#6-transaction-management-deep-dive-3-approaches)
   - [6.1 The Spring Proxy Mechanism](#61-the-spring-proxy-mechanism)
   - [6.2 Approach 1: Declarative `@Transactional` Boundaries](#62-approach-1-declarative-transactional-boundaries)
   - [6.3 Approach 2: Programmatic `TransactionTemplate`](#63-approach-2-programmatic-transactiontemplate)
   - [6.4 Approach 3: Read-Only Optimization (`readOnly = true`)](#64-approach-3-read-only-optimization-readonly--true)
   - [6.5 Transaction Propagation & Isolation Levels](#65-transaction-propagation--isolation-levels)
7. [Lazy Loading Traps & The OSIV Anti-Pattern](#7-lazy-loading-traps--the-osiv-anti-pattern)
   - [7.1 `LazyInitializationException` Mechanics](#71-lazyinitializationexception-mechanics)
   - [7.2 Open Session In View (OSIV) Production Hazards](#72-open-session-in-view-osiv-production-hazards)
8. [Database Concurrency & Locking Strategies (4 Approaches)](#8-database-concurrency--locking-strategies-4-approaches)
   - [8.1 Approach 1: Optimistic Locking with `@Version`](#81-approach-1-optimistic-locking-with-version)
   - [8.2 Approach 2: Pessimistic Write Lock (`PESSIMISTIC_WRITE`)](#82-approach-2-pessimistic-write-lock-pessimistic_write)
   - [8.3 Approach 3: Pessimistic Read Lock (`PESSIMISTIC_READ`)](#83-approach-3-pessimistic-read-lock-pessimistic_read)
   - [8.4 Approach 4: Database Unique Idempotency Keys](#84-approach-4-database-unique-idempotency-keys)
   - [8.5 Locking Strategies Comparison Matrix](#85-locking-strategies-comparison-matrix)
9. [Database Schema Migrations (Flyway vs Liquibase)](#9-database-schema-migrations-flyway-vs-liquibase)
   - [9.1 Flyway Setup & Zero-Downtime Migration Patterns](#91-flyway-setup--zero-downtime-migration-patterns)
   - [9.2 Liquibase Overview](#92-liquibase-overview)
10. [Connection Pooling with HikariCP](#10-connection-pooling-with-hikaricp)
    - [10.1 Key Configuration Parameters](#101-key-configuration-parameters)
    - [10.2 Pool Sizing Formula & Saturation Protection](#102-pool-sizing-formula--saturation-protection)
11. [Entity Auditing Strategies (3 Approaches)](#11-entity-auditing-strategies-3-approaches)
12. [⚠️ High-Frequency Interview Traps & JPA Gotchas](#12-️-high-frequency-interview-traps--jpa-gotchas)
13. [📝 Architecture Principle Checklist](#13--architecture-principle-checklist)
14. [❓ Practice Questions & Answers](#14--practice-questions--answers)

---

## 1. Spring Data JPA & Hibernate Architecture

### 1.1 The Layer Cake: JDBC vs JPA vs Hibernate vs Spring Data JPA

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Spring Data JPA Layer Cake                           │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Spring Data JPA                                                     │
│    Repository Interfaces (JpaRepository, CrudRepository)               │
│    Dynamic proxy generation, query method parsing                      │
├────────────────────────────────────────────────────────────────────────┤
│ 2. JPA (Jakarta Persistence API 3.1)                                   │
│    Standard API specification: @Entity, @Id, EntityManager             │
│    Zero implementation code (interfaces and metadata rules only)        │
├────────────────────────────────────────────────────────────────────────┤
│ 3. Hibernate ORM 6.x (JPA Provider / Implementation)                   │
│    Session, First-Level Cache, Dirty Checking, HQL/JPQL to SQL parser   │
├────────────────────────────────────────────────────────────────────────┤
│ 4. JDBC (Java Database Connectivity) & HikariCP                        │
│    Connection pools, PreparedStatement execution, ResultSet mapping    │
├────────────────────────────────────────────────────────────────────────┤
│ 5. Database Engine (PostgreSQL, MySQL, Oracle, H2)                     │
└────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> When you invoke `userRepository.save(user)`:
> 1. Spring Data JPA routes the call to an auto-generated repository proxy.
> 2. The proxy delegates to the active JPA `EntityManager`.
> 3. Hibernate's `SessionImpl` registers the entity in its **First-Level Cache (Persistence Context)**.
> 4. Hibernate delays SQL generation until flush time (dirty checking), batching statements.
> 5. HikariCP leases a physical JDBC connection to send batch `INSERT`/`UPDATE` SQL to the database.

---

### 1.2 Spring Data Repository Hierarchy

```
Repository<T, ID> (Marker interface — tells Spring to generate a proxy)
    └── CrudRepository<T, ID> (Basic CRUD: save, findById, delete)
            └── ListCrudRepository<T, ID> (Spring Boot 3: returns List<T> instead of Iterable<T>)
                    └── PagingAndSortingRepository<T, ID> (findAll(Pageable), findAll(Sort))
                            └── JpaRepository<T, ID> (flush(), saveAndFlush(), deleteInBatch())
```

*Guideline:* Default to `JpaRepository<T, ID>`. If you wish to intentionally restrict client capabilities (e.g. read-only repositories where writes must be impossible), extend `Repository<T, ID>` and declare only query methods.

---

## 2. Query Formulation Strategies (6 Approaches)

Modern Spring Boot 3.x applications require different querying mechanisms based on query complexity, dynamic filtering requirements, and throughput demands.

---

### 2.1 Approach 1: Derived Query Methods

#### Scenario
- Simple, predictable queries matching 1 to 3 fields without complex subqueries or joins.

#### Use-Case
Looking up an active user by their email address: `SELECT * FROM users WHERE email = ? AND status = ?`.

#### Benefits Over Others
- **Zero Query String Writing**: Spring parses the method name tokens (`findBy`, `And`, `GreaterThan`, `OrderBy`).
- **Compile-Time Startup Validation**: If an entity field is renamed or misspelled, Spring Boot fails during bootstrap with `PropertyReferenceException`.

#### Documented Code Example

```java
package com.enterprise.repository;

import com.enterprise.domain.User;
import com.enterprise.domain.UserStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived Query: Generates SELECT u FROM User u WHERE u.email = :email
    Optional<User> findByEmail(String email);

    // Compound Query: Generates SELECT u FROM User u WHERE u.status = :status AND u.createdAt >= :afterDate
    List<User> findByStatusAndCreatedAtGreaterThanEqual(UserStatus status, LocalDateTime afterDate);

    // Existence check: Generates SELECT COUNT(u) > 0 FROM User u WHERE u.taxId = :taxId
    boolean existsByTaxId(String taxId);
}
```

---

### 2.2 Approach 2: Explicit JPQL Queries (`@Query`)

#### Scenario
- Moderately complex queries, cross-entity relationships, selective projections, or when derived method names become unreadably long.

#### Use-Case
Querying active orders belonging to customers in a specific geographical country code.

#### Benefits Over Others
- **Database-Agnostic Portability**: Queries reference Java entities (`Order`, `Customer`) and fields rather than underlying database table and column names.
- **Supports Projections**: Cleanly constructs DTOs via `new com.enterprise.dto.OrderSummaryDto(...)`.

#### Documented Code Example

```java
package com.enterprise.repository;

import com.enterprise.domain.Order;
import com.enterprise.domain.OrderStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o " +
           "JOIN o.customer c " +
           "WHERE o.status = :status AND c.address.countryCode = :country " +
           "ORDER BY o.createdAt DESC")
    List<Order> findRecentOrdersByCountry(
            @Param("status") OrderStatus status,
            @Param("country") String country);
}
```

---

### 2.3 Approach 3: Native SQL Queries (`nativeQuery = true`)

#### Scenario
- Utilizing database-specific proprietary SQL features (PostgreSQL JSONB operators, Common Table Expressions `WITH RECURSIVE`, MySQL window functions, or spatial PostGIS queries) that JPQL cannot express.

#### Use-Case
Querying orders storing unstructured audit payloads in a PostgreSQL `jsonb` column using the `->>` json navigation operator.

#### Benefits Over Others
- **Direct Database Optimization**: Zero JPQL abstraction layer overhead; utilizes exact DB execution plans and specific vendor indexes.

#### Limitations
- Tied strictly to the target database dialect; breaks if switching from PostgreSQL to Oracle/MySQL.

#### Documented Code Example

```java
package com.enterprise.repository;

import com.enterprise.domain.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface OrderNativeRepository extends JpaRepository<Order, Long> {

    // PostgreSQL specific native query utilizing jsonb containment and window functions
    @Query(value = "SELECT * FROM orders o " +
                   "WHERE o.metadata ->> 'channel' = :channel " +
                   "AND o.created_at >= NOW() - INTERVAL '30 days'",
           nativeQuery = true)
    List<Order> findRecentOrdersByChannelNative(@Param("channel") String channel);
}
```

---

### 2.4 Approach 4: Spring Data JPA Specifications (Criteria API)

#### Scenario
- Dynamic search screens with 5 to 15 optional filter criteria (e.g. search by name, date range, min price, max price, category, tags) where building individual query methods is impossible.

#### Use-Case
An administrative product search catalog where users can filter by any permutation of 8 different optional form fields.

#### Benefits Over Others
- **Composable and Reusable Predicates**: Specifications can be chained programmatically using `.and()`, `.or()`, and `.not()`.
- Built natively into Spring Data JPA via `JpaSpecificationExecutor<T>`.

#### Documented Code Example

```java
package com.enterprise.specification;

import com.enterprise.domain.Product;
import com.enterprise.domain.ProductStatus;
import org.springframework.data.jpa.domain.Specification;

import java.math.BigDecimal;

public class ProductSpecifications {

    public static Specification<Product> hasStatus(ProductStatus status) {
        return (root, query, cb) -> 
            status == null ? cb.conjunction() : cb.equal(root.get("status"), status);
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min != null && max != null) return cb.between(root.get("price"), min, max);
            if (min != null) return cb.greaterThanOrEqualTo(root.get("price"), min);
            if (max != null) return cb.lessThanOrEqualTo(root.get("price"), max);
            return cb.conjunction();
        };
    }

    public static Specification<Product> nameContains(String keyword) {
        return (root, query, cb) -> 
            keyword == null || keyword.isBlank() 
                ? cb.conjunction() 
                : cb.like(cb.lower(root.get("name")), "%" + keyword.toLowerCase() + "%");
    }
}
```

Service execution:
```java
@Service
public class ProductSearchService {
    private final ProductRepository productRepo;

    public ProductSearchService(ProductRepository productRepo) {
        this.productRepo = productRepo;
    }

    public Page<Product> search(ProductSearchFilter filter, Pageable pageable) {
        Specification<Product> spec = Specification.where(ProductSpecifications.hasStatus(filter.status()))
                .and(ProductSpecifications.priceBetween(filter.minPrice(), filter.maxPrice()))
                .and(ProductSpecifications.nameContains(filter.keyword()));

        return productRepo.findAll(spec, pageable);
    }
}
```

---

### 2.5 Approach 5: QueryDSL (Type-Safe Fluent Queries)

#### Scenario
- Complex enterprise domain query graphs where Criteria API code is too verbose, brittle, and unreadable.

#### Use-Case
Enterprise reporting dashboards joining 5+ entities with dynamic aggregations (`SUM`, `AVG`), group-by clauses, and having filters.

#### Benefits Over Others
- **100% Type-Safe**: Generates `Q` metadata classes (e.g. `QOrder.order`, `QCustomer.customer`). Compile errors occur immediately if a field name changes.
- **Fluent, Readable DSL**: Reads almost identically to natural SQL.

#### Documented Code Example

```java
package com.enterprise.repository.custom;

import com.enterprise.domain.QCustomer;
import com.enterprise.domain.QOrder;
import com.enterprise.dto.CustomerOrderStatsDto;
import com.querydsl.core.types.Projections;
import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public class OrderAnalyticsQueryDslRepository {

    private final JPAQueryFactory queryFactory;

    public OrderAnalyticsQueryDslRepository(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }

    public List<CustomerOrderStatsDto> calculateCustomerStats(double minTotalSpend) {
        QOrder order = QOrder.order;
        QCustomer customer = QCustomer.customer;

        return queryFactory
                .select(Projections.constructor(CustomerOrderStatsDto.class,
                        customer.id,
                        customer.email,
                        order.count(),
                        order.totalAmount.sum()))
                .from(order)
                .join(order.customer, customer)
                .groupBy(customer.id, customer.email)
                .having(order.totalAmount.sum().goe(minTotalSpend))
                .orderBy(order.totalAmount.sum().desc())
                .fetch();
    }
}
```

---

### 2.6 Approach 6: Fluent `JdbcClient` (Spring Boot 3.2+ High Performance)

#### Scenario
- Ultra-high-throughput bulk ingestion, analytical reporting queries, or batch jobs where Hibernate's entity state tracking and dirty checking overhead is a performance bottleneck.

#### Use-Case
Querying 100,000 telemetry records to generate a CSV export in sub-second response times.

#### Benefits Over Others
- **Zero ORM Overhead**: Bypasses Hibernate First-Level Cache, Proxy generation, and dirty checking snapshots entirely.
- **Modern Fluent API**: Introduced in Spring Boot 3.2 / Spring 6.1 as the modern replacement for verbose `JdbcTemplate`.

#### Documented Code Example

```java
package com.enterprise.analytics;

import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.stereotype.Repository;

import java.time.LocalDate;
import java.util.List;

public record DailySummaryRecord(LocalDate summaryDate, long totalOrders, double totalRevenue) {}

@Repository
public class FastAnalyticsJdbcRepository {

    private final JdbcClient jdbcClient;

    public FastAnalyticsJdbcRepository(JdbcClient jdbcClient) {
        this.jdbcClient = jdbcClient;
    }

    public List<DailySummaryRecord> fetchDailySummaries(LocalDate since) {
        String sql = """
            SELECT order_date as summaryDate, 
                   COUNT(*) as totalOrders, 
                   SUM(amount) as totalRevenue 
            FROM orders 
            WHERE order_date >= :since 
            GROUP BY order_date 
            ORDER BY order_date ASC
            """;

        // Modern fluent mapping directly into Java record!
        return jdbcClient.sql(sql)
                .param("since", since)
                .query(DailySummaryRecord.class)
                .list();
    }
}
```

---

### 2.7 Query Formulation Comparison Matrix

| Approach | Type Safety | Compile-Time Checking | Dynamic Query Ergonomics | Performance Overhead | Best Use-Case |
|---|---|---|---|---|---|
| **Derived Methods** | ✅ High | ✅ At Bootstrap | ❌ Impossible | Low (ORM cached) | 1-3 field lookups (`findByEmail`) |
| **JPQL (`@Query`)** | ⚠️ String-based | ⚠️ At Bootstrap | ❌ Hardcoded strings | Low (ORM cached) | Static joins, DTO projections |
| **Native SQL** | ❌ None | ❌ Runtime only | ❌ Hardcoded strings | Low (Direct DB) | PostgreSQL JSONB, window functions |
| **Specifications** | ⚠️ Verbose | ⚠️ Semi-safe | 🌟 **High (Composable)**| Moderate | Optional search filters |
| **QueryDSL** | 🌟 **Full (Q-Classes)**| 🌟 **At Compile Time**| 🌟 **High (Fluent)** | Moderate | Complex enterprise aggregations |
| **Fluent `JdbcClient`**| ⚠️ String SQL | ❌ Runtime only | ⚠️ SQL concatenation | 🌟 **Ultra-Fast (Zero ORM)** | Bulk analytics, high-throughput exports |

---

## 3. Entity Mapping & Schema Modeling

### 3.1 Core Entity Annotations & Mapping

```java
package com.enterprise.domain;

import jakarta.persistence.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(
    name = "products",
    indexes = {
        @Index(name = "idx_product_sku", columnList = "sku", unique = true),
        @Index(name = "idx_product_status_cat", columnList = "status, category_id")
    }
)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq_gen")
    @SequenceGenerator(name = "product_seq_gen", sequenceName = "product_sequence", allocationSize = 50)
    private Long id;

    @Column(name = "sku", nullable = false, unique = true, length = 64)
    private String sku;

    @Column(name = "name", nullable = false, length = 200)
    private String name;

    @Column(precision = 12, scale = 2, nullable = false)
    private BigDecimal price;

    @Enumerated(EnumType.STRING) // MANDATORY: Always persist STRING name, never integer ORDINAL!
    @Column(name = "status", nullable = false, length = 32)
    private ProductStatus status;

    @Version // Optimistic Concurrency Control
    private Long version;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

---

### 3.2 Primary Key Generation Strategies & JDBC Batching Impact

| Strategy | Mechanism | Performance & JDBC Batching Impact |
|---|---|---|
| **`IDENTITY`** | Database auto-increment column (`SERIAL`, `AUTO_INCREMENT`) | 🚫 **DISABLES JDBC BATCHING**. Hibernate must execute immediate SQL `INSERT` statements to capture the generated ID from the DB, completely disabling batch optimizations! |
| **`SEQUENCE`** | Database sequence generator (`sequenceName`, `allocationSize = 50`) | 🌟 **TOP RECOMMENDATION**. Hibernate pre-fetches 50 IDs into JVM memory in a single round-trip, enabling true high-performance JDBC batch inserts (`batch_size = 50`). |
| **`UUID`** | UUID v4 (Random) or UUID v7 (Time-Ordered) | Random UUIDs cause severe B-Tree index fragmentation and disk page splits in clustered index databases (MySQL InnoDB). Use time-ordered UUIDv7 or TSID. |
| **`TABLE`** | Simulates a sequence in a lock table | 🚫 **NEVER USE IN PRODUCTION**. Causes severe row-level lock contention on the sequence table. |

---

### 3.3 Composite Keys (`@Embeddable` & `@EmbeddedId`)

```java
@Embeddable
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;

    // equals() and hashCode() are MANDATORY for correct Hibernate First-Level Cache identity!
    @Override
    public boolean equals(Object o) { ... }
    @Override
    public int hashCode() { ... }
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;

    private int quantity;
    private BigDecimal unitPrice;
}
```

---

### 3.4 Base Entity Inheritance (`@MappedSuperclass`)

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseAuditableEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "global_seq")
    @SequenceGenerator(name = "global_seq", sequenceName = "global_id_sequence", allocationSize = 50)
    private Long id;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime lastModifiedAt;

    @Version
    private Long version;
}
```

---

## 4. Entity Relationships & Performance Gotchas

### 4.1 Owning vs Inverse Sides of Relationships

In relational databases, foreign keys reside in a single table. In JPA:
- **Owning Side**: Contains `@JoinColumn(name = "...")`. Hibernate inspects this side to generate SQL `INSERT` and `UPDATE` statements for the foreign key.
- **Inverse Side**: Declares `mappedBy = "fieldNameOnOwningSide"`. Hibernate ignores changes made to this collection unless synchronized with the owning side!

```
Department (Inverse Side: mappedBy = "department")
    ▲
    │  (1 to Many)
    │
Employee (Owning Side: @JoinColumn(name = "department_id")) -> Controls DB Foreign Key!
```

---

### 4.2 Bidirectional `@OneToMany` & Synchronization Helpers

> [!WARNING]
> **The Synchronization Trap**: If you execute `department.getEmployees().add(employee)` but forget `employee.setDepartment(department)`, Hibernate will **persist `NULL` into the foreign key column**! Always encapsulate both operations in helper methods.

```java
@Entity
public class Department {
    @Id private Long id;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();

    // Synchronization helper methods guarantee consistency
    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this); // Updates the owning side!
    }

    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
}
```

---

### 4.3 Cascade Types & The `@ManyToMany` REMOVE Hazard

* `CascadeType.PERSIST`: Cascades entity save.
* `CascadeType.MERGE`: Cascades entity update.
* `CascadeType.REMOVE`: Cascades entity deletion.

> [!CAUTION]
> **NEVER use `CascadeType.REMOVE` or `CascadeType.ALL` on `@ManyToMany` relationships!** If an `Author` is linked to multiple `Books`, deleting one author will trigger cascading deletes on all associated books, which cascades and deletes all other authors who co-wrote those books, wiping out entire database tables!

---

### 4.4 FetchType: LAZY vs EAGER Defaults

| Relationship | JPA Default Fetch Type | Recommended Enterprise Best Practice |
|---|---|---|
| **`@ManyToOne`** | ⚠️ **`EAGER`** | 🌟 **Override to `LAZY` immediately** |
| **`@OneToOne`** | ⚠️ **`EAGER`** | 🌟 **Override to `LAZY` immediately** |
| **`@OneToMany`** | `LAZY` | `LAZY` |
| **`@ManyToMany`** | `LAZY` | `LAZY` |

> [!IMPORTANT]
> The JPA specification defaults `@ManyToOne` and `@OneToOne` to `FetchType.EAGER`. This is the single greatest cause of unexpected N+1 query storms. **Every single relationship must be declared `FetchType.LAZY`**.

---

## 5. The N+1 Select Problem & 4 Architectural Solutions

### 5.1 The Root Cause & Detection Strategies

The **N+1 Select Problem** occurs when an application executes 1 initial SQL query to retrieve N parent rows, followed by N subsequent queries to fetch a lazy-loaded child relationship for each individual row.

```
Query 1: SELECT * FROM orders LIMIT 100; (Returns 100 orders)
Query 2: SELECT * FROM customers WHERE id = ? (For Order 1)
Query 3: SELECT * FROM customers WHERE id = ? (For Order 2)
...
Query 101: SELECT * FROM customers WHERE id = ? (For Order 100)
Total Queries = 1 + 100 = 101 queries! (Crashes connection pool under high traffic)
```

Detection in `application.yml`:
```yaml
spring:
  jpa:
    show-sql: false # Do not use in production
    properties:
      hibernate:
        generate_statistics: true
logging:
  level:
    org.hibernate.stat: DEBUG
```

---

### 5.2 Solution 1: JPQL `JOIN FETCH` (Explicit Query Association)

#### Scenario
- Dedicated repository queries where you know in advance that the child association is required.

#### Mechanism
Instructs Hibernate to execute an SQL `INNER JOIN` or `LEFT JOIN`, mapping both parent and child columns in a **single SQL query**.

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Resolves N+1 in 1 single SQL query!
    @Query("SELECT o FROM Order o JOIN FETCH o.customer c JOIN FETCH o.items WHERE o.status = :status")
    List<Order> findAllWithCustomerAndItems(@Param("status") OrderStatus status);
}
```

---

### 5.3 Solution 2: Declarative `@EntityGraph`

#### Scenario
- Reusing standard Spring Data JPA query methods (`findAll()`, `findById()`) while dynamically telling Hibernate which lazy fields to fetch eagerly for specific callers.

```java
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    // Overrides default findById to eagerly fetch orders in 1 query without writing custom JPQL
    @EntityGraph(attributePaths = {"orders", "address"})
    Optional<Customer> findWithOrdersById(Long id);
}
```

---

### 5.4 Solution 3: Hibernate Batch Fetching (`@BatchSize`)

#### Scenario
- When multiple lazy collections must be loaded across nested hierarchies without causing a `MultipleBagFetchException` (Cartesian Product Problem).

#### Mechanism
Instead of executing 1 query per child, Hibernate batches IDs into an `IN (?, ?, ...)` clause, reducing N queries to `N / BatchSize`.

```yaml
# application.yml: Global batch sizing
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50
```

Or on specific entity collections:
```java
@Entity
public class Customer {
    @BatchSize(size = 50)
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
}
```

---

### 5.5 Solution 4: DTO Constructor Projections (Best for Read-Only Queries)

#### Scenario
- High-performance read operations (API responses) where updating the entity is not required.

#### Mechanism
Selects only the exact columns required directly into a Java record DTO, bypassing Hibernate entity management, proxies, and dirty checking snapshots entirely.

```java
public record OrderSummaryDto(Long orderId, String customerName, BigDecimal amount) {}

@Repository
public interface OrderProjectionRepository extends JpaRepository<Order, Long> {

    @Query("SELECT new com.enterprise.dto.OrderSummaryDto(o.id, c.name, o.totalAmount) " +
           "FROM Order o JOIN o.customer c WHERE o.status = :status")
    List<OrderSummaryDto> findSummaries(@Param("status") OrderStatus status);
}
```

---

### 5.6 N+1 Resolution Comparison Matrix

| Solution | Single SQL Query? | Memory Footprint | Handles Multiple `@OneToMany` Lists? | Best For |
|---|---|---|---|---|
| **`JOIN FETCH`** | 🌟 **Yes (1 Query)** | High (Cartesian duplicate rows) | 🚫 Throws `MultipleBagFetchException` | Single parent-to-child queries |
| **`@EntityGraph`** | 🌟 **Yes (1 Query)** | High (Cartesian duplicate rows) | 🚫 Same cartesian limit | Dynamic method-level overrides |
| **`@BatchSize` (50)**| ⚠️ `1 + N/50` Queries | 🌟 **Low (No cartesian product)** | 🌟 **Yes (Multiple bags allowed)** | Deep hierarchical graphs & collections |
| **DTO Projections** | 🌟 **Yes (1 Query)** | 🌟 **Lowest (Zero entity proxies)** | ⚠️ Requires flattening | 🌟 **Read-only REST API queries** |

---

## 6. Transaction Management Deep Dive (3 Approaches)

### 6.1 The Spring Proxy Mechanism

When `@Transactional` is invoked from an external bean:
1. Spring's `TransactionInterceptor` intercepts the call.
2. Acquires a JDBC connection from `HikariDataSource`.
3. Sets `connection.setAutoCommit(false)`.
4. Binds the connection to `TransactionSynchronizationManager` (ThreadLocal).
5. Executes the target business method.
6. If an unchecked exception occurs -> `connection.rollback()`.
7. If successful -> `connection.commit()`.

---

### 6.2 Approach 1: Declarative `@Transactional` Boundaries

```java
@Service
@Transactional(readOnly = true) // Class level: optimizes all read operations
public class BankAccountService {

    private final AccountRepository accountRepo;

    public BankAccountService(AccountRepository accountRepo) {
        this.accountRepo = accountRepo;
    }

    public AccountDto getAccount(Long id) {
        return accountRepo.findById(id).map(AccountDto::from).orElseThrow();
    }

    @Transactional(
        isolation = Isolation.READ_COMMITTED,
        propagation = Propagation.REQUIRED,
        rollbackFor = Exception.class, // Roll back on checked exceptions
        timeout = 10 // Timeout in seconds
    )
    public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepo.findById(fromId).orElseThrow();
        Account to = accountRepo.findById(toId).orElseThrow();
        from.debit(amount);
        to.credit(amount);
        // Hibernate dirty checking automatically triggers SQL UPDATEs at commit!
    }
}
```

---

### 6.3 Approach 2: Programmatic `TransactionTemplate`

#### Scenario
- Long-running workflows where only a tiny sub-segment requires database locking (e.g. Call third-party API for 10 seconds -> Save to DB for 50ms -> Send email).
- Wrapping the entire 10-second method in `@Transactional` holds a HikariCP database connection hostage for 10 seconds, quickly starving the pool!

```java
@Service
public class OrderCheckoutService {

    private final TransactionTemplate transactionTemplate;
    private final PaymentClient paymentClient;
    private final OrderRepository orderRepository;

    public OrderCheckoutService(TransactionTemplate transactionTemplate,
                                PaymentClient paymentClient,
                                OrderRepository orderRepository) {
        this.transactionTemplate = transactionTemplate;
        this.paymentClient = paymentClient;
        this.orderRepository = orderRepository;
    }

    public OrderResult checkout(OrderRequest request) {
        // 1. NON-TRANSACTIONAL: Call external payment gateway (takes 2-5 seconds, holds NO DB connection)
        PaymentResponse payment = paymentClient.authorize(request.cardToken(), request.amount());

        // 2. TRANSACTIONAL: Fast 20ms database transaction
        return transactionTemplate.execute(status -> {
            try {
                Order order = new Order(request, payment.txId());
                Order saved = orderRepository.save(order);
                return new OrderResult(saved.getId(), "SUCCESS");
            } catch (Exception ex) {
                status.setRollbackOnly();
                throw ex;
            }
        });
    }
}
```

---

### 6.4 Approach 3: Read-Only Optimization (`readOnly = true`)

```java
@Transactional(readOnly = true)
public User getUser(Long id) { ... }
```
**Technical Under-the-Hood Impact:**
1. Sets JDBC connection `connection.setReadOnly(true)` (enables database replica read routing).
2. **Disables Hibernate Dirty Checking**: Hibernate skips snapshot memory allocation and dirty inspection at flush time, reducing memory and CPU usage by up to 50%!

---

### 6.5 Transaction Propagation & Isolation Levels

* **`REQUIRED`** (Default): Join existing transaction; create one if absent.
* **`REQUIRES_NEW`**: Suspend current transaction; execute in an independent, newly committed transaction (e.g. audit logging that must survive outer rollbacks).
* **`MANDATORY`**: Must run inside an existing transaction; throws exception otherwise.

---

## 7. Lazy Loading Traps & The OSIV Anti-Pattern

### 7.1 `LazyInitializationException` Mechanics
Occurs when accessing a lazy proxy property outside of an active Hibernate Session (e.g. accessing `order.getCustomer().getName()` in the Controller layer after the `@Transactional` service returned).

---

### 7.2 Open Session In View (OSIV) Production Hazards

Spring Boot defaults to `spring.jpa.open-in-view=true`.
* **Hazard**: OSIV keeps the Hibernate Session and JDBC connection open across the entire HTTP response lifecycle (including JSON serialization).
* **Catastrophe**: If Jackson serializes a lazy collection, it executes N+1 SQL queries silently in the presentation layer. Under heavy traffic, all Hikari connections become exhausted by slow client HTTP downloads.

**Mandatory Enterprise Standard**:
```yaml
spring:
  jpa:
    open-in-view: false # Always set to false in production!
```
*Resolution*: Always fetch required child associations within the `@Transactional` service using `JOIN FETCH`, `@EntityGraph`, or map to DTOs before exiting the service boundary.

---

## 8. Database Concurrency & Locking Strategies (4 Approaches)

---

### 8.1 Approach 1: Optimistic Locking with `@Version`

#### Scenario
- Low to moderate concurrency (e.g. editing user profiles, updating product catalog descriptions) where collisions are rare.

```java
@Entity
public class Product {
    @Id private Long id;
    private String name;

    @Version // Automatically checked by Hibernate during UPDATE
    private Long version;
}
```
SQL Generated:
`UPDATE products SET name = ?, version = 2 WHERE id = ? AND version = 1;`
If another transaction updated the row first, 0 rows are updated, and Hibernate throws `OptimisticLockingFailureException`.

---

### 8.2 Approach 2: Pessimistic Write Lock (`PESSIMISTIC_WRITE`)

#### Scenario
- High contention where conflicts are guaranteed (e.g. flash-sale ticket booking, bank balance withdrawals).

```java
@Repository
public interface TicketRepository extends JpaRepository<Ticket, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT t FROM Ticket t WHERE t.id = :id")
    Optional<Ticket> findByIdForUpdate(@Param("id") Long id);
}
```
SQL Generated:
`SELECT * FROM tickets WHERE id = ? FOR UPDATE;`
Other concurrent transactions block at the database level until the holding transaction commits.

---

### 8.3 Approach 3: Pessimistic Read Lock (`PESSIMISTIC_READ`)

#### Scenario
- Data must be read and guaranteed not to be modified by other transactions until our transaction finishes (shared lock).

---

### 8.4 Approach 4: Database Unique Idempotency Keys

#### Scenario
- Preventing duplicate payment processing or duplicate webhook processing.
- A unique index on `idempotency_key` guarantees that concurrent duplicate requests fail fast with `DataIntegrityViolationException`.

---

### 8.5 Locking Strategies Comparison Matrix

| Strategy | Contention Level | Database Locks Held | Retry Required? | Deadlock Risk |
|---|---|---|---|---|
| **Optimistic (`@Version`)** | Low to Moderate | ❌ None | 🌟 **Yes (Catch & Retry)** | ❌ Zero |
| **Pessimistic Write (`FOR UPDATE`)** | 🌟 **Very High** | 🌟 **Row-Level Lock** | ❌ No (Threads wait) | ⚠️ High (lock ordering crucial) |
| **Pessimistic Read (Shared Lock)** | Moderate | Shared Row Lock | ❌ No | ⚠️ Moderate |
| **Unique Constraints** | Idempotency boundaries| Unique Index Lock | ❌ No (Throws 409) | ❌ Zero |

---

## 9. Database Schema Migrations (Flyway vs Liquibase)

### 9.1 Flyway Setup & Zero-Downtime Migration Patterns

```
src/main/resources/db/migration/
├── V1__init_schema.sql
├── V2__add_tax_id_to_customers.sql
└── V3__create_orders_table.sql
```

#### Zero-Downtime Expand/Contract Migration Pattern:
1. **Release 1 (Expand)**: Add new column `customer_name` as nullable. Application dual-writes to both `name` and `customer_name`.
2. **Release 2 (Migrate)**: Run background SQL migration backfilling historical data from `name` to `customer_name`.
3. **Release 3 (Switch)**: Application reads and writes exclusively to `customer_name`.
4. **Release 4 (Contract)**: Drop old column `name`.

---

### 9.2 Liquibase Overview
* Uses XML, YAML, or SQL changelogs. Better suited than Flyway for multi-database vendor portability (e.g. supporting both Oracle and PostgreSQL in the same enterprise product).

---

## 10. Connection Pooling with HikariCP

### 10.1 Key Configuration Parameters

```yaml
spring:
  datasource:
    hikari:
      pool-name: EnterpriseHikariPool
      maximum-pool-size: 20
      minimum-idle: 10
      connection-timeout: 30000        # 30 seconds wait for connection
      idle-timeout: 600000             # 10 minutes idle timeout
      max-lifetime: 1800000            # 30 minutes max connection lifetime
      leak-detection-threshold: 2000   # Warns if connection held > 2 seconds!
```

---

### 10.2 Pool Sizing Formula & Saturation Protection

PostgreSQL / HikariCP Official Pool Sizing Formula:
$$\text{Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindle Count}$$

For an 8-core database server with SSD storage:
$$\text{Pool Size} = (8 \times 2) + 1 = 17 \text{ connections}$$

> [!TIP]
> Setting `maximum-pool-size = 200` is a classic anti-pattern. Over-allocating connections causes disk trashing and severe CPU context-switching on the database server. A small, tightly tuned pool of 15 to 25 connections consistently yields the highest throughput.

---

## 11. Entity Auditing Strategies (3 Approaches)

1. **Spring Data JPA Auditing (`@EnableJpaAuditing`)**: Populates `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy` via `AuditorAware`.
2. **Hibernate Built-In Annotations**: `@CreationTimestamp` and `@UpdateTimestamp` generate timestamps directly within Hibernate.
3. **Hibernate Envers (`@Audited`)**: Maintains complete historical audit mirror tables (e.g. `orders_aud`) recording full change deltas and user revisions across time.

---

## 12. ⚠️ High-Frequency Interview Traps & JPA Gotchas

### Trap 1: `GenerationType.IDENTITY` Disables JDBC Batch Inserts
- Hibernate must immediately execute `INSERT` statements to obtain the generated primary key from the database, completely preventing JDBC batching. Use `GenerationType.SEQUENCE` with `allocationSize = 50`.

### Trap 2: Modifying Attached Entities Without Calling `save()`
- In an active `@Transactional` method, any changes made to an entity are automatically persisted to the database via **Hibernate Dirty Checking** at flush time. Calling `repository.save()` is completely redundant!

### Trap 3: Self-Invocation Bypasses `@Transactional`
- Calling a `@Transactional` method from another method in the same class bypasses the CGLIB proxy, meaning **no transaction is created**.

### Trap 4: Equals & HashCode in JPA Entities
- Never use autogenerated Lombok `@Data` or `@EqualsAndHashCode` on JPA entities. If an entity uses autogenerated IDs, `id` is `null` before persistence, corrupting `Set` collections. Implement `equals()` and `hashCode()` using a stable business key or check object identity.

---

## 13. 📝 Architecture Principle Checklist

- [ ] Set `spring.jpa.open-in-view = false` in all production configurations.
- [ ] Declare `FetchType.LAZY` on all `@ManyToOne` and `@OneToOne` associations.
- [ ] Use `GenerationType.SEQUENCE` with `allocationSize = 50` for high-throughput batching.
- [ ] Eliminate N+1 queries using `JOIN FETCH`, `@EntityGraph`, or DTO projections.
- [ ] Protect high-contention inventory updates using `@Version` or `PESSIMISTIC_WRITE`.
- [ ] Keep HikariCP pool sizes small ($(\text{CPU} \times 2) + 1$).
- [ ] Manage long-running operations outside database transactions using `TransactionTemplate`.

---

## 14. ❓ Practice Questions & Answers

### Q1: Why does `GenerationType.IDENTITY` disable JDBC batching in Hibernate?
**Answer**: With `IDENTITY`, the primary key is generated by the database upon row insertion (e.g. auto-increment). To satisfy the contract of the Persistence Context—where every managed entity must possess a non-null identifier—Hibernate is forced to execute the SQL `INSERT` statement immediately rather than queueing it. This immediate execution bypasses JDBC statement batching. In contrast, `SEQUENCE` allows Hibernate to allocate a block of IDs in advance, queue all statements, and execute them in a single batch.

### Q2: What is the exact purpose of `readOnly = true` on `@Transactional`?
**Answer**: `readOnly = true` optimizes data access in two major ways:
1. It passes a hint to the JDBC driver/connection (`connection.setReadOnly(true)`), allowing routing to read-replicas.
2. It instructs Hibernate to disable **Dirty Checking**. Hibernate skips taking in-memory snapshot copies of entity state upon retrieval and skips comparison loops during transaction flush, resulting in significant CPU and heap memory savings.

### Q3: Why is Open Session in View (OSIV) considered an anti-pattern in production microservices?
**Answer**: OSIV keeps the database connection and Hibernate Session open throughout the entire web request, including controller execution and JSON view rendering. This conceals architectural flaws by allowing lazy loading in presentation layers, triggering silent N+1 query storms. Furthermore, slow client downloads hold database connections open for seconds, leading to rapid HikariCP connection pool starvation.

---

[🏠 Course Index](./00-course-index.md) | [← 04 — Exception Handling](./04-exception-handling.md) | **Module 05** | [Next: 06 — Concurrency →](./06-concurrency.md)
