# Spring Boot Deep-Dive Course

Welcome to the ultimate Spring Boot Deep-Dive Course! This course is designed to take you from a basic understanding of Spring to an expert level, ready to tackle complex system design problems, production scenarios, and rigorous technical interviews.

## 🌟 Why Spring Boot?
Spring Boot simplifies the development of production-ready Spring applications. It eliminates boilerplate configuration, provides sensible defaults, and embeds a web server so you can simply run your application. In the modern backend ecosystem, Spring Boot is the de facto standard for building Java microservices.

## 📚 Prerequisites
To get the most out of this course, you should have:
- Solid understanding of Core Java (Java 8+ features, especially Java 17)
- Basic understanding of Maven or Gradle
- Familiarity with basic REST concepts (HTTP methods, status codes)

## 🗺️ Course Modules (Table of Contents)

### Part 1: Core Spring & Web
1. [Module 1: Spring Boot Fundamentals](./01-fundamentals.md) - IoC, DI, Application Context, Bean Lifecycle
2. [Module 2: Annotations Deep-Dive](./02-annotations.md) - Stereotypes, Web, Data, Configuration, and Traps
3. [Module 3: REST API & Web Layer](./03-rest-api.md) - Controllers, Filters, Interceptors, DispatcherServlet
4. [Module 4: Exception Handling](./04-exception-handling.md) - @ControllerAdvice, ProblemDetail, Error Hierarchies

### Part 2: Data & State Management
5. [Module 5: Database & JPA](./05-database-jpa.md) - Hibernate, Transactions, Lazy Loading, N+1 Problem
6. [Module 6: Concurrency & Race Conditions](./06-concurrency.md) - Singletons, @Async, Thread Pools, Synchronization
7. [Module 7: Scheduling & Background Jobs](./07-scheduling.md) - @Scheduled, Quartz, ShedLock, Threading Traps
8. [Module 8: Caching Strategies](./08-caching.md) - Cacheable, Redis, Caffeine, Invalidation, Stampede
13. [Module 13: Advanced Databases](./13-advanced-databases.md) - Spring Data JDBC, R2DBC, MongoDB, MyBatis

### Part 3: Architecture & Scale
9. [Module 9: Rate Limiting & Traffic Management](./09-rate-limiting.md) - Bucket4j, Algorithms, Distributed Limits
10. [Module 10: Security & JWT](./10-security.md) - Filter Chains, Authentication, Authorization, OAuth2
11. [Module 11: File Management](./11-file-management.md) - Multipart, S3, Streaming, Async Processing
14. [Module 14: Messaging & Event-Driven](./14-message-queues.md) - Kafka, RabbitMQ, Outbox Pattern, Saga
19. [Module 19: GraphQL & WebSockets](./19-graphql-websockets.md) - Alternative APIs, Real-time, DataLoader N+1 Fix

### Part 4: Production, Testing & Observability
15. [Module 15: Observability & Actuator](./15-observability.md) - Micrometer, Grafana, Distributed Tracing (Sleuth/OpenTelemetry)
16. [Module 16: Testing Strategies](./16-testing.md) - Testcontainers, @WebMvcTest, Mocking, WireMock
17. [Module 17: Reactive WebFlux](./17-reactive-webflux.md) - Event Loop, Mono/Flux, Backpressure, BlockHound
18. [Module 18: Deployment & GraalVM](./18-deployment-docker-graalvm.md) - Docker, Layered JARs, Kubernetes Probes, AOT Native Images

### Part 5: Interview Preparation
12. [Module 12: Interview Traps & Gotchas](./12-interview-traps.md) - Tricky questions, AOP proxy issues, top misconceptions

## 🚀 Recommended Learning Path

```mermaid
flowchart TD
    A[01 Fundamentals] --> B[02 Annotations]
    B --> C[03 REST API]
    C --> D[04 Exception Handling]
    D --> E[05 Database & JPA]
    E --> M[13 Advanced DBs]
    E --> F[06 Concurrency]
    F --> G[07 Scheduling]
    E --> H[08 Caching]
    C --> I[09 Rate Limiting]
    C --> J[10 Security]
    C --> K[11 File Management]
    
    M --> N[14 Message Queues]
    N --> O[17 WebFlux]
    C --> U[19 GraphQL & WebSockets]
    
    C --> P[16 Testing]
    P --> Q[15 Observability]
    Q --> R[18 Deployment & GraalVM]
    
    F --> L[12 Interview Traps]
    H --> L
    J --> L
    R --> L
```

## 🛠️ Quick Start & Setup

To follow along with the code examples:
1. Go to [Spring Initializr](https://start.spring.io/)
2. Select:
   - Project: Maven or Gradle
   - Language: Java
   - Spring Boot: 3.x (latest stable)
   - Java: 17+
   - Dependencies: Spring Web, Spring Data JPA, H2 Database (for quick local testing), Validation
3. Generate, extract, and open in your favorite IDE (IntelliJ IDEA, Eclipse, VS Code).
4. Run `mvn spring-boot:run` or `./gradlew bootRun`.

Let's get started with [Module 1: Fundamentals](./01-fundamentals.md)!
