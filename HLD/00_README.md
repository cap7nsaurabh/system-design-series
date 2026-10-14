# 📚 System Design Series — Complete High-Level Design (HLD) Curriculum

[🏠 Master Index](../README.md) | [⚡ Backend Problems](../Backend%20Problems/00-index.md) | [🧱 Low-Level Design (LLD)](../LLD/00-index.md) | [📬 Message Queues](../Message%20Queues/00-index.md) | [🧩 Microservices](../Microservices%20Design/00-index.md) | [🍃 Spring Boot](../Spring%20Boot/00-course-index.md) | [💾 Databases](../Databases/00-index.md)

---

> **Role:** Principal Software Engineer  
> **Focus:** Highly scalable, available, and consistent distributed backend systems  
> **Format:** Deep-dive markdown guides with real-world examples, Big Tech stories, trade-off analysis, and code

---

## 🗺️ What This Series Covers

This is an **18-document, production-grade curriculum** on distributed systems and backend engineering. Each document is a standalone deep dive on one core topic, written at the level of a senior/principal engineer at a top tech company.

**Every document includes:**
- ✅ Core concepts explained from first principles
- ✅ Why the technology exists (the problem it solves)
- ✅ When to use it (and when NOT to)
- ✅ Real-life scenario / concrete example
- ✅ 🔥 **Big Tech Story** — how Google, Netflix, Amazon, Facebook, Uber, or WhatsApp faced and solved a real engineering challenge
- ✅ Trade-offs and drawbacks
- ✅ Best practices for production
- ✅ Architecture diagrams (Mermaid/ASCII)
- ✅ Code examples and configuration snippets
- ✅ Comparison tables

---

## 📂 Document Index

| # | Document | Key Topics | Big Tech Story |
|---|---|---|---|
| 01 | [DNS & Networking Fundamentals](01_DNS_and_Networking_Fundamentals.md) | DNS resolution, record types, GeoDNS, CDN routing, BGP | AWS Route 53 saves Netflix |
| 02 | [Load Balancers](02_Load_Balancers.md) | L4 vs L7, algorithms, sticky sessions, health checks, GSLB | GitHub DDoS mitigation |
| 03 | [Caching](03_Caching.md) | Cache-aside, write-through, Redis internals, stampede, hot keys | Facebook Memcache at scale (TAO) |
| 04 | [Databases](04_Databases.md) | ACID, isolation levels, B-Tree, sharding, replication, CAP | Discord: MongoDB → Cassandra → ScyllaDB |
| 05 | [Message Queues & Event-Driven](05_Message_Queues_and_Event_Driven_Architecture.md) | Kafka internals, Saga, CQRS, outbox pattern, delivery semantics | LinkedIn's Kafka origin story |
| 06 | [API Design](06_API_Design.md) | REST, GraphQL, gRPC, WebSockets, SSE, API Gateway, rate limiting | Netflix GraphQL Federation |
| 07 | [Content Delivery Networks](07_CDN.md) | CDN architecture, Cache-Control headers, edge computing, video CDN | Cloudflare BGP hijacking incident |
| 08 | [Microservices Architecture](08_Microservices.md) | DDD, circuit breaker, service mesh, Kubernetes, Strangler Fig | Amazon's Two-Pizza Team & API Mandate |
| 09 | [Scalability Patterns](09_Scalability_Patterns.md) | Consistent hashing, auto-scaling, CQRS reads, cell-based arch | WhatsApp: 900M users, 50 engineers |
| 10 | [Distributed Systems Fundamentals](10_Distributed_Systems_Fundamentals.md) | CAP, PACELC, Raft, Paxos, vector clocks, gossip, 2PC | Google Chubby → Apache ZooKeeper |
| 11 | [Storage Systems](11_Storage_Systems.md) | Block, file, object storage, S3, LSM vs B-tree, WAL, erasure coding | Facebook Haystack (needle in a haystack) |
| 12 | [Security](12_Security.md) | OAuth 2.0, JWT, mTLS, Zero Trust, Vault, TLS, OWASP Top 10 | Uber 2022 data breach |
| 13 | [Observability](13_Observability.md) | Metrics, logs, traces, SLOs, error budgets, chaos engineering | Netflix Chaos Monkey & Simian Army |
| 14 | [Data Pipelines & Analytics](14_Data_Pipelines.md) | Spark, Flink, Lambda/Kappa, Delta Lake, CDC, ClickHouse | Uber's real-time analytics platform |
| 15 | [High Availability & Disaster Recovery](15_High_Availability_and_Disaster_Recovery.md) | RTO/RPO, multi-region, Aurora Global, backup strategies, graceful degradation | AWS us-east-1 recurring outages |
| 16 | [System Design Case Studies](16_System_Design_Case_Studies.md) | URL shortener, Twitter, YouTube, Uber, WhatsApp — full end-to-end designs | All combined |
| 17 | [Algorithms & Data Structures](17_Algorithms_and_Data_Structures.md) | Rate limiting (all 5 algorithms), Consistent Hashing, Bloom Filter, Count-Min Sketch, HyperLogLog, Top-K, Trie, Geohash, LRU/LFU, Leader Election, Merkle Trees | — |
| 18 | [Scaling from 0 to Billions](18_Scaling_From_Zero_to_Billions.md) | 0→100→10K→500K→10M→100M→1B users: architecture decisions, costs, technology choices at each stage | Instagram (13 engineers, 1B users) & Stack Overflow (9 servers, 1.5B page views/month) |

---

## 🎓 Recommended Learning Path

### 🟢 Start Here — Foundations (Week 1-2)

These are the building blocks that everything else depends on.

1. **[01: DNS & Networking](01_DNS_and_Networking_Fundamentals.md)** — How requests get from user to your server
2. **[02: Load Balancers](02_Load_Balancers.md)** — Distributing traffic at scale
3. **[03: Caching](03_Caching.md)** — The most powerful performance tool in your arsenal
4. **[04: Databases](04_Databases.md)** — SQL, NoSQL, ACID, sharding — the core of every system

### 🟡 Communication & APIs (Week 3)

How components talk to each other.

5. **[05: Message Queues](05_Message_Queues_and_Event_Driven_Architecture.md)** — Async communication, Kafka, event-driven design
6. **[06: API Design](06_API_Design.md)** — REST, GraphQL, gRPC, WebSockets
7. **[07: CDN](07_CDN.md)** — Global content delivery

### 🟠 Architecture Patterns (Week 4)

How to structure large systems.

8. **[08: Microservices](08_Microservices.md)** — Breaking the monolith
9. **[09: Scalability Patterns](09_Scalability_Patterns.md)** — Growing from 10K to 100M users

### 🔴 Advanced Distributed Systems (Week 5)

The hard theoretical and practical foundations.

10. **[10: Distributed Systems Fundamentals](10_Distributed_Systems_Fundamentals.md)** — CAP theorem, consensus, consistency
11. **[11: Storage Systems](11_Storage_Systems.md)** — How data is physically stored

### 🟣 Cross-Cutting Concerns (Week 6)

Essential for production systems.

12. **[12: Security](12_Security.md)** — Auth, encryption, Zero Trust
13. **[13: Observability](13_Observability.md)** — Monitoring, alerting, tracing, chaos engineering
14. **[14: Data Pipelines](14_Data_Pipelines.md)** — Batch, streaming, analytics
15. **[15: High Availability & DR](15_High_Availability_and_Disaster_Recovery.md)** — Never go down

### 🏆 Algorithms & Scaling (Week 7-8)

Apply everything you've learned.

16. **[16: Case Studies](16_System_Design_Case_Studies.md)** — Design Twitter, YouTube, Uber, WhatsApp from scratch
17. **[17: Algorithms & Data Structures](17_Algorithms_and_Data_Structures.md)** — Rate limiting, Bloom filters, HyperLogLog, Trie, Geohash, LRU/LFU with full implementations
18. **[18: Scaling from 0 to Billions](18_Scaling_From_Zero_to_Billions.md)** — The complete architectural evolution: single server → global distributed system

---

## 🧩 Technology Coverage Map

### Databases & Storage
| Technology | Document |
|---|---|
| PostgreSQL (ACID, indexing, sharding) | 04, 11 |
| MySQL / Aurora | 04, 15 |
| MongoDB | 04 |
| Cassandra / ScyllaDB | 04, 10, 16 |
| Redis (Caching, pub/sub, data structures) | 03, 09 |
| Amazon DynamoDB | 04, 10 |
| Amazon S3 / Object Storage | 11, 16 |
| Elasticsearch | 04, 16 |
| ClickHouse | 14, 16 |
| Apache Druid / Pinot | 14 |
| Google Spanner | 04, 10 |
| CockroachDB | 04, 10 |

### Messaging & Streaming
| Technology | Document |
|---|---|
| Apache Kafka | 05, 14 |
| RabbitMQ | 05 |
| AWS SQS / SNS | 05 |
| Apache Flink | 14 |
| Apache Spark | 14 |
| Kafka Streams | 05, 14 |
| Debezium (CDC) | 14 |

### Infrastructure
| Technology | Document |
|---|---|
| Docker & Kubernetes | 08 |
| AWS (EC2, EBS, EFS, S3, RDS, Aurora, ElastiCache, ALB, CloudFront, Route 53) | All |
| CDN (Cloudflare, Fastly, Akamai, CloudFront) | 07 |
| Nginx / HAProxy | 02 |
| Istio / Envoy (Service Mesh) | 08 |
| HashiCorp Vault | 12 |

### Observability
| Technology | Document |
|---|---|
| Prometheus + Grafana | 13 |
| Jaeger / Zipkin (Distributed Tracing) | 08, 13 |
| OpenTelemetry | 08, 13 |
| ELK Stack / Loki | 13 |
| Chaos Monkey / Chaos Engineering | 13 |

### API & Communication
| Technology | Document |
|---|---|
| REST APIs | 06 |
| GraphQL + Federation | 06 |
| gRPC + Protobuf | 06 |
| WebSockets | 06, 16 |
| OAuth 2.0 + JWT | 12 |
| mTLS | 08, 12 |

### Data Engineering
| Technology | Document |
|---|---|
| Apache Spark | 14 |
| Apache Flink | 14 |
| Delta Lake / Apache Iceberg | 14 |
| dbt | 14 |
| Apache Pinot | 14 |
| BigQuery / Snowflake / Redshift | 14 |
| Change Data Capture (Debezium) | 14 |

---

## 🧠 Key Concepts Quick Reference

### Consistency
| Concept | Document | One-Line |
|---|---|---|
| ACID | 04, 10 | All-or-nothing transactions with durability |
| BASE | 04, 10 | Basically Available, Eventually Consistent |
| CAP Theorem | 10 | Choose C or A when partition occurs |
| PACELC | 10 | C vs A during partition; L vs C normally |
| Linearizability | 10 | Strongest consistency — global real-time order |
| Eventual Consistency | 03, 04, 10 | Replicas converge given no new updates |

### Scalability
| Concept | Document | One-Line |
|---|---|---|
| Consistent Hashing | 09 | Minimize key remapping when adding/removing nodes |
| Horizontal Scaling | 09 | More machines, stateless services |
| Sharding | 04, 09 | Partition data across DB instances |
| Read Replicas | 04, 09 | Distribute reads; one writer |
| CQRS | 05, 09 | Separate read/write models |
| Fan-out on Write | 16 (Twitter) | Pre-compute timelines for followers |
| Cell-Based Architecture | 09 | Isolate blast radius with independent cells |

### Performance
| Concept | Document | One-Line |
|---|---|---|
| Cache-aside | 03 | Load into cache on miss |
| Write-through | 03 | Write to cache and DB simultaneously |
| Cache stampede | 03 | Many requests miss cache at once → fix with mutex/early refresh |
| B-Tree | 04, 11 | Read-optimized index structure |
| LSM Tree | 04, 11 | Write-optimized; Cassandra, RocksDB |
| CDN + Edge | 07 | Serve content from the nearest PoP |
| Bloom filter | 11 | Probabilistic: "definitely not in set" to skip disk reads |

### Resilience
| Concept | Document | One-Line |
|---|---|---|
| Circuit Breaker | 08, 09 | Stop calling a failing service; fail fast |
| Retry + Jitter | 08 | Retry transient failures with random delay |
| Bulkhead | 08 | Isolate resource pools between services |
| Graceful Degradation | 15 | Partial failure → reduced functionality, not total failure |
| Chaos Engineering | 13 | Break production on purpose to find weaknesses |
| Multi-region Active-Active | 15 | Multiple regions serve live traffic; near-zero failover |

---

## 🏢 Big Tech Stories Index

| Story | Company | Document | Topic |
|---|---|---|---|
| Route 53 + Health Checks save Netflix | AWS / Netflix | 01 | GeoDNS, DNS failover |
| DDoS mitigation & custom load balancer | GitHub | 02 | L7 load balancing, DDoS |
| Memcache at scale (TAO) | Facebook | 03 | Distributed caching |
| MongoDB → Cassandra → ScyllaDB | Discord | 04 | Database migration |
| Kafka origin story | LinkedIn | 05 | Message queue design |
| GraphQL Federation | Netflix | 06 | API architecture |
| BGP hijacking incident | Cloudflare | 07 | CDN, anycast, BGP |
| Two-pizza team & API mandate | Amazon | 08 | Microservices, organizational design |
| 900M users with 50 engineers | WhatsApp | 09 | Erlang, scalability |
| Chubby → ZooKeeper → etcd | Google | 10 | Distributed consensus |
| Haystack photo storage | Facebook | 11 | Storage optimization |
| 2022 data breach | Uber | 12 | Security, social engineering |
| Chaos Monkey & Simian Army | Netflix | 13 | Chaos engineering, resilience |
| Real-time analytics platform | Uber | 14 | Pinot, real-time OLAP |
| us-east-1 recurring outages | AWS | 15 | HA, multi-region, cascade failure |
| All systems (URL, Twitter, YouTube, Uber, WhatsApp) | Multiple | 16 | End-to-end system design |

---

## 💡 How to Get the Most From This Series

### For Interview Prep
1. Read documents 01-09 first (most commonly asked topics)
2. For each topic: understand the problem, the solution, and the trade-offs
3. Practice drawing the architecture diagrams from memory
4. Practice the estimation (back-of-envelope) calculations in document 16
5. Use document 16 to practice full case study walkthroughs

### For Day-to-Day Engineering
1. Use as a reference when evaluating technology choices
2. Read the Big Tech Stories to understand why certain patterns exist
3. Use the best practices sections as checklists for production readiness

### For Team Education
1. Assign one document per week as team reading
2. Discuss the trade-offs in your specific context
3. Run a "system design review" of one of your existing systems using the frameworks here
4. Identify the Big Tech Story patterns in your own architecture

---

## 📝 Series Statistics

| Metric | Count |
|---|---|
| Total documents | 18 |
| Big Tech stories | 18 |
| Technologies covered | 60+ |
| Algorithms covered | 15+ (with full Python implementations) |
| Companies referenced | Amazon, Google, Netflix, Facebook, Uber, WhatsApp, Discord, LinkedIn, GitHub, Cloudflare, Twitter, Instagram, Stack Overflow |
| Core design patterns | 40+ |

---

*Written from the perspective of a Principal Software Engineer with deep knowledge of cloud infrastructure, distributed systems theory, and production operations at scale.*

*Last updated: 2026*
