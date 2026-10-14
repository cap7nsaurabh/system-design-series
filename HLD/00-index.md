# High-Level Design (HLD) Master Index

[🏠 Master Index](../README.md) | [⚡ Backend Problems](../Backend%20Problems/00-index.md) | [🧱 Low-Level Design (LLD)](../LLD/00-index.md) | [📬 Message Queues](../Message%20Queues/00-index.md) | [🧩 Microservices](../Microservices%20Design/00-index.md) | [🍃 Spring Boot](../Spring%20Boot/00-course-index.md) | [💾 Databases](../Databases/00-index.md)

---

Welcome to the **High-Level Design (HLD)** curriculum. This module contains 18 comprehensive, production-grade architectural deep dives on distributed systems, scalability, cloud infrastructure, and real-world system design interview case studies.

> For the complete curriculum guide, learning paths, technology coverage maps, and Big Tech stories index, see [**00_README.md**](./00_README.md).

---

## 📚 Table of Contents

| # | Topic | Document | Focus Areas | Big Tech Architecture Story |
|---|---|---|---|---|
| 01 | **DNS & Networking** | [01_DNS_and_Networking_Fundamentals.md](./01_DNS_and_Networking_Fundamentals.md) | DNS resolution, Anycast, GeoDNS, BGP routing, CDN edge resolution | AWS Route 53 saves Netflix |
| 02 | **Load Balancers** | [02_Load_Balancers.md](./02_Load_Balancers.md) | L4 vs L7, consistent hashing, health checks, sticky sessions, Maglev | GitHub DDoS mitigation |
| 03 | **Caching** | [03_Caching.md](./03_Caching.md) | Cache-aside, Write-through, Write-back, Cache stampede, Redis, TAO | Facebook Memcache at scale |
| 04 | **Databases** | [04_Databases.md](./04_Databases.md) | ACID vs BASE, B-Tree vs LSM, sharding, replication topologies, CAP | Discord: MongoDB → Cassandra → ScyllaDB |
| 05 | **Message Queues** | [05_Message_Queues_and_Event_Driven_Architecture.md](./05_Message_Queues_and_Event_Driven_Architecture.md) | Kafka, RabbitMQ, SQS, Saga orchestration, CQRS, Transactional Outbox | LinkedIn's Kafka origin story |
| 06 | **API Design** | [06_API_Design.md](./06_API_Design.md) | REST, GraphQL Federation, gRPC, Protobuf, WebSockets, SSE | Netflix GraphQL Federation |
| 07 | **Content Delivery Networks** | [07_CDN.md](./07_CDN.md) | Edge caching, PoP topologies, Cache-Control, Video on demand | Cloudflare BGP route leak incident |
| 08 | **Microservices** | [08_Microservices.md](./08_Microservices.md) | Domain-Driven Design, Circuit Breakers, Service Mesh, Envoy, mTLS | Amazon Two-Pizza Teams & API Mandate |
| 09 | **Scalability Patterns** | [09_Scalability_Patterns.md](./09_Scalability_Patterns.md) | Horizontal scaling, Consistent Hashing, Cell-based architectures | WhatsApp: 900M users with 50 engineers |
| 10 | **Distributed Systems Fundamentals** | [10_Distributed_Systems_Fundamentals.md](./10_Distributed_Systems_Fundamentals.md) | CAP, PACELC, Raft consensus, Paxos, Vector Clocks, 2PC, Gossip | Google Chubby → ZooKeeper |
| 11 | **Storage Systems** | [11_Storage_Systems.md](./11_Storage_Systems.md) | Block, File, and Object storage, S3 internals, Haystack, Erasure coding | Facebook Haystack photo storage |
| 12 | **Security** | [12_Security.md](./12_Security.md) | OAuth 2.0, OpenID Connect, JWT, Zero Trust, Vault, mTLS, OWASP | Uber 2022 internal data breach |
| 13 | **Observability** | [13_Observability.md](./13_Observability.md) | Metrics (Prometheus), Distributed Tracing (Jaeger), Logs, SLOs | Netflix Chaos Monkey & Simian Army |
| 14 | **Data Pipelines** | [14_Data_Pipelines.md](./14_Data_Pipelines.md) | Batch vs Streaming, Spark, Flink, Kafka Streams, CDC, ClickHouse | Uber real-time analytics platform |
| 15 | **High Availability & DR** | [15_High_Availability_and_Disaster_Recovery.md](./15_High_Availability_and_Disaster_Recovery.md) | RTO / RPO, Multi-region Active-Active, Split-brain mitigation | AWS us-east-1 cascade outages |
| 16 | **System Design Case Studies** | [16_System_Design_Case_Studies.md](./16_System_Design_Case_Studies.md) | URL Shortener, Twitter Feed, YouTube Streaming, Uber Dispatch, WhatsApp | Full end-to-end architectures |
| 17 | **Algorithms & Data Structures** | [17_Algorithms_and_Data_Structures.md](./17_Algorithms_and_Data_Structures.md) | Rate Limiting, Bloom Filters, Count-Min Sketch, HyperLogLog, Trie, Geohash | Full Python implementations |
| 18 | **Scaling Zero to Billions** | [18_Scaling_From_Zero_to_Billions.md](./18_Scaling_From_Zero_to_Billions.md) | Architectural roadmap from 1 server to global multi-region fleet | Instagram & Stack Overflow scale |

---

## 🔗 Cross-Curriculum References
- **Low-Level Code Implementations**: See [LLD Module](../LLD/00-index.md) for object-oriented designs of Caches, Rate Limiters, and Booking Systems.
- **Production Backend Scenarios**: See [Backend Problems](../Backend%20Problems/00-index.md) for deep dives into flash sales, location tracking, and financial ledgers.
- **Enterprise Messaging**: See [Message Queues](../Message%20Queues/00-index.md) for Kafka, RabbitMQ, SQS, and Redis Streams internals.
- **Microservices Patterns**: See [Microservices Design](../Microservices%20Design/00-index.md) for the 12 core microservice architectural patterns.
- **Database Internals**: See [Databases](../Databases/00-index.md) for MySQL, PostgreSQL, Cassandra, DynamoDB, and Redis.
