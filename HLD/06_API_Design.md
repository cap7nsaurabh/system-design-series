# 🌐 API Design: REST, GraphQL, gRPC & WebSockets

> *"An API is a contract between a server and a client. Like all contracts, the details matter enormously."*

---

## 📋 Table of Contents

1. [Why API Design Matters](#why-api-design-matters)
2. [REST API — Deep Dive](#rest)
3. [REST Best Practices & Design Patterns](#rest-best-practices)
4. [GraphQL — Deep Dive](#graphql)
5. [gRPC — Deep Dive](#grpc)
6. [WebSockets](#websockets)
7. [Server-Sent Events (SSE)](#sse)
8. [Long Polling vs Short Polling](#polling)
9. [API Gateway Pattern](#api-gateway)
10. [Rate Limiting Algorithms](#rate-limiting)
11. [API Versioning](#api-versioning)
12. [Real-Life Scenario](#real-life-scenario)
13. [🔥 Big Tech Story: Netflix's GraphQL Federation](#big-tech-story)
14. [Trade-offs & Drawbacks](#trade-offs)
15. [Best Practices](#best-practices)
16. [Key Takeaways](#key-takeaways)

---

## 🤔 Why API Design Matters

An API is the **public interface** of your service. Once published and used by clients, changing it is painful and expensive. Bad API design causes:
- Breaking changes that require coordinated client updates
- Over-fetching (clients receive more data than needed → wasted bandwidth)
- Under-fetching (clients need multiple requests to get enough data)
- Security vulnerabilities from poorly designed auth
- Performance issues from inefficient query patterns

A well-designed API should be:
- **Intuitive:** Developers can guess the behavior without reading docs
- **Consistent:** Same patterns throughout
- **Forward-compatible:** Can evolve without breaking existing clients
- **Performant:** Efficient data transfer, minimal round trips
- **Secure:** Authentication, authorization, and input validation built in

---

## 🏛️ REST API — Deep Dive

**REST (Representational State Transfer)** is an architectural style for distributed hypermedia systems, defined by Roy Fielding in 2000.

REST is not a protocol or standard — it's a set of **constraints**:
1. **Client-Server:** Separation of concerns between UI and data storage
2. **Stateless:** Each request contains all information needed; no session state on server
3. **Cacheable:** Responses must define whether they're cacheable
4. **Uniform Interface:** Standardized way to interact with resources
5. **Layered System:** Client doesn't need to know if it's talking to end server or intermediary
6. **Code on Demand (optional):** Server can send executable code to clients

### Richardson Maturity Model

A way to measure how "RESTful" an API is:

**Level 0 — The Swamp of POX (Plain Old XML)**
```
POST /api
Body: { "action": "getUser", "id": 123 }
```
Everything through a single endpoint. Like RPC over HTTP.

**Level 1 — Resources**
```
GET /users/123
POST /orders
GET /products
```
Separate endpoints per resource, but still not using HTTP methods correctly.

**Level 2 — HTTP Verbs**
```
GET    /users/123       ← Read
POST   /users           ← Create
PUT    /users/123       ← Full update
PATCH  /users/123       ← Partial update
DELETE /users/123       ← Delete
```
Most real-world APIs are here. Uses proper HTTP methods and status codes.

**Level 3 — Hypermedia Controls (HATEOAS)**
```json
GET /orders/456
{
    "id": 456,
    "status": "pending",
    "total": 99.99,
    "_links": {
        "self": { "href": "/orders/456" },
        "cancel": { "href": "/orders/456/cancel", "method": "DELETE" },
        "pay": { "href": "/payments", "method": "POST" },
        "user": { "href": "/users/123" }
    }
}
```
Responses include links to related actions. Client discovers API dynamically.

**Most REST APIs are at Level 2.** HATEOAS is rarely implemented in practice.

### HTTP Methods Semantics

| Method | Idempotent? | Safe? | Body? | Use For |
|---|---|---|---|---|
| GET | ✅ Yes | ✅ Yes | No | Read resources |
| POST | ❌ No | ❌ No | Yes | Create, non-idempotent actions |
| PUT | ✅ Yes | ❌ No | Yes | Full replace of resource |
| PATCH | ❌ No | ❌ No | Yes | Partial update |
| DELETE | ✅ Yes | ❌ No | Optional | Delete resource |
| HEAD | ✅ Yes | ✅ Yes | No | GET without response body |
| OPTIONS | ✅ Yes | ✅ Yes | No | CORS preflight, available methods |

**Idempotent:** Calling the same request multiple times has the same effect as calling it once.
**Safe:** Does not modify server state.

### HTTP Status Codes

| Range | Meaning | Examples |
|---|---|---|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

**Critical distinction: 401 vs 403**
- `401 Unauthorized`: Not authenticated — provide credentials
- `403 Forbidden`: Authenticated but not authorized — you don't have permission

### Pagination

**Offset Pagination (Simple)**
```
GET /posts?page=3&per_page=20
GET /posts?offset=40&limit=20
```
- Simple to implement
- ❌ Inconsistent results if data changes between pages (new item inserted shifts all offsets)
- ❌ Poor performance for deep pages (DB must skip N rows: `OFFSET 100000`)
- ❌ Total count query is expensive

**Cursor Pagination (Production Standard)**
```
GET /posts?after=eyJpZCI6MTAwfQ==&limit=20

Response:
{
    "data": [...],
    "cursor": {
        "next": "eyJpZCI6MTIwfQ==",
        "has_more": true
    }
}
```
The cursor is an opaque base64-encoded value representing the position in the result set (e.g., the last seen ID or timestamp).

- ✅ Stable (inserting new items doesn't affect cursor position)
- ✅ O(1) regardless of page depth (uses index scan from cursor position)
- ❌ Cannot jump to arbitrary pages (no "go to page 5")
- ❌ More complex to implement

**Keyset Pagination (Similar to cursor, explicit)**
```
GET /posts?created_before=2024-01-15T10:00:00Z&limit=20
```
Uses actual column values for the position.

### Filtering and Sorting

```
GET /products?category=electronics&min_price=100&max_price=500
GET /products?sort=-created_at  (minus = descending)
GET /products?sort=price,-rating (price asc, then rating desc)
GET /products?fields=id,name,price  (sparse fieldsets)
```

---

## ✨ REST Best Practices & Design Patterns

### URL Design

```
✅ GOOD:
GET  /users/{id}                      # Get a user
POST /users                           # Create a user
GET  /users/{id}/orders               # Get orders for a user
POST /users/{id}/orders               # Create order for a user
POST /orders/{id}/cancel              # Cancel action (verb ok for actions)
GET  /search?q=shoes&category=sports  # Search

❌ BAD:
GET  /getUser?id=123                  # Verb in URL, not RESTful
POST /user/create                     # "create" is redundant (POST implies creation)
DELETE /users/deleteAll               # Destructive bulk operation in REST is risky
GET  /api/v1/getUserOrders/123        # Camel case in URL, confusing structure
```

**Rules:**
- Use nouns, not verbs (resources, not actions)
- Lowercase with hyphens: `/user-profiles/{id}`
- Nested resources for containment: `/users/{id}/orders/{order_id}`
- Use plural nouns: `/users`, `/products`, `/orders`
- Exception: actions that don't fit CRUD: `POST /orders/{id}/cancel`, `POST /users/{id}/activate`

### Request/Response Design

```json
// ✅ GOOD: Consistent, structured, with metadata
GET /users/123
HTTP 200 OK
{
    "data": {
        "id": 123,
        "name": "Alice Smith",
        "email": "alice@example.com",
        "created_at": "2024-01-15T10:00:00Z"
    },
    "meta": {
        "request_id": "req_abc123"
    }
}

// ✅ GOOD: Errors
HTTP 422 Unprocessable Entity
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Request validation failed",
        "details": [
            { "field": "email", "message": "Invalid email format" },
            { "field": "age", "message": "Must be at least 18" }
        ]
    },
    "request_id": "req_abc123"
}

// ❌ BAD: Inconsistent error format
HTTP 200 OK
{ "success": false, "msg": "Invalid email" }
// Returns 200 for errors! Clients can't tell success from failure without parsing body.
```

---

## 🔷 GraphQL — Deep Dive

**GraphQL** is a query language for APIs and a runtime for executing those queries, developed by Facebook in 2012 and open-sourced in 2015.

### Core Concepts

**Schema:** Defines the types and operations available.

```graphql
type User {
    id: ID!
    name: String!
    email: String!
    orders(first: Int, after: String): OrderConnection!
    followers: [User!]!
}

type Order {
    id: ID!
    total: Float!
    status: OrderStatus!
    items: [OrderItem!]!
    user: User!
}

type Query {
    user(id: ID!): User
    users(first: Int, after: String): UserConnection!
    search(query: String!): [SearchResult!]!
}

type Mutation {
    createOrder(input: CreateOrderInput!): Order!
    cancelOrder(id: ID!): Order!
}

type Subscription {
    orderStatusChanged(orderId: ID!): Order!
}
```

**Query (reading data):**
```graphql
# Client specifies EXACTLY what fields it needs
query {
    user(id: "123") {
        name
        email
        orders(first: 5) {
            edges {
                node {
                    id
                    total
                    status
                    items {
                        productName
                        quantity
                    }
                }
            }
        }
    }
}
```

**Response — only requested fields returned:**
```json
{
    "data": {
        "user": {
            "name": "Alice Smith",
            "email": "alice@example.com",
            "orders": {
                "edges": [
                    {
                        "node": {
                            "id": "order_456",
                            "total": 99.99,
                            "status": "SHIPPED",
                            "items": [
                                { "productName": "Laptop", "quantity": 1 }
                            ]
                        }
                    }
                ]
            }
        }
    }
}
```

**No over-fetching!** If you only need `name` and `email`, only those are returned.

**Mutation (writing data):**
```graphql
mutation {
    createOrder(input: {
        userId: "123",
        items: [{ productId: "789", quantity: 2 }]
    }) {
        id
        total
        status
    }
}
```

**Subscription (real-time via WebSocket):**
```graphql
subscription {
    orderStatusChanged(orderId: "456") {
        id
        status
        updatedAt
    }
}
```

### The N+1 Problem in GraphQL

```graphql
query {
    orders(first: 100) {  # 1 query for 100 orders
        id
        total
        user {             # 100 queries for user per order! N+1!
            name
        }
    }
}
```

**Solution: DataLoader**

DataLoader batches all user-fetch requests within a single tick into one query.

```javascript
const userLoader = new DataLoader(async (userIds) => {
    // Called once with all unique user IDs
    const users = await db.query(
        'SELECT * FROM users WHERE id IN (?)',
        [userIds]
    );
    return userIds.map(id => users.find(u => u.id === id));
});

// In resolver:
const user = await userLoader.load(order.userId);
// DataLoader batches: 100 orders → 1 SQL query with 100 user IDs
```

### GraphQL Federation

For microservices, different services own different parts of the schema.

```
User Service:           Order Service:          Product Service:
  type User {             type Order {              type Product {
    id: ID!                 id: ID!                   id: ID!
    name: String!           total: Float!             name: String!
    email: String!          user: User          }     price: Float!
  }                       }
  
                        API Gateway / Router
                        ↑ merges schemas from all services
                        
Client queries:
{
    user(id: "123") {
        name               ← from User Service
        orders {
            total          ← from Order Service
            items {
                product {
                    name   ← from Product Service
                    price
                }
            }
        }
    }
}
```

The gateway (Apollo Federation, Netflix DGS) routes sub-queries to the appropriate service and assembles the result.

---

## ⚡ gRPC — Deep Dive

**gRPC (Google Remote Procedure Call)** is a high-performance RPC framework using HTTP/2 and Protocol Buffers.

### Protocol Buffers (Protobuf)

Language-agnostic IDL (Interface Definition Language) for defining service contracts.

```protobuf
// order.proto
syntax = "proto3";

package order;

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (Order);
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc ListOrders(ListOrdersRequest) returns (stream Order);  // Server streaming
    rpc UpdateOrderItems(stream OrderItem) returns (Order);   // Client streaming
    rpc OrderChat(stream ChatMessage) returns (stream ChatMessage); // Bidirectional
}

message CreateOrderRequest {
    string user_id = 1;
    repeated OrderItem items = 2;
}

message Order {
    string id = 1;
    string user_id = 2;
    OrderStatus status = 3;
    double total = 4;
    repeated OrderItem items = 5;
    google.protobuf.Timestamp created_at = 6;
}

enum OrderStatus {
    PENDING = 0;
    PAID = 1;
    SHIPPED = 2;
    DELIVERED = 3;
}

message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
    double unit_price = 3;
}
```

Generated code (TypeScript, Python, Go, Java, etc.) from this .proto file gives you:
- Strongly-typed client and server stubs
- Efficient binary serialization
- Documentation from the schema itself

### Why gRPC is Fast

**Protobuf binary encoding vs JSON text:**
```
JSON:   {"id":"order_456","total":99.99,"status":"SHIPPED"}
        → 55 bytes (text, verbose field names repeated every time)

Protobuf: 0a0b6f726465725f34353611713d0a d7a3705840
          → ~15 bytes (binary, field names encoded as numbers)
```

Protobuf is **3-10x smaller** than JSON. Faster to serialize/deserialize.

**HTTP/2:**
gRPC is built on HTTP/2, which provides:
- **Multiplexing:** Multiple RPC calls over a single TCP connection (no head-of-line blocking)
- **Header compression (HPACK):** Reduced overhead for repeated headers
- **Binary framing:** More efficient than HTTP/1.1 text parsing
- **Flow control and stream prioritization**

### gRPC Streaming Types

**Unary RPC:** Client sends one request, server sends one response. Standard RPC.

**Server Streaming:** Client sends one request, server sends a stream of responses.
```
Use case: Export large datasets, real-time price feeds, progress updates
Client: "Give me all orders for January"
Server: streams 100K orders one at a time (no need to load all into memory)
```

**Client Streaming:** Client sends a stream of requests, server sends one response.
```
Use case: Uploading large file in chunks, batch inserts
Client: streams 1000 sensor readings
Server: acknowledges all at once after processing
```

**Bidirectional Streaming:** Both sides send a stream of messages simultaneously.
```
Use case: Chat, gaming, collaborative editing
Client: sends keystrokes
Server: sends other players' positions
← Both streams operate concurrently →
```

### gRPC vs REST Comparison

| Feature | REST/HTTP | gRPC |
|---|---|---|
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 (always) |
| Data format | JSON (text) | Protobuf (binary) |
| Contract | OpenAPI/Swagger (optional) | Protobuf (mandatory) |
| Code generation | Optional | Core feature |
| Streaming | Limited (SSE, WebSocket separately) | Native (4 streaming modes) |
| Browser support | ✅ Native | ❌ Requires gRPC-Web proxy |
| Performance | Good | Excellent |
| Human-readable | ✅ JSON is readable | ❌ Binary is not |
| Error handling | HTTP status codes | gRPC status codes (richer) |
| Tooling | Excellent | Good (growing) |
| Use case | Public APIs, web apps | Internal microservice communication |

**When to use gRPC:**
- Service-to-service communication inside your infrastructure
- Low-latency, high-throughput requirements
- Polyglot systems (auto-generated clients in any language)
- Streaming data (sensor data, stock prices, real-time updates)

**When to use REST:**
- Public APIs that need to be browser-accessible
- Simple CRUD APIs where developer ergonomics matter
- External partners/customers who will consume your API

---

## 🔌 WebSockets

### The Problem with HTTP for Real-Time

HTTP is request-response. The client always initiates. The server cannot push data to the client.

For real-time features (chat, live scores, collaborative editing), HTTP alone requires:
- **Short polling:** Client asks "anything new?" every second → wasted requests
- **Long polling:** Client holds connection open until server has data → complex, inefficient
- **WebSockets:** True bidirectional, full-duplex communication over a single TCP connection

### The WebSocket Handshake

WebSocket starts as HTTP and upgrades:

```
Client → Server:
GET /chat HTTP/1.1
Host: example.com
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

← Connection upgraded; bidirectional communication begins →
```

After the handshake, both sides can send **frames** at any time:

```javascript
// Client (browser)
const ws = new WebSocket('wss://chat.example.com/ws');

ws.onopen = () => {
    ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    displayMessage(msg);
};

// Send message
ws.send(JSON.stringify({ type: 'message', text: 'Hello!', room: 'general' }));
```

```python
# Server (Python with websockets)
async def handler(websocket):
    room = await websocket.recv()
    
    async for message in websocket:
        # Broadcast to all in room
        for client in room_clients[room]:
            await client.send(message)
```

### WebSockets at Scale

A single WebSocket connection stays open for the user's entire session. With 1 million concurrent users, you need 1 million open connections.

**Challenges:**
- **Stateful:** Each user is pinned to a specific server (connection is alive)
- **Memory:** Each connection uses ~50KB RAM → 1M connections = 50GB RAM per server
- **Horizontal scaling:** Load balancers must support sticky sessions for WebSockets

**Solutions:**
- Use a message bus (Redis Pub/Sub) to propagate messages between servers
- Purpose-built solutions: Pusher, Ably, AWS API Gateway WebSockets

```
User A (Server 1) sends message to room "general"
Server 1 publishes to Redis: "general" channel
Server 2 (subscribed) receives → sends to its users in "general"
Server 3 (subscribed) receives → sends to its users in "general"
```

---

## 📡 Server-Sent Events (SSE)

SSE is a simpler alternative to WebSockets for **server-to-client** unidirectional streaming.

```
Client opens: GET /events HTTP/1.1
Accept: text/event-stream

Server responds with streaming body:
Content-Type: text/event-stream

data: {"type":"price_update","AAPL":175.32}

data: {"type":"price_update","GOOG":140.15}

event: alert
data: {"type":"market_close"}

```

**Characteristics:**
- HTTP/1.1 compatible (unlike WebSockets, works through proxies easily)
- Automatic reconnection (built into the browser's EventSource API)
- Unidirectional: server → client only
- Text-based (not binary)

```javascript
// Browser
const eventSource = new EventSource('/events');
eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updatePriceDisplay(data);
};
```

**Use SSE when:** Server needs to push updates to clients (notifications, live scores, real-time dashboards) and clients don't need to send data back (or can do so via separate REST calls).

**Use WebSockets when:** True bidirectional communication is needed (chat, collaborative editing, gaming).

---

## 🔄 Long Polling vs Short Polling vs SSE vs WebSockets

| Method | Bidirectional | Complexity | Latency | HTTP Compatible | Use Case |
|---|---|---|---|---|---|
| Short Polling | ❌ | Low | High (poll interval) | ✅ | Simple notifications, low-frequency |
| Long Polling | ❌ | Medium | Low (near real-time) | ✅ | Chat (legacy), compatibility needed |
| SSE | ❌ (server→client) | Low | Very low | ✅ | Push notifications, live feeds |
| WebSocket | ✅ | Medium | Very low | Upgrade needed | Chat, gaming, collaborative |

---

## 🚪 API Gateway Pattern

An **API Gateway** is a server that acts as the entry point for all clients, routing to the appropriate backend service.

```
Mobile App ─────────────────────────────────────────────────────────────►
Web App    ──────────────────────────────────────────────────────────────►  ┌─────────────┐
Third-party Partners ──────────────────────────────────────────────────────►│ API Gateway │
                                                                            └──────┬──────┘
                                                        ┌───────────────────────────┤
                                                        │           │               │
                                                ┌───────▼──┐  ┌─────▼───┐  ┌──────▼───┐
                                                │  User    │  │  Order  │  │ Product  │
                                                │ Service  │  │ Service │  │ Service  │
                                                └──────────┘  └─────────┘  └──────────┘
```

### What an API Gateway Does

**1. Authentication/Authorization**
- Validate JWT tokens, API keys before requests reach services
- Services trust the gateway; don't need to validate auth themselves

**2. Rate Limiting**
- Enforce per-user or per-API-key limits before traffic hits services
- Protects services from abuse and DDoS

**3. Request Routing**
- `GET /users/*` → User Service
- `GET /orders/*` → Order Service
- `POST /payments/*` → Payment Service

**4. Load Balancing**
- Distribute requests across multiple instances of each service

**5. SSL Termination**
- Handle HTTPS at the gateway; services use HTTP internally

**6. Request/Response Transformation**
- Rename fields, combine responses, add headers

**7. Logging and Metrics**
- Centralized logging of all API traffic

**8. Caching**
- Cache responses for GET requests

**9. Protocol Translation**
- External clients use REST; gateway translates to gRPC for internal services

### API Gateway Products

| Product | Type | Best For |
|---|---|---|
| AWS API Gateway | Managed cloud | Lambda/AWS-native APIs |
| AWS Application Load Balancer | Managed cloud | ECS/EKS HTTP routing |
| Kong | Open source / cloud | Plugin-based, extensible |
| Nginx | Open source | High-performance, simple routing |
| Envoy | Open source | Service mesh, microservices |
| Traefik | Open source | Kubernetes-native, dynamic discovery |
| Apigee (Google) | Managed cloud | Enterprise API management |

---

## 🚦 Rate Limiting Algorithms

### Token Bucket

A bucket holds tokens. Each request consumes one token. Tokens are added at a fixed rate. If bucket is empty, request is rejected.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

User sends 50 requests instantly → bucket has 50 tokens remaining ✓
User sends 60 more instantly → only 50 tokens → 60th+ requests rejected ✗
After 5 seconds → 50 tokens refilled → user can make 50 more requests
```

**Allows bursting** up to the bucket capacity. Natural behavior for APIs.

### Leaky Bucket

Requests enter a queue (the "bucket") and are processed at a fixed rate. If the bucket is full, new requests are dropped.

```
Queue capacity: 100 requests
Processing rate: 10 requests/second

Even if 1000 requests arrive, only 10/second are processed → smooth, steady output
Excess requests are dropped (or queued if queue has space)
```

**Smooths bursty traffic.** No bursting allowed — output is always at fixed rate.

### Fixed Window Counter

Count requests within a fixed time window (e.g., minute). If count exceeds limit, reject.

```
Window: 00:00 - 01:00 (1 minute)
Limit: 100 requests/minute

Requests at 00:59: 99 (ok)
Requests at 01:00: window resets → 100 more requests allowed!
Attack: send 99 requests at 00:59 + 100 at 01:00 = 199 requests in 2 seconds!
```

**Vulnerability:** Burst at window boundary.

### Sliding Window

Maintains a count of requests in the last N seconds (rolling window), not a fixed time slice.

**Sliding Window Log:** Store timestamp of each request. Count timestamps within window.
- ✅ Accurate
- ❌ Memory: stores timestamp per request

**Sliding Window Counter (hybrid):** Approximate using two fixed windows.
```
Current count ≈ (prev_window_count × overlap_ratio) + curr_window_count

At 01:15 with 1-minute window:
overlap_ratio = (60-15)/60 = 0.75
count = (prev_window_count × 0.75) + curr_window_count
```

### Distributed Rate Limiting with Redis

```python
# Sliding window rate limiting using Redis
import redis
import time

r = redis.Redis()

def is_rate_limited(user_id: str, limit: int, window_seconds: int) -> bool:
    key = f"ratelimit:{user_id}"
    now = time.time()
    window_start = now - window_seconds
    
    pipe = r.pipeline()
    # Remove old requests outside the window
    pipe.zremrangebyscore(key, 0, window_start)
    # Count requests in window
    pipe.zcard(key)
    # Add current request
    pipe.zadd(key, {str(now): now})
    # Set key expiration
    pipe.expire(key, window_seconds)
    
    results = pipe.execute()
    request_count = results[1]
    
    return request_count >= limit  # True = rate limited

# Usage
if is_rate_limited("user:123", limit=100, window_seconds=60):
    return "429 Too Many Requests"
```

**Rate limiting response headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1705315260
Retry-After: 60
```

---

## 🔢 API Versioning

Changes to APIs break existing clients. Versioning strategies:

**URI Path Versioning (Most Common)**
```
GET /v1/users/123
GET /v2/users/123  (v2 has new response format)
```
- ✅ Explicit, easy to cache, easy to route
- ❌ URI "pollution", resources logically same but in different paths

**Header Versioning**
```
GET /users/123
Accept: application/vnd.myapi.v2+json
```
- ✅ Clean URIs
- ❌ Can't be bookmarked/linked directly, harder to test in browser

**Query Parameter Versioning**
```
GET /users/123?version=2
```
- ✅ Easy to test
- ❌ Caching issues (version param changes cache key)

**Best practice for backwards compatibility:**
- Add new fields instead of removing old ones
- Mark deprecated fields in documentation
- Support old format for at least 6-12 months after deprecating
- Use `Sunset` header to indicate when a version will be removed

---

## 💬 Real-Life Scenario: Building a Chat Application

**The situation:** Building a real-time chat app (like Slack). Different parts need different API styles.

**REST API for:**
```
User management:     GET /users, POST /users, PATCH /users/{id}
Channel management:  POST /channels, GET /channels
Message history:     GET /channels/{id}/messages?before=cursor&limit=50
File uploads:        POST /files (multipart)
```

**WebSockets for:**
```
Real-time message delivery:
  Client → WS → Server: { type: "message", channel_id: "123", text: "Hello!" }
  Server → WS → All clients in channel: { type: "new_message", ... }
  
Typing indicators:
  Client → WS → Server: { type: "typing_start", channel_id: "123" }
  Server → WS → Others: { type: "user_typing", user_id: "456" }
  
Presence (online/offline):
  WS connection open = online; WS close = offline
```

**gRPC for:**
```
Internal service communication:
  MessageService ──gRPC──► NotificationService (send push notification)
  MessageService ──gRPC──► SearchIndexService (index message for search)
```

**SSE for:**
```
Notification sidebar:
  Client subscribes to: GET /notifications/stream
  Server pushes: { type: "mention", message_id: "789", channel: "general" }
```

**API Gateway configuration:**
```
POST /api/* → Route to appropriate microservice
WebSocket /ws → WebSocket Gateway Service (maintains WS connections)
GET /files/* → CDN (static assets bypass backend entirely)

Rate limits:
  POST /messages: 60/minute per user (1 per second)
  GET endpoints: 1000/minute per user
```

---

## 🔥 Big Tech Story: Netflix's GraphQL Federation

### The Problem (2019)

Netflix runs on dozens of microservices: user service, catalog service, playback service, recommendation service, etc. Their mobile and web clients needed to aggregate data from many of these services per page load.

**The old approach:**
Each client team (iOS, Android, web, TV) built its own **Backend for Frontend (BFF)** — a thin aggregation layer specific to their needs.

```
iOS BFF → User Service, Catalog Service, Recommendation Service
Android BFF → (same services, different queries)
Web BFF → (same services, different queries)
Smart TV BFF → (same services, different queries)
```

Problems:
- 4 BFFs duplicating business logic
- If catalog service changed, 4 BFF teams had to update
- Frontend teams needed to know the internals of every backend service
- Huge amount of duplicated code and coordination overhead

### The GraphQL Solution

Netflix built a **federated GraphQL layer** where each microservice defined its own GraphQL schema, and a gateway merged them.

```
                     ┌──────────────────────┐
All Clients ────────►│   Federated GraphQL  │
                     │      Gateway         │
                     └──────────────────────┘
                         │           │
                   ┌─────▼──┐   ┌───▼──────┐
                   │  User  │   │ Catalog  │
                   │Service │   │ Service  │
                   │(schema)│   │(schema)  │
                   └────────┘   └──────────┘
                         │           │
               ┌──────────────────────────────┐
               │ Recommendation │  Playback   │
               │    Service     │  Service    │
               │   (schema)     │  (schema)   │
               └──────────────────────────────┘
```

Each service owns a piece of the schema:
```graphql
# User Service:
type User @key(fields: "id") {
    id: ID!
    name: String!
    email: String!
}

# Catalog Service (extends User with watchlist):
extend type User @key(fields: "id") {
    id: ID! @external
    watchlist: [Movie!]!
}

# Recommendation Service (extends User with recommendations):
extend type User @key(fields: "id") {
    id: ID! @external
    recommendations: [Movie!]!
}
```

**The client now makes ONE query** that seamlessly spans multiple services:
```graphql
query {
    user(id: "me") {
        name           # from User Service
        watchlist {    # from Catalog Service
            title
            posterUrl
        }
        recommendations {  # from Recommendation Service
            title
            matchScore
        }
    }
}
```

**Results:**
- Eliminated 4 separate BFF layers
- 40% reduction in data over-fetching
- Frontend teams could add new data requirements without involving backend teams
- New services could add their data to the unified schema independently
- Faster feature development: teams work autonomously

**Netflix open-sourced their solution.** The concept of GraphQL Federation became a standard (Apollo Federation) and is now widely adopted by companies like Shopify, GitHub, and Airbnb.

**The lesson:** GraphQL Federation is most valuable when you have many client platforms consuming data from many services. The "single unified graph" dramatically reduces the coordination tax between teams.

---

## ⚖️ Trade-offs & Drawbacks

### REST
| Pro | Con |
|---|---|
| Simple, universal | Over-fetching / under-fetching |
| Browser-native | Multiple round trips for related data |
| Excellent caching (GET is cacheable) | No standard for real-time |
| Stateless | Versioning is painful |

### GraphQL
| Pro | Con |
|---|---|
| Exact data requested | Complex caching (POST requests, unique queries) |
| Fewer round trips | N+1 problem (solved with DataLoader) |
| Strong typing | Learning curve for teams |
| Great for complex UIs | Potentially expensive queries if not rate-limited |
| | Security: clients can request anything (depth limiting needed) |

### gRPC
| Pro | Con |
|---|---|
| High performance | Not browser-native (needs gRPC-Web) |
| Streaming native | Steeper learning curve |
| Strong typing | Binary not human-readable (debugging harder) |
| Code generation | Tooling less mature than REST |

### WebSockets
| Pro | Con |
|---|---|
| True real-time, low latency | Stateful (scaling is complex) |
| Bidirectional | Firewall/proxy issues |
| Efficient (no HTTP overhead per message) | Connection management overhead |
| | No built-in request-response matching |

---

## ✅ Best Practices

1. **Use REST for public APIs, gRPC for internal services.** REST is universally understood; gRPC gives performance for high-frequency internal calls.

2. **Version your API from day one.** Start with `/v1/`. It's easier to start versioned than to add versioning later.

3. **Design for backwards compatibility.** Adding fields is safe. Removing fields or changing types is breaking. Mark deprecated fields; don't remove them immediately.

4. **Use cursor-based pagination** for all list endpoints in production. Offset pagination breaks under concurrent writes.

5. **Always return meaningful error responses** with machine-readable error codes (not just HTTP status + message string). Include a request ID for tracing.

6. **Rate limit all public endpoints.** Use Redis for distributed rate limiting. Return proper `429 Too Many Requests` with `Retry-After` header.

7. **Document your API with OpenAPI (Swagger)** for REST or gRPC protobuf. Auto-generate documentation; don't write it by hand.

8. **Use an API Gateway** for authentication, rate limiting, and routing. Don't implement these in every service.

9. **Make APIs idempotent where possible.** `PUT` and `DELETE` should be idempotent. For `POST`, consider accepting an idempotency key from the client.

10. **GraphQL: protect against expensive queries.** Use query depth limiting, query complexity analysis, and persisted queries in production.

11. **WebSockets: always handle reconnection.** Network hiccups happen. Clients must implement exponential backoff reconnection with state recovery.

12. **gRPC: implement deadlines and cancellation.** Always set deadlines on RPC calls. Without them, a slow service can cascade failures indefinitely.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **REST** | Resource-based, stateless; use HTTP methods semantically; cursor pagination for lists |
| **GraphQL** | Client specifies exact data; eliminates over/under-fetching; use DataLoader for N+1 |
| **gRPC** | Binary Protobuf + HTTP/2; fastest for internal services; native streaming support |
| **WebSockets** | Full-duplex real-time; stateful (use Redis pub/sub for scaling) |
| **SSE** | Server→client push; HTTP-compatible; simpler than WebSockets for one-way streaming |
| **API Gateway** | Single entry point; handles auth, rate limiting, routing, SSL termination |
| **Rate Limiting** | Token bucket for APIs; Redis for distributed state; return proper 429 responses |
| **Versioning** | Start versioned (v1); add fields not remove them; sunset deprecated versions |
| **Netflix's Lesson** | GraphQL Federation eliminates BFF duplication for multi-platform apps at scale |
| **Design principle** | APIs are contracts; design for your clients' needs, not your data model |

---

*Next: [07_CDN.md](07_CDN.md) — Content Delivery Networks: caching at the edge for global performance.*
