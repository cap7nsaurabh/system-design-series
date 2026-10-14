# Module 5: Database & JPA/Hibernate Deep Dive in Spring Boot

Working with databases in Spring Boot is one of the most critical skills for a backend engineer. While Spring Data JPA abstracts away much of the boilerplate, not understanding the underlying mechanics of Hibernate and JPA can lead to disastrous performance issues in production, such as the infamous N+1 problem, memory leaks, and transaction anomalies.

This extremely detailed deep-dive module covers the entire spectrum of Spring Boot database interactions, from mapping and querying to performance optimization, transaction management, and connection pooling.

---

## 1. Spring Data JPA Basics

### 1.1 JPA vs Hibernate vs Spring Data JPA

Before diving into code, it is absolutely essential to understand the layer cake of data access in Spring Boot. Many candidates confuse JPA with Hibernate.

1. **JDBC (Java Database Connectivity)**: The lowest level. The standard Java API for connecting to relational databases. Very verbose.
2. **JPA (Jakarta Persistence API, formerly Java Persistence API)**: A **specification** (a set of interfaces and rules) for Object-Relational Mapping (ORM) in Java. It defines annotations like `@Entity`, `@Id`, and interfaces like `EntityManager`. *It does not contain implementation code.*
3. **Hibernate**: The most popular **implementation** (provider) of the JPA specification. It contains the actual code that translates your Java objects into SQL queries and JDBC calls. Hibernate also has its own proprietary features beyond the JPA spec.
4. **Spring Data JPA**: An abstraction layer *on top of* JPA. It removes the boilerplate of writing `EntityManager` code and DAO implementations by generating them dynamically at runtime based on repository interfaces.

> [!IMPORTANT] 
> Spring Data JPA uses Hibernate under the hood by default. When you call `repository.save()`, Spring Data JPA delegates to an `EntityManager`, which is implemented by Hibernate, which generates SQL and executes it via JDBC.

### 1.2 Repository Hierarchy

Spring Data provides several repository interfaces, each building on the previous one:

- **`Repository<T, ID>`**: The marker interface. Doesn't provide any methods but tells Spring to create a proxy.
- **`CrudRepository<T, ID>`**: Provides basic CRUD operations (`save`, `findById`, `delete`, etc.).
- **`PagingAndSortingRepository<T, ID>`**: Extends `CrudRepository` and adds methods that take `Pageable` and `Sort` arguments.
- **`JpaRepository<T, ID>`**: Extends `PagingAndSortingRepository` and adds JPA-specific methods like `flush()`, `saveAllAndFlush()`, and batch deletes in a single query.

**Which one to use?**
Use `JpaRepository` for most cases, but if you want to restrict the API (e.g., you don't want clients to be able to call `flush()`), you can extend `CrudRepository` or even just `Repository` and selectively declare the methods you want.

### 1.3 Derived Query Methods

Spring Data JPA can generate SQL queries by parsing method names.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Generates: SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // Generates: SELECT * FROM users WHERE last_name = ? AND age > ?
    List<User> findByLastNameAndAgeGreaterThan(String lastName, int age);

    // Generates: SELECT * FROM users WHERE first_name LIKE ? ORDER BY created_at DESC
    List<User> findByFirstNameStartingWithOrderByCreatedAtDesc(String prefix);
}
```

**How it works:**
At application startup, Spring creates a proxy for the interface. It parses the method names using a query parser tree, validates the property names against the `User` entity, and pre-compiles JPQL queries. If a property name is wrong, the application will fail to start.

### 1.4 @Query Annotation: JPQL vs Native

When method names become too long or the query is too complex, use `@Query`.

**JPQL (Java Persistence Query Language):**
Queries against the *entity classes and properties*, not database tables and columns. This makes it database-agnostic.

```java
@Query("SELECT u FROM User u WHERE u.status = :status AND u.department.name = :deptName")
List<User> findActiveUsersInDepartment(@Param("status") UserStatus status, @Param("deptName") String deptName);
```

**Native Queries:**
Queries against the actual *database tables*. Ties you to a specific database dialect but allows you to use DB-specific features (e.g., PostgreSQL CTEs or window functions).

```java
@Query(value = "SELECT * FROM users u WHERE u.created_at >= NOW() - INTERVAL '30 days'", nativeQuery = true)
List<User> findRecentUsers();
```

---

## 2. Entity Mapping Deep Dive

### 2.1 Basic Mapping Annotations

```java
@Entity
@Table(name = "employees", indexes = {
    @Index(name = "idx_emp_email", columnList = "email", unique = true)
})
public class Employee {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name", nullable = false, length = 50)
    private String firstName;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING) // ALWAYS use STRING, never ORDINAL!
    private EmployeeStatus status;
}
```

> [!CAUTION]
> Never use `@Enumerated(EnumType.ORDINAL)` (the default). It saves the enum's integer index to the database. If someone adds a new enum value in the middle of the Java enum, all existing database records will suddenly point to the wrong values!

### 2.2 Primary Key Strategies

The `@GeneratedValue(strategy = ...)` defines how IDs are generated.

| Strategy | Description | Performance / Implications |
| :--- | :--- | :--- |
| **AUTO** | Default. Hibernate chooses based on the DB dialect. | Usually picks SEQUENCE for PostgreSQL/Oracle, and TABLE or IDENTITY for MySQL. Can be unpredictable. |
| **IDENTITY** | Relies on the database auto-increment column (e.g., MySQL `AUTO_INCREMENT`). | **Disables JDBC batch inserts** in Hibernate! Hibernate must execute `INSERT` immediately to fetch the generated ID, breaking batching. |
| **SEQUENCE** | Uses a database sequence. Hibernate fetches the next value before inserting. | Highly recommended for PostgreSQL/Oracle. Supports batch inserts. Use with `@SequenceGenerator` to define allocation size (fetch multiple IDs at once to reduce roundtrips). |
| **TABLE** | Simulates a sequence using a dedicated database table. | **Terrible performance.** Requires row-level locks on the sequence table. Avoid at all costs. |
| **UUID** | Not a GenerationType enum, but you can generate UUIDs. | Random UUIDs cause massive index fragmentation (page splits) in B-Tree clustered indexes (like InnoDB). If using UUIDs, use time-sorted UUIDs (TSID) or `uuid2`. |

**Best Practice for Batching with PostgreSQL:**
```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
@SequenceGenerator(name = "emp_seq", sequenceName = "employee_sequence", allocationSize = 50)
private Long id;
```
*(Hibernate will fetch 50 IDs in one query, then assign them in memory, allowing true batch inserts).*

### 2.3 Composite Keys (@Embeddable / @EmbeddedId)

When an entity has a multi-column primary key.

```java
@Embeddable
public class OrderItemId implements Serializable { // Must implement Serializable
    private Long orderId;
    private Long productId;
    
    // equals() and hashCode() are MANDATORY
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;

    private int quantity;
}
```

### 2.4 @MappedSuperclass

If you want common fields (like `createdAt`, `updatedAt`, `id`) across multiple entities, use `@MappedSuperclass`.

```java
@MappedSuperclass
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreationTimestamp
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

@Entity
public class Product extends BaseEntity {
    private String name;
}
```
*(No `BaseEntity` table is created. `Product` will have `id`, `createdAt`, `updatedAt`, and `name` columns).*

### 2.5 @Version (Optimistic Locking)

Adds a version column to prevent lost updates in concurrent environments.

```java
@Version
private Integer version;
```
When two threads read the same entity (Version=1) and attempt to update it, the first one succeeds (Version becomes 2). The second one will issue `UPDATE ... WHERE id=? AND version=1`. This will affect 0 rows, and Hibernate will throw an `ObjectOptimisticLockingFailureException`.

---

## 3. Relationships Deep Dive

### 3.1 Owning Side vs Inverse Side

In a bidirectional relationship, one side must "own" the relationship. The owning side is the one that contains the foreign key in the database table. The inverse side must use `mappedBy`.

**Rule of Thumb:**
- In `@OneToOne`, either can own, but usually the one with the FK column.
- In `@OneToMany` / `@ManyToOne`, the `@ManyToOne` side is **always** the owning side.
- In `@ManyToMany`, pick one to be the owner (usually the more "active" entity).

### 3.2 Bidirectional One-to-Many

```java
@Entity
public class Department {
    @Id private Long id;

    // INVERSE SIDE (Does NOT control the foreign key)
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();
    
    // Helper methods for bidirectional consistency
    public void addEmployee(Employee emp) {
        employees.add(emp);
        emp.setDepartment(this);
    }
    public void removeEmployee(Employee emp) {
        employees.remove(emp);
        emp.setDepartment(null);
    }
}

@Entity
public class Employee {
    @Id private Long id;

    // OWNING SIDE (Controls the department_id foreign key)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
}
```

> [!WARNING]
> **Interview Trap:** If you add an employee to the department's list `dept.getEmployees().add(emp)` but DO NOT set the department on the employee `emp.setDepartment(dept)`, Hibernate will **not** save the foreign key! The owning side (`Employee.department`) is null, so the database FK will be null. Always provide helper methods.

### 3.3 Cascade Types

Cascading propagates state transitions from a parent entity to children.

- `PERSIST`: When parent is saved, save children.
- `MERGE`: When parent is merged, merge children.
- `REMOVE`: When parent is deleted, delete children.
- `REFRESH`: When parent is refreshed, refresh children.
- `DETACH`: When parent is detached, detach children.
- `ALL`: All of the above.

> [!CAUTION]
> **CascadeType.REMOVE on @ManyToMany is a disaster waiting to happen.**
> If you have a `User` @ManyToMany `Role`, and `User` cascades REMOVE, deleting a user will delete the `Role` from the database, effectively breaking it for all other users! Never cascade REMOVE on ManyToMany.

### 3.4 FetchType: LAZY vs EAGER

- **EAGER**: When the entity is fetched, the related entity/collection is fetched immediately (usually via an outer join or a secondary select).
- **LAZY**: The related entity/collection is NOT fetched. Instead, Hibernate puts a **Proxy** object in its place. The actual database query to fetch the related data only happens when you access a property of that proxy (e.g., `employee.getDepartment().getName()`).

> [!IMPORTANT]
> **Default Fetch Types:**
> - `@OneToOne`, `@ManyToOne` default to **EAGER**.
> - `@OneToMany`, `@ManyToMany` default to **LAZY**.
> 
> **Best Practice:** ALWAYS set all relationships to `FetchType.LAZY` (even ManyToOne). EAGER loading is the root cause of 90% of JPA performance issues because it pulls data you don't need and forces massive cartesian products or N+1 queries.


---

## 4. The N+1 Problem (Critical Interview Topic)

### 4.1 What is it?
The N+1 select problem occurs when your application executes 1 query to fetch N parent entities, and then executes N additional queries to fetch a lazy-loaded relationship for each of those parents.

**Example Scenario:**
You have a list of 100 `Orders`, and each has a lazy-loaded `Customer`.
You loop through the orders and print the customer name.

```java
// Query 1: SELECT * FROM orders;
List<Order> orders = orderRepository.findAll();

for (Order order : orders) {
    // Queries 2 through 101: SELECT * FROM customers WHERE id = ?
    System.out.println(order.getCustomer().getName());
}
```
**Result:** 101 queries executed to fetch 100 orders and their customers. If you have 10,000 orders, you crash the database.

### 4.2 How to Detect It
1. Enable SQL logging: `spring.jpa.show-sql=true`
2. Use **p6spy** or **datasource-proxy** to log exact queries and execution times.
3. Enable Hibernate Statistics: 
   ```properties
   spring.jpa.properties.hibernate.generate_statistics=true
   logging.level.org.hibernate.stat=DEBUG
   ```

### 4.3 Solutions

#### Solution 1: JOIN FETCH (JPQL)
The most common solution. Tells Hibernate to do an `INNER JOIN` or `LEFT JOIN` and populate the entities in a single query.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o JOIN FETCH o.customer")
    List<Order> findAllWithCustomer();
}
```

#### Solution 2: @EntityGraph
A declarative way to specify which lazy fields should be eagerly fetched for a specific query.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph(attributePaths = {"customer"})
    List<Order> findAll(); // Overrides default findAll to fetch customer
}
```

#### Solution 3: @BatchSize (Hibernate Specific)
Doesn't solve the extra queries entirely, but reduces N queries to N/BatchSize queries. If you have 100 orders and batch size is 50, it will do 1 initial query + 2 batch queries using `IN (id1, id2... id50)`.

```java
@Entity
@BatchSize(size = 50)
public class Customer { ... }
```
Alternatively, configure it globally in `application.yml`:
```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size: 50
```

> [!CAUTION]
> **Multiple Bag Fetch Exception**
> If you try to `JOIN FETCH` two or more `@OneToMany` `List` collections at the same time, Hibernate will throw a `MultipleBagFetchException` because it creates a massive cartesian product.
> *Solution*: Change `List` to `Set` (still causes cartesian product in memory, bad performance), or use `@BatchSize` globally, or execute two separate queries.

---

## 5. Transactions Deep Dive

Spring Boot abstracts transaction management using the `@Transactional` annotation.

### 5.1 The Proxy Mechanism (How it works)
When you annotate a class or method with `@Transactional`, Spring doesn't modify your bytecode (by default). Instead, it creates a **Proxy** object that wraps your actual class.

When another class calls `myService.saveUser()`:
1. The call hits the Proxy.
2. The Proxy starts a database transaction (`connection.setAutoCommit(false)`).
3. The Proxy delegates the call to the actual `myService.saveUser()`.
4. If the method completes normally, the Proxy commits the transaction (`connection.commit()`).
5. If the method throws a **Runtime Exception (Unchecked)**, the Proxy rolls back (`connection.rollback()`).

> [!WARNING]
> **The Self-Invocation Problem**
> If a method inside `MyService` calls another `@Transactional` method *within the same class*, **the transaction will NOT start**. 
> Why? Because the internal call bypasses the Spring Proxy and calls the `this` reference directly.

### 5.2 Transaction Propagation

Defines what happens if a transactional method is called by another transactional method.

| Propagation | Behavior |
| :--- | :--- |
| **REQUIRED** (Default) | If a transaction exists, join it. If not, create a new one. |
| **REQUIRES_NEW** | Suspend the current transaction (if any) and ALWAYS create a completely new transaction. |
| **NESTED** | If a transaction exists, create a savepoint and run within it. If it fails, roll back only to the savepoint. |
| **SUPPORTS** | If a transaction exists, join it. If not, run non-transactionally. |
| **NOT_SUPPORTED** | Suspend current transaction, run non-transactionally. |
| **MANDATORY** | Must be called within a transaction; otherwise, throws exception. |
| **NEVER** | Must NOT be called within a transaction; otherwise, throws exception. |

### 5.3 Isolation Levels

Deals with concurrent transactions modifying/reading the same data.

1. **READ_UNCOMMITTED**: Transaction A can see uncommitted changes from Transaction B (Dirty Read). Rarely used.
2. **READ_COMMITTED** (Default in Postgres/SQL Server/Oracle): Prevents Dirty Reads. But if Transaction A reads row X twice, and Transaction B updates X in between, A gets two different values (Non-repeatable read).
3. **REPEATABLE_READ** (Default in MySQL): Prevents Non-repeatable reads. But Transaction A might see new rows inserted by B (Phantom read).
4. **SERIALIZABLE**: Highest level. Transactions are effectively executed sequentially. Terrible concurrency performance.

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void processOrder() { ... }
```

### 5.4 Rollback Rules

By default, `@Transactional` **only rolls back on Unchecked Exceptions** (`RuntimeException`, `Error`).
It does **NOT** roll back on Checked Exceptions (`IOException`, `SQLException`, `Exception`).

To fix this, ALWAYS specify `rollbackFor` if your method throws checked exceptions:
```java
@Transactional(rollbackFor = Exception.class)
public void doSomethingRisky() throws IOException { ... }
```

### 5.5 readOnly = true
```java
@Transactional(readOnly = true)
```
What does this actually do?
1. Sets JDBC connection to read-only (allows database engine to optimize locking).
2. **Crucially for Hibernate:** It completely bypasses dirty checking. Hibernate will not keep snapshot copies of entities in memory to compare against, saving significant memory and CPU. Use this on all `get`/`find` methods!

---

## 6. Lazy Loading Traps

### 6.1 LazyInitializationException (LIE)

The most common JPA error.
**Why it happens:**
You fetch an entity (e.g., `Order`) inside a transaction, but a relationship (`Customer`) is LAZY. You return the `Order` to the controller. The controller tries to call `order.getCustomer().getName()`. However, the transaction (and the Hibernate Session) was closed when the service method finished. The proxy cannot reach the database without a session. BOOM. `LazyInitializationException`.

### 6.2 The OSIV Anti-Pattern

By default, Spring Boot has a property: `spring.jpa.open-in-view=true`.
**Open Session In View (OSIV)** keeps the Hibernate Session open during the entire HTTP request (all the way to the view/JSON rendering phase in the controller).

**Why is it controversial/bad?**
It hides `LazyInitializationException` by silently executing N+1 queries in the Controller/View layer! Furthermore, it holds a database connection open for the entire HTTP request lifecycle, rapidly exhausting your connection pool under load.

**Best Practice:**
```yaml
# ALWAYS disable this in production
spring:
  jpa:
    open-in-view: false
```
**How to fix the resulting LIEs:**
1. Initialize the required data in the `@Transactional` service layer using `JOIN FETCH` or `@EntityGraph`.
2. Map your Entities to DTOs (Data Transfer Objects) inside the transaction boundary. Return DTOs to the controller, never Entities.


---

## 7. JPA Performance Optimization

### 7.1 Projections
Instead of fetching full heavy entities (with all columns), fetch exactly what you need.

**Interface-based Projections:**
```java
// Spring Data generates a proxy at runtime
public interface UserSummary {
    String getFirstName();
    String getEmail();
}

public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByStatus(String status);
}
```

**Class-based (DTO) Projections (Faster):**
Using JPQL constructor expressions avoids the overhead of proxy creation.
```java
public class UserDto {
    private String name;
    private String email;
    public UserDto(String name, String email) { ... }
}

@Query("SELECT new com.example.UserDto(u.firstName, u.email) FROM User u WHERE u.status = ?1")
List<UserDto> findActiveUsers(String status);
```

### 7.2 Dynamic Queries: Specifications vs QueryDSL

When you have a search screen with 10 optional filters, you can't write 1024 derived query methods.

**Option A: JPA Specifications (Built into Spring Data)**
Very verbose and hard to read, relies on the `CriteriaBuilder` API.
```java
public static Specification<User> hasStatus(String status) {
    return (root, query, cb) -> status == null ? cb.conjunction() : cb.equal(root.get("status"), status);
}
// Usage: repository.findAll(Specification.where(hasStatus("ACTIVE")).and(hasAge(25)));
```

**Option B: QueryDSL (Industry Standard for complex apps)**
Requires a plugin to generate `Q` classes, but provides a beautiful, type-safe fluent API.
```java
QUser qUser = QUser.user;
JPAQueryFactory queryFactory = new JPAQueryFactory(entityManager);

List<User> users = queryFactory.selectFrom(qUser)
    .where(
        statusEq(status),
        ageGt(age)
    ).fetch();
```

### 7.3 Batch Inserts and Updates

To insert 10,000 records, doing 10,000 separate `INSERT` statements will take forever.

**Hibernate properties required:**
```yaml
spring.jpa.properties.hibernate.jdbc.batch_size: 50
spring.jpa.properties.hibernate.order_inserts: true
spring.jpa.properties.hibernate.order_updates: true
```
> [!NOTE] 
> Remember from Section 2.2: Batch inserts will NOT work if your `@Id` generation strategy is `IDENTITY`. You must use `SEQUENCE`.

### 7.4 First and Second Level Cache

- **First-Level Cache (L1):** Enabled by default. Tied to the Hibernate Session (`EntityManager` / `@Transactional` scope). If you fetch User ID 1 twice in the same transaction, the database is queried only once.
- **Second-Level Cache (L2):** Disabled by default. Shared across all sessions/transactions. Requires a caching provider like Hazelcast, Ehcache, or Redis. Complex to configure and leads to stale data issues in distributed systems. Usually, application-level caching (e.g., Spring `@Cacheable` with Redis) is preferred over Hibernate L2 cache.

---

## 8. Optimistic vs Pessimistic Locking

When two users try to buy the last ticket concurrently, how do you prevent overselling?

### 8.1 Optimistic Locking (@Version)
Assume conflicts are rare. Allow both to read the ticket. When they try to save, check if the version changed.

```java
@Entity
public class Ticket {
    @Id private Long id;
    private int quantity;
    @Version private Integer version;
}
```
If User A and User B read quantity=1, version=0.
User A buys, updates quantity=0, version becomes 1.
User B tries to buy, Hibernate generates: `UPDATE ticket SET quantity=-1, version=2 WHERE id=1 AND version=0`.
This updates 0 rows. Hibernate throws `OptimisticLockingFailureException`. You catch it, and tell User B "Ticket sold out".

**Pros:** No database locks. Highly scalable.
**Cons:** If conflicts are high, many transactions will fail and require application-level retries.

### 8.2 Pessimistic Locking (LockModeType)
Assume conflicts will happen. Lock the database row immediately upon reading.

```java
public interface TicketRepository extends JpaRepository<Ticket, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT t FROM Ticket t WHERE t.id = :id")
    Optional<Ticket> findByIdForUpdate(@Param("id") Long id);
}
```
Generates: `SELECT * FROM ticket WHERE id = ? FOR UPDATE`.
User A reads and locks the row at the DB level. User B's thread will block (wait) until User A's transaction commits.

**Pros:** Guarantees data consistency, no retry logic needed.
**Cons:** Reduces throughput, can cause database deadlocks.

---

## 9. Database Migration (Flyway vs Liquibase)

Never rely on `spring.jpa.hibernate.ddl-auto=update` in production. It cannot handle renaming columns or complex data migrations, and it might accidentally drop data.

Use a schema migration tool. **Flyway** is the most common in Spring Boot.

### 9.1 Flyway Setup
Add dependency `flyway-core`.
Create SQL files in `src/main/resources/db/migration`.

**Naming Convention:**
`V{Version}__{Description}.sql` (Note the double underscore).
Example: `V1__init_schema.sql`, `V2__add_email_to_users.sql`

Flyway creates a table `flyway_schema_history` to track which scripts have run. On startup, it runs only the new ones.

> [!TIP]
> **Zero-Downtime Migrations:**
> If you want to rename a column `first_name` to `given_name` without downtime:
> 1. Release 1: Add new column `given_name`. App writes to both, reads from old.
> 2. Script: Backfill data from `first_name` to `given_name`.
> 3. Release 2: App reads/writes to `given_name`.
> 4. Release 3: Drop `first_name` column.
> Never rename/drop columns in a single step in a high-availability system.


---

## 10. Connection Pooling

Opening a TCP connection to a database is extremely slow and expensive. Connection pools keep a set of connections open and reuse them. **HikariCP** is the default, high-performance pool in Spring Boot.

### 10.1 Key Configuration Properties

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
      connection-timeout: 30000 # 30 seconds wait before throwing exception
      idle-timeout: 600000 # 10 mins before an idle connection is closed
      max-lifetime: 1800000 # 30 mins max life of a connection (prevents DB side drops)
```

### 10.2 Pool Sizing Formula
How big should your pool be? Too small = threads wait. Too big = database thrashes (context switching).
The industry standard formula (from PostgreSQL/HikariCP docs):
**Pool Size = ((Core Count * 2) + Effective Spindle Count)**
For a typical 4-core database server with SSDs: `(4 * 2) + 1 = 9`.
Surprise! A pool of 10-20 connections is usually enough to saturate a database. Setting `maximum-pool-size=200` often *decreases* performance.

---

## 11. Auditing

Spring Data JPA provides automatic auditing to track who created/modified a record and when.

### 11.1 Setup
Add annotations to your base entity:
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;
}
```

### 11.2 Enable Auditing
Add `@EnableJpaAuditing` to a configuration class. To populate `@CreatedBy`, provide an `AuditorAware` bean that reads the username from the Spring Security context.

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class AuditConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .map(Authentication::getName);
    }
}
```

---

## ⚠️ 12. Interview Traps & Gotchas

1. **"Why is my @Transactional not rolling back on my custom exception?"**
   Because you threw a Checked Exception. `@Transactional` only rolls back on unchecked (Runtime) exceptions by default. Fix: `@Transactional(rollbackFor = MyCustomException.class)`.

2. **"Why did calling my transactional method from another method in the same class not start a transaction?"**
   The Spring AOP Proxy trap. Internal method calls bypass the proxy. Fix: Inject the service into itself (anti-pattern) or move the transactional method to another service.

3. **"What happens if you use CascadeType.REMOVE on a @ManyToMany?"**
   You delete the actual related entity, not just the link. E.g., deleting a Post removes its Tags, which deletes those tags for all other Posts.

4. **"How do you fix LazyInitializationException?"**
   Do not just turn on OSIV. Initialize the data within the transaction using `JOIN FETCH`, `@EntityGraph`, or map to a DTO before exiting the service layer.

5. **"Why aren't my Hibernate batch inserts working?"**
   You are using `@GeneratedValue(strategy = GenerationType.IDENTITY)`. Change it to `SEQUENCE`.

6. **"Difference between save() and saveAndFlush()?"**
   `save()` just puts the entity in the Persistence Context. The actual SQL `INSERT`/`UPDATE` is delayed until transaction commit (or right before a query). `saveAndFlush()` forces the SQL execution immediately. (Useful when you need the DB to generate a trigger-based value immediately).

---

## 📝 Summary & Key Takeaways

- **Spring Data JPA** generates boilerplate code, but **Hibernate** does the heavy lifting.
- Understand the **Persistence Context (L1 Cache)**: It tracks entity state. You don't need to call `.save()` on an attached entity if you modify it inside a transaction; Hibernate's dirty checking will update it automatically.
- Always use `FetchType.LAZY` for everything. Use `JOIN FETCH` when you need the data.
- **N+1 Problem** is the #1 killer of ORM performance. Know how to detect and solve it.
- Never use `IDENTITY` if you want batch inserts. Use `SEQUENCE`.
- **Transactions** are proxy-based. Understand propagation and rollback rules.
- Connection pools should be small (usually < 20).

---

## ❓ Practice Questions

1. Describe the exact journey of a method call annotated with `@Transactional`. When is the connection opened, and when is the transaction committed?
2. Write a Spring Data JPA Repository method that returns a custom DTO using a JPQL constructor expression to avoid fetching unused columns.
3. Your application logs show 1 `SELECT` for a `Category` and 50 `SELECT` statements for its `Products` during an API call. What is this, and write code to fix it.
4. Explain why `@GeneratedValue(strategy = GenerationType.IDENTITY)` breaks JDBC batching in Hibernate, while `SEQUENCE` does not.
5. In a bidirectional `@OneToMany` relationship between `Author` and `Book`, explain what the `mappedBy` attribute does and why helper methods are necessary to maintain the relationship.
