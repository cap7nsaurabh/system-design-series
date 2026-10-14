# Advanced Database Techniques in Spring Boot (Beyond JPA)

When developers think of database access in Spring Boot, **Spring Data JPA** (backed by Hibernate) is usually the first tool that comes to mind. It provides a rich set of features, including an abstraction over database dialects, automatic query generation, lazy loading, caching, and dirty checking. However, as applications scale and data access patterns become more complex, the "magic" of JPA can become a bottleneck.

In this comprehensive module, we will explore advanced database techniques in Spring Boot that go **beyond JPA**. We will dive deep into alternative relational data access strategies (like JdbcTemplate, Spring Data JDBC, R2DBC, and MyBatis), as well as NoSQL integrations (MongoDB and Redis). 

By the end of this module, you will understand when to stick with JPA, when to drop down to a lower-level SQL-centric approach, how to build reactive data pipelines, and how to conquer distributed, non-relational persistence.

---

## 1. Beyond Hibernate/JPA: The Breaking Point

### 1.1 Why Move Away From JPA?
While JPA is exceptional for simple CRUD applications and managing complex object graphs, it introduces significant overhead that can hinder performance in high-throughput or data-intensive applications.

**1. Overhead and Memory Footprint**
Hibernate requires a massive amount of memory to maintain its First-Level Cache (the Persistence Context). Every entity retrieved from the database is attached to this context, mapped, and proxy-wrapped. For batch processing or fetching large datasets, this overhead leads to frequent Garbage Collection (GC) pauses and `OutOfMemoryError` exceptions.

**2. Dirty Checking Performance**
To support automatic updates (dirty checking), Hibernate must retain a snapshot of every entity's initial state. When the transaction commits, Hibernate compares the current state of every managed entity against its snapshot. In scenarios where you load 10,000 records just to update 10, the dirty checking mechanism will waste cycles checking all 10,000 entities.

**3. Complex Queries and The N+1 Problem**
Writing complex analytical queries (e.g., joins across 5 tables with group bys and window functions) in JPQL or Criteria API is extremely painful. Furthermore, JPA's default mapping strategies (especially `FetchType.EAGER` vs `FetchType.LAZY`) notoriously lead to the **N+1 Selects Problem**, where fetching a parent entity and its children results in hundreds of individual SQL queries being fired sequentially.

**4. Loss of SQL Control**
Hibernate generates SQL for you. While usually correct, the generated SQL might not be optimal for your specific database index structure. When a DBA gives you a highly optimized raw SQL query with specific database hints, translating it into JPQL or managing it as a native query in JPA can be cumbersome.

### 1.2 Layered Architecture: When to Use What
A common, modern architectural pattern is to **mix and match** data access technologies based on the use case. This is sometimes referred to as the **Command Query Responsibility Segregation (CQRS)** pattern at the data access level.

| Use Case | Recommended Technology | Rationale |
|----------|------------------------|-----------|
| **Simple CRUD & Object Graphs** | Spring Data JPA | Rapid development, built-in relationship mapping. |
| **Heavy Reads & Reporting** | JdbcTemplate / MyBatis | Pure SQL control, lightweight DTO mapping, no persistence context overhead. |
| **High-Performance Batch Inserts**| JdbcTemplate | `batchUpdate` allows sending thousands of rows to the DB in a single network round-trip. |
| **Domain-Driven Design (DDD)** | Spring Data JDBC | Forces aggregate roots, no lazy loading traps, no dirty checking overhead. |
| **Reactive / Non-blocking IO** | Spring Data R2DBC | Essential for WebFlux applications handling massive concurrency with few threads. |
| **Unstructured / Fast Data** | MongoDB / Redis | Schema-less design, in-memory caching, specialized data structures (geospatial, graphs). |

> [!TIP]
> **Production Best Practice:** It is completely fine (and often encouraged) to use **Spring Data JPA** for handling your business logic inserts/updates, while injecting a **JdbcTemplate** in the same service to handle complex read-only queries that project directly into lightweight DTOs.

---

## 2. Spring JdbcTemplate

`JdbcTemplate` is Spring's core API for interacting with relational databases. It removes the boilerplate of raw JDBC (opening connections, handling `SQLException`, closing ResultSets) while giving you 100% control over the executed SQL.

### 2.1 Setup and Basics
To use JdbcTemplate, simply include the JDBC starter in your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

Spring Boot auto-configures a `DataSource` and a `JdbcTemplate` bean.

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class EmployeeJdbcRepository {

    private final JdbcTemplate jdbcTemplate;

    public EmployeeJdbcRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public int countEmployees() {
        return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM employees", Integer.class);
    }
}
```

### 2.2 RowMapper and ResultSetExtractor
When querying for objects, you need to tell Spring how to map the `ResultSet` to your Java objects.

**RowMapper:** Maps a single row of a `ResultSet` to an object. Used for `query()` and `queryForObject()`.
**ResultSetExtractor:** Maps an entire `ResultSet` to an object (or collection). Useful when processing joins where multiple rows represent a single logical entity.

```java
import org.springframework.jdbc.core.RowMapper;
import java.sql.ResultSet;
import java.sql.SQLException;

public class EmployeeRowMapper implements RowMapper<Employee> {
    @Override
    public Employee mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new Employee(
            rs.getLong("id"),
            rs.getString("first_name"),
            rs.getString("last_name"),
            rs.getString("department"),
            rs.getBigDecimal("salary")
        );
    }
}
```

Usage in the repository:
```java
public Employee findById(Long id) {
    String sql = "SELECT * FROM employees WHERE id = ?";
    return jdbcTemplate.queryForObject(sql, new EmployeeRowMapper(), id);
}

public List<Employee> findByDepartment(String dept) {
    String sql = "SELECT * FROM employees WHERE department = ?";
    return jdbcTemplate.query(sql, new EmployeeRowMapper(), dept);
}
```

### 2.3 NamedParameterJdbcTemplate (Advantages over `?` placeholders)
Standard `JdbcTemplate` uses standard JDBC `?` placeholders. This becomes extremely error-prone when dealing with many parameters, as the order must match exactly.

`NamedParameterJdbcTemplate` allows you to use named placeholders (e.g., `:firstName`).

```java
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.stereotype.Repository;

@Repository
public class AdvancedEmployeeRepository {

    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    public AdvancedEmployeeRepository(NamedParameterJdbcTemplate namedJdbcTemplate) {
        this.namedJdbcTemplate = namedJdbcTemplate;
    }

    public List<Employee> findByCriteria(String dept, BigDecimal minSalary) {
        String sql = "SELECT * FROM employees WHERE department = :dept AND salary >= :minSal";
        
        MapSqlParameterSource params = new MapSqlParameterSource();
        params.addValue("dept", dept);
        params.addValue("minSal", minSalary);
        
        return namedJdbcTemplate.query(sql, params, new EmployeeRowMapper());
    }
}
```
> [!NOTE]
> `NamedParameterJdbcTemplate` wraps the standard `JdbcTemplate`. Spring Boot auto-configures this for you. Use it by default to make your SQL queries much more readable.

### 2.4 Batch Operations for High-Performance Inserts
If you need to insert 10,000 records, looping over `jdbcTemplate.update()` will cause 10,000 network round trips to the database. Instead, use `batchUpdate()`.

```java
import org.springframework.jdbc.core.BatchPreparedStatementSetter;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;

public void batchInsertEmployees(List<Employee> employees) {
    String sql = "INSERT INTO employees (first_name, last_name, department, salary) VALUES (?, ?, ?, ?)";
    
    jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
        @Override
        public void setValues(PreparedStatement ps, int i) throws SQLException {
            Employee emp = employees.get(i);
            ps.setString(1, emp.getFirstName());
            ps.setString(2, emp.getLastName());
            ps.setString(3, emp.getDepartment());
            ps.setBigDecimal(4, emp.getSalary());
        }
        
        @Override
        public int getBatchSize() {
            return employees.size();
        }
    });
}
```

> [!IMPORTANT]
> To truly benefit from batch inserts at the driver level, you must append `?rewriteBatchedStatements=true` (for MySQL) or `?reWriteBatchedInserts=true` (for PostgreSQL) to your JDBC connection URL. Without this, the JDBC driver might just execute single inserts sequentially behind the scenes!

---

## 3. Spring Data JDBC

Spring Data JDBC represents a paradigm shift. It offers the familiar repository interfaces (`CrudRepository`, `PagingAndSortingRepository`) without the complexities of JPA. 

### 3.1 Difference Between Spring Data JDBC and Spring Data JPA

| Feature | Spring Data JPA | Spring Data JDBC |
|---------|-----------------|------------------|
| **Underlying Tech** | Hibernate / ORM | Raw JDBC + simple mapping |
| **Context** | Uses Persistence Context | No Persistence Context |
| **Lazy Loading** | Yes, proxies everywhere | **No.** Everything is eager. |
| **Dirty Checking** | Yes, auto-updates on commit | **No.** Must explicitly call `save()` |
| **Caching** | 1st and 2nd level caches | No caching |
| **Query Generation** | Very advanced | Basic, mostly requires `@Query` |

### 3.2 Domain-Driven Design (DDD) Approach
Spring Data JDBC is heavily opinionated towards DDD. It expects you to model your data as **Aggregates**. An aggregate is a cluster of domain objects that can be treated as a single unit.

In Spring Data JDBC, you only create Repositories for **Aggregate Roots**. If a `Customer` has an `Address`, the `Address` belongs to the `Customer`. You save the `Customer`, and the framework handles deleting and recreating the `Address`.

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.relational.core.mapping.MappedCollection;
import java.util.Set;

@Table("customer")
public class Customer {
    
    @Id
    private Long id;
    private String name;

    // The Set represents a one-to-many relationship
    @MappedCollection(idColumn = "customer_id")
    private Set<Address> addresses;

    // Constructors, Getters, Setters
}

@Table("address")
public class Address {
    private String street;
    private String city;
    // Note: No @Id needed if it's purely a value object managed by the aggregate root.
    // However, best practice is to include an ID if it requires independent updates.
}
```

### 3.3 Lack of Lazy Loading (Feature, Not a Bug)
Spring Data JDBC deliberately omits lazy loading. When you fetch a `Customer`, all `Address` records are fetched immediately via a single query or a secondary query. 
*Why?* Because lazy loading causes the N+1 problem and `LazyInitializationException` outside of active transactions. By forcing eager loading, Spring Data JDBC guarantees that when you have an object, it is fully populated and completely detached from any database connection.

### 3.4 @MappedCollection and Relationships
Spring Data JDBC handles relationships differently. If you load an aggregate root, modify its collection, and call `save()`, Spring Data JDBC does not do complex diffing. Instead, it typically **deletes all child entities** and **re-inserts** the ones currently in the collection.

> [!WARNING]
> This behavior (delete and re-insert) is fast but can cause issues if your child tables have foreign keys pointing to them from outside the aggregate. Ensure you truly understand aggregate boundaries before using collections in Spring Data JDBC!

---

## 4. Spring Data R2DBC (Reactive)

Traditional JDBC is **blocking**. When you execute a query, the calling thread halts and waits for the database to return the results. In high-concurrency systems (like WebFlux applications), blocking a thread is fatal because there are very few threads available.

### 4.1 What is R2DBC?
**R2DBC (Reactive Relational Database Connectivity)** is an API specification designed from the ground up for reactive programming with SQL databases. It uses non-blocking I/O, meaning threads are released to do other work while waiting for the database network response.

### 4.2 Setup and Basics
Include the starter and the reactive driver for your database (e.g., PostgreSQL):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

Configure `application.yml`:
```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypass
```

### 4.3 ReactiveCrudRepository and R2dbcEntityTemplate
Spring provides familiar repository abstractions, returning Project Reactor types (`Mono` for 0-1 items, `Flux` for 0-N items).

```java
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface ProductRepository extends ReactiveCrudRepository<Product, Long> {
    
    // Returns a Flux (stream of products)
    Flux<Product> findByCategory(String category);
    
    // Custom query requires R2DBC SQL format ($1 instead of ?)
    @Query("SELECT * FROM products WHERE price > $1")
    Flux<Product> findExpensiveProducts(BigDecimal price);
}
```

For more complex dynamic queries, use `DatabaseClient` or `R2dbcEntityTemplate`:

```java
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

@Service
public class ProductService {

    private final DatabaseClient databaseClient;

    public ProductService(DatabaseClient databaseClient) {
        this.databaseClient = databaseClient;
    }

    public Flux<String> getProductNamesByCategory(String category) {
        return databaseClient.sql("SELECT name FROM products WHERE category = :category")
                .bind("category", category)
                .map((row, rowMetadata) -> row.get("name", String.class))
                .all();
    }
}
```

### 4.4 Transaction Management in R2DBC
You can still use `@Transactional` in reactive applications! However, the underlying implementation uses a `ReactiveTransactionManager`. The transaction context is stored in the **Reactor Context**, not in a `ThreadLocal` (since threads switch constantly in reactive flows).

```java
import org.springframework.transaction.annotation.Transactional;
import reactor.core.publisher.Mono;

@Service
public class OrderService {
    
    @Transactional
    public Mono<Void> placeOrder(Order order) {
        return orderRepository.save(order)
            .flatMap(savedOrder -> inventoryService.deductStock(savedOrder.getItems()))
            .then();
    }
}
```

---

## 5. NoSQL Integrations

Spring Data abstracts NoSQL interactions beautifully, making it easy to swap paradigms without learning entirely new APIs.

### 5.1 Spring Data MongoDB
MongoDB is a document database. Instead of rows and columns, it stores JSON-like BSON documents.

#### 5.1.1 Document Mapping
Use `@Document` instead of `@Entity`, and `@Id` for the primary key (typically a `String` mapped to Mongo's `ObjectId`).

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "users")
public class User {
    @Id
    private String id; // Maps to _id
    
    private String username;
    private String email;
    
    // Getters and Setters
}
```

#### 5.1.2 MongoTemplate vs MongoRepository
`MongoRepository` works identically to JPA repositories for CRUD operations. For complex queries (like aggregations), use `MongoTemplate`.

```java
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.stereotype.Repository;

@Repository
public class CustomUserRepository {
    
    private final MongoTemplate mongoTemplate;

    public CustomUserRepository(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    public List<User> findByDomain(String domain) {
        Query query = new Query();
        // Regex search
        query.addCriteria(Criteria.where("email").regex(".*@" + domain));
        return mongoTemplate.find(query, User.class);
    }
}
```

#### 5.1.3 Aggregation Framework
MongoDB's aggregation pipeline is extremely powerful. Spring Data provides a fluent API for it.

```java
import org.springframework.data.mongodb.core.aggregation.Aggregation;
import org.springframework.data.mongodb.core.aggregation.AggregationResults;

public void getAverageSalaryByDepartment() {
    Aggregation agg = Aggregation.newAggregation(
        Aggregation.group("department").avg("salary").as("averageSalary"),
        Aggregation.sort(Sort.Direction.DESC, "averageSalary")
    );

    AggregationResults<DepartmentStats> results = mongoTemplate.aggregate(
        agg, "employees", DepartmentStats.class
    );
    
    List<DepartmentStats> stats = results.getMappedResults();
}
```

### 5.2 Spring Data Redis (Advanced)
Redis is usually viewed as a cache, but it can operate as a full-fledged database.

#### 5.2.1 Redis Repositories
Spring Data Redis allows you to map objects to Redis Hashes.

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.RedisHash;

@RedisHash("Session")
public class UserSession {
    @Id
    private String sessionId;
    private String userId;
    private long expiryTimestamp;
}
```
You can use `CrudRepository<UserSession, String>` to save this object. Behind the scenes, Spring creates a Redis HASH with key `Session:<sessionId>`.

#### 5.2.2 Pub/Sub with RedisMessageListenerContainer
Redis can act as a lightweight message broker.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;

@Configuration
public class RedisPubSubConfig {

    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory,
                                            MessageListenerAdapter listenerAdapter) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listenerAdapter, new ChannelTopic("order_events"));
        return container;
    }

    @Bean
    MessageListenerAdapter listenerAdapter(OrderEventReceiver receiver) {
        return new MessageListenerAdapter(receiver, "receiveMessage");
    }
}

// Receiver class
public class OrderEventReceiver {
    public void receiveMessage(String message) {
        System.out.println("Received order event: " + message);
    }
}
```

---

## 6. MyBatis Integration

MyBatis takes a fundamentally different approach from JPA. Instead of mapping Java Objects to Database Tables (ORM), it maps **Java Methods to SQL Statements**.

### 6.1 Why MyBatis?
- **SQL-Centric:** You retain 100% control over the exact SQL being run.
- **Easy optimization:** DBAs can write complex XML files with SQL, and Java devs just call them.
- **Dynamic SQL:** MyBatis excels at building dynamic `WHERE` clauses based on incoming parameters.

### 6.2 Setup
Add the MyBatis Spring Boot Starter:
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>
```

### 6.3 Annotation vs XML Mappers

**Annotation Approach (Best for simple queries):**
```java
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Insert;

@Mapper
public interface UserMapper {
    
    @Select("SELECT id, username, email FROM users WHERE id = #{id}")
    User findById(Long id);

    @Insert("INSERT INTO users(username, email) VALUES(#{username}, #{email})")
    void insert(User user);
}
```

**XML Approach (Best for complex, dynamic queries):**
Create an interface:
```java
@Mapper
public interface ReportMapper {
    List<ReportDto> generateComplexReport(ReportCriteria criteria);
}
```

Create an XML file in `src/main/resources/mapper/ReportMapper.xml`:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.ReportMapper">

    <select id="generateComplexReport" resultType="com.example.dto.ReportDto">
        SELECT a.name, b.total_sales
        FROM table_a a
        JOIN table_b b ON a.id = b.a_id
        <where>
            <if test="startDate != null">
                AND b.created_at >= #{startDate}
            </if>
            <if test="status != null">
                AND a.status = #{status}
            </if>
        </where>
    </select>
</mapper>
```

---

## 🔍 Deep Dive: How Spring Manages Connection Pools

Whether you use JPA, JdbcTemplate, or MyBatis, Spring Boot relies on a **Connection Pool** under the hood. By default, Spring Boot 2.x and 3.x use **HikariCP**. 

HikariCP is notoriously fast because it utilizes bytecode manipulation (Javassist) and lock-free data structures (ConcurrentBag).
When `jdbcTemplate.query()` is called:
1. `JdbcTemplate` asks `DataSourceUtils.getConnection(dataSource)`.
2. HikariCP checks its pool. If a connection is idle, it returns a proxy to the connection.
3. If no connection is idle, it blocks until `connectionTimeout` is reached.
4. The query executes.
5. `JdbcTemplate` ensures the connection is returned to the pool (closed) in a `finally` block, preventing resource leaks.

---

## ⚠️ Interview Traps & Gotchas

**1. Mixing JPA and JdbcTemplate in the Same Transaction**
> **Trap:** You update an entity using JPA, and then immediately query the database using JdbcTemplate in the same `@Transactional` method. The JdbcTemplate query doesn't see the update!
> **Why:** JPA delays executing `UPDATE` statements until flush time (usually before transaction commit). The JdbcTemplate fires immediately directly to the DB.
> **Fix:** Call `entityManager.flush()` manually before executing the JdbcTemplate query.

**2. R2DBC Blocking Thread Trap**
> **Trap:** Using `RestTemplate` or a standard JDBC driver (like `java.sql.Connection`) inside an R2DBC reactive chain.
> **Why:** Project Reactor uses a small thread pool (usually CPU cores). If one thread blocks waiting for a synchronous network call, it paralyzes a large portion of the application.
> **Fix:** Use `WebClient` for HTTP calls and ensure ALL database access uses R2DBC when in a reactive flow. Use BlockHound in tests to detect blocking calls.

**3. MongoDB Transaction Support**
> **Trap:** Decorating a Mongo method with `@Transactional` and expecting it to roll back on error.
> **Why:** MongoDB only supports multi-document ACID transactions if it is running as a **Replica Set**. A standalone MongoDB instance (default in local dev) will silently ignore transactions or throw an exception depending on configuration.
> **Fix:** Ensure your local Docker-compose spins up MongoDB as a single-node replica set if you need transaction testing.

**4. Batch Processing with JPA**
> **Trap:** Trying to insert 100,000 records using `jpaRepository.saveAll()`.
> **Why:** Hibernate loads everything into memory. It will eventually crash with `OutOfMemoryError`.
> **Fix:** Use `JdbcTemplate.batchUpdate()` for bulk inserts. If you MUST use JPA, configure `hibernate.jdbc.batch_size`, and periodically call `entityManager.flush()` and `entityManager.clear()` inside a loop.

---

## 📝 Summary & Key Takeaways

1. **Spring Data JPA is not a silver bullet.** It's excellent for domain modeling but can suffer under heavy read loads or batch processing.
2. **JdbcTemplate** provides raw SQL control and is highly recommended for complex reads, reporting, and bulk inserts.
3. **Spring Data JDBC** is a lighter alternative to JPA, enforcing strict Domain-Driven Design principles without the overhead of lazy loading or dirty checking.
4. **Spring Data R2DBC** brings non-blocking, reactive database access to the relational world, essential for WebFlux applications.
5. **MyBatis** maps Java methods directly to raw SQL, making it ideal for legacy databases or DBA-heavy environments.
6. **NoSQL Integrations** (Mongo, Redis) follow the same repository patterns, allowing developers to switch data paradigms effortlessly.

---

## ❓ Practice Questions

1. Explain the architectural scenario where you would use both Spring Data JPA and JdbcTemplate in the same Spring Boot microservice.
2. What is the fundamental difference in how Spring Data JPA and Spring Data JDBC handle one-to-many relationships?
3. How does R2DBC achieve higher concurrency than standard JDBC, and what is the trade-off?
4. Write a snippet demonstrating a bulk insert using `NamedParameterJdbcTemplate`.
5. Why does mixing JPA state management with pure JDBC queries inside a single transaction cause data inconsistency, and how do you resolve it?
