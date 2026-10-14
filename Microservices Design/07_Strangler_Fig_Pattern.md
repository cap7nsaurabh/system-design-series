# Strangler Fig Pattern

## 1. Pattern Overview and Intent
The **Strangler Fig Pattern** (sometimes just called Strangler Pattern) is an approach to safely migrating a monolithic application to a microservices architecture. Instead of rewriting the entire system at once (a "big bang" rewrite), you incrementally extract features into new microservices. A routing facet (the "strangler") intercepts calls to the system and directs them either to the legacy monolith or the new microservices.

**Intent:**
- Mitigate the immense risk of a "big bang" system rewrite.
- Allow continuous delivery of business value during a migration.
- Provide a clear fallback mechanism if the new implementation fails.

## 2. Detailed Explanation of How it Works

The pattern gets its name from the strangler fig tree, which seeds in the upper branches of a host tree and gradually grows down to the ground, eventually replacing the host entirely.

**The Process:**
1. **Identify:** Choose a specific, well-defined component or feature of the monolith to migrate.
2. **Build:** Develop the new microservice alongside the monolith.
3. **Route (Strangle):** Implement a Routing Facade (often an API Gateway or reverse proxy like Nginx/Envoy). This facade routes requests for the newly extracted feature to the new microservice, while all other requests continue to route to the monolith.
4. **Iterate:** Repeat this process for other components. Over time, the monolith receives fewer and fewer requests.
5. **Retire:** Once all features have been strangled and migrated, the monolith is safely decommissioned.

### Architecture Diagram

```mermaid
flowchart TD
    Client((Client)) --> Gateway{API Gateway / Router}
    
    Gateway -->|Legacy Routes| Monolith[Legacy Monolith Application]
    Gateway -->|New Routes| ServiceA[Microservice A (Migrated)]
    Gateway -->|New Routes| ServiceB[Microservice B (Migrated)]
    
    Monolith -.-> DB[(Legacy DB)]
    ServiceA -.-> DB_A[(DB A)]
    ServiceB -.-> DB_B[(DB B)]
```

## 3. Real-World Example: Legacy E-Commerce Migration

An e-commerce company has a massive, 10-year-old monolithic application built in Java. They want to move to a Node.js/Go microservices architecture.

### The Strategy
They decide to migrate the "User Reviews" system first, as it's loosely coupled to the rest of the monolith.

### The Routing Configuration (Nginx Proxy)

```nginx
server {
    listen 80;
    server_name my-ecommerce.com;

    # The New Microservice (Strangled functionality)
    location /api/v1/reviews {
        proxy_pass http://reviews-microservice:8080;
    }

    # The Legacy Monolith (Everything else)
    location / {
        proxy_pass http://legacy-monolith:8080;
    }
}
```

When a user loads a product page, the page requests reviews via `/api/v1/reviews`. The proxy routes this to the new Go service. If the user adds an item to the cart (`/api/v1/cart`), the proxy routes it to the legacy Java monolith.

## 4. Pros and Cons

### Pros
- **Risk Mitigation:** You migrate small pieces at a time. If something breaks, you only roll back that specific piece.
- **Immediate Value:** You don't have to wait 2 years for a full rewrite to finish before seeing the benefits of the new architecture.
- **Parallel Running:** You can run both systems in parallel for a time (using tools like Dark Launching) to verify the new system behaves identically to the old one before switching traffic.
- **Clear Rollback:** If the new service fails, updating the routing table immediately sends traffic back to the legacy monolith.

### Cons
- **Data Sync Complexity:** The monolith and the new services often need to share data during the transition period. Keeping legacy and modern databases synchronized is very difficult.
- **Performance Overhead:** The routing layer adds a network hop, which can slightly increase latency.
- **Maintenance of Two Systems:** During the migration (which can take years), teams must maintain both the legacy codebase and the new microservices.

## 5. When to Use / When NOT to Use

**When to Use:**
- When migrating large, complex, monolithic legacy applications to microservices.
- When the business cannot afford a feature freeze during a rewrite.
- When you have a clear way to intercept and route external requests.

**When NOT to Use:**
- When the legacy application is small enough that a complete rewrite is trivial and low-risk.
- When the monolith cannot be broken down (e.g., highly coupled "spaghetti code" where extracting one piece breaks everything else).

## 6. Common Interview Questions
1. **How do you handle data integration between the monolith and the new microservice during the Strangler migration?**
   *Answer:* This is the hardest part. You can use an Anti-Corruption Layer (ACL), database triggers, Change Data Capture (CDC), or have the new service call the legacy monolith's internal APIs to fetch/sync data until the migration is complete.
2. **What is the rollback strategy if the newly strangled service fails?**
   *Answer:* The rollback strategy is purely configuration-based at the routing layer. You simply update the API Gateway or reverse proxy to route traffic for that specific endpoint back to the legacy monolith.
3. **How do you choose which component to 'strangle' first?**
   *Answer:* Start with a low-risk, loosely coupled component to prove the pattern and build team confidence. Later, tackle components that require the highest scalability or are blocking feature development.
