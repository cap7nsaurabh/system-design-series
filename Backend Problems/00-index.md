# System Design Series: Real-World Backend Challenges

Welcome to the **Backend Problems** series. This section contains deep-dive architectural analyses of real-world backend engineering challenges, production failure post-mortems from top-tier tech companies (YouTube, Instagram, Netflix, Uber, Stripe, Discord, Pinterest), and production-grade code implementations.

---

## 📚 Complete Table of Contents

### 1. Concurrency, Consistency & Contention
* [01 Concurrency Control & Resource Booking](01-concurrency-resource-booking.md)
  * *Domain:* E-Commerce Flash Sales & Ticket Booking (Avengers Endgame, iPhone sales).
  * *Key Concepts:* Race conditions, Lost Updates, Pessimistic Locking (`FOR UPDATE`), Optimistic Locking with versions, Distributed Mutexes via Redis Redlock.
* [02 High-Throughput Counters & Volatile Entities](02-high-throughput-counters.md)
  * *Domain:* Social Media Metrics (YouTube Views, Instagram Accounts Reached, Twitch Live Viewers).
  * *Key Concepts:* YouTube's "301+ Freeze" pipeline, Google Spanner & Vitess, Instagram Reach vs Impressions (Kafka & Flink stream processing), Cassandra per-user events, HyperLogLog (`PFADD`/`PFCOUNT`), Sharded Counters, Count-Min Sketch.
* [05 Distributed Transactions & Consistencies](05-distributed-transactions-sagas.md)
  * *Domain:* Microservice Workflows & Distributed State.
  * *Key Concepts:* Two-Phase Commit (2PC) & 3PC, In-Doubt state anomalies, SAGA Patterns (Choreography vs Orchestration), Transactional Outbox & CDC, Try-Confirm-Cancel (TCC), Distributed SQL (Spanner/CockroachDB Raft + 2PC), Saga Isolation Anomalies (ACD without I) & Countermeasures.

### 2. High-Scale Traffic & Resiliency
* [03 Handling Burst Traffic & Resiliency](03-handling-burst-traffic-and-resiliency.md)
  * *Domain:* High-Volume E-Commerce & Flash Traffic Spikes.
  * *Key Concepts:* Circuit Breakers, Bulkhead isolation, Load shedding, Exponential backoff with jitter, Rate limiting at ingress.
* [13 Rate Limiting & Throttling at Scale](13-rate-limiting-and-throttling.md)
  * *Domain:* API Protection & DDoS Defense (Stripe, GitHub, Cloudflare).
  * *Key Concepts:* Token Bucket, Leaky Bucket (GCRA), Fixed Window 2x spike vulnerability, Sliding Window Log, Sliding Window Counter (Cloudflare hybrid), Distributed Redis atomic Lua scripts (`EVALSHA`), Hybrid L1/L2 token batching, Hot-Key sub-slotting, Distributed Concurrency Limiter semaphores, Circuit Breaker fail-open/fail-closed policies, IETF standard headers (`RFC 6585`), AWS Full Jitter client backoff.

### 3. Real-Time Data & Geolocation
* [04 Real-Time Location Tracking & Logistics](04-real-time-location-tracking.md)
  * *Domain:* Ride-Hailing & On-Demand Delivery (Uber, Lyft, DoorDash).
  * *Key Concepts:* Geospatial indexing (Uber H3 hexagonal hierarchical spatial index, Google S2 geometry, Geohash), QuadTrees, Redis Geospatial (`GEOSEARCH`), High-velocity GPS ingestion via WebSockets/Kafka.
* [17 Real-Time Notifications & Push Systems](17-real-time-notifications-and-push.md)
  * *Domain:* Instant Messaging & Push Notification Gateways (WhatsApp, Slack, Instagram).
  * *Key Concepts:* WebSockets vs SSE vs Long Polling, 10-Million connection WebSocket Gateway architectures with Redis Session Registry, Mobile Push mechanics (Apple APNs & Google FCM), The Celebrity Fanout problem, WhatsApp Erlang architecture.

### 4. Data Architecture, Caching & Scalability
* [06 Heavy Read Workloads & Social Feeds](06-heavy-read-workloads-social-feeds.md)
  * *Domain:* Social Media Timelines & High-Read Activity Streams (Twitter/X, Instagram, Facebook, TikTok, LinkedIn).
  * *Key Concepts:* Fan-Out on Write (Push / Eager) vs Fan-Out on Read (Pull / Scatter-Gather), Hybrid Dynamic Tiering, Two-Stage ML Candidate Sourcing & Ranking, Inverted Index Feed Architecture (Twitter Earlybird), 64-bit Snowflake ID cursor pagination, Roaring Bitsets, Redis Skiplist memory layout, Split-Brain fault tolerance, and Capacity Planning.
* [12 Search & Indexing Systems](12-search-and-indexing-systems.md)
  * *Domain:* E-Commerce & Job Search Engines (Airbnb, LinkedIn, Google Autocomplete).
  * *Key Concepts:* Inverted indexes, BM25 relevance scoring mechanics, Change Data Capture (CDC) via Debezium and Kafka, In-memory Trie autocomplete data structures, Semantic Vector Search with HNSW and embeddings.
* [15 Caching Strategies Deep Dive](15-caching-strategies-deep-dive.md)
  * *Domain:* Ultra-Low Latency Read Stores & Massive Caching Fleets (Facebook Memcached & Tao, Netflix EVCache).
  * *Key Concepts:* Amdahl's Law for Caching, Eviction Mechanics (LRU, LFU, W-TinyLFU, Redis Approximated LRU), High-Level & Low-Level Designs for 4 Core Patterns (Cache-Aside with circuit breakers, Read-Through/Write-Through with atomicity analysis, Write-Behind with write coalescing and ring buffer flusher, Refresh-Ahead with proactive worker pool), Cache Catastrophes (Stampedes mitigated via SingleFlight, Distributed Mutex, and XFetch probabilistic early expiration; Penetration mitigated via Bloom/Cuckoo Filters and Negative Caching; Avalanche mitigated via TTL Jitter), Invalidation & Consistency (Dual-Write race analysis, CDC via Debezium + Kafka WAL streaming, Facebook Leases Protocol), Multi-Tier L1/L2/L3 topologies and L1 cache coherence via Redis 6+ RESP3 Client-Side Tracking.
* [16 Database Sharding Strategies](16-database-sharding-strategies.md)
  * *Domain:* Horizontal Scale Beyond Single-Node Hardware (Pinterest, Discord, Slack).
  * *Key Concepts:* Range-based, Hash-based, and Directory-based sharding, Consistent Hashing with Virtual Nodes, Twitter Snowflake 64-bit time-sortable distributed ID generation, Cross-shard join mitigation, Zero-downtime resharding.

### 5. Domain-Specific Industry Architectures
* [07 FinTech & Banking Systems](07-fintech-and-banking-systems.md)
  * *Domain:* Financial Core Banking & Payment Ledgers.
  * *Key Concepts:* Double-entry bookkeeping, Absolute idempotency keys, Reconciliation engines, Audit logs, Zero-loss transactional guarantees.
* [08 High-Frequency Trading & Capital Markets](08-high-frequency-trading.md)
  * *Domain:* Low-Latency Exchanges, Prop Trading & Quantitative Finance.
  * *Key Concepts:* Nanosecond Latency Hierarchy, Sub-microsecond Tick-to-Trade lifecycle, Real-world market disasters (Knight Capital, 2010 Flash Crash, BATS IPO), Binary ITCH 5.0 / SBE / OUCH protocols vs FIX, Flat array + intrusive doubly-linked list Order Books, Kernel bypass (Solarflare EF_VI / DPDK), Lock-free LMAX Disruptor with cache line padding (MESI false sharing prevention), Branchless pre-trade risk engines, PTP (IEEE 1588v2) PHY timestamping, Colocation & Layer 1 cut-through switches (Arista 7130), Microwave networks, and Bare-metal Linux kernel tuning (`isolcpus`, `nohz_full`, hugepages).
* [09 Gaming & Real-Time Collaboration](09-gaming-and-collaboration.md)
  * *Domain:* Multiplayer Games & Collaborative Docs (Google Docs, Figma).
  * *Key Concepts:* Authoritative game servers, Client-side prediction & server reconciliation, Operational Transformation (OT) vs Conflict-Free Replicated Data Types (CRDTs).
* [10 IoT & Telemetry Streaming](10-iot-and-telemetry.md)
  * *Domain:* Connected Hardware & Industrial Sensor Monitoring (Tesla Fleet, Datadog Agents, Smart Grids).
  * *Key Concepts:* MQTT 5.0 vs CoAP edge gateways, Decorrelated Jitter backoff, Kafka partitioned ingestion, Event Time vs Ingestion Time vs Processing Time, Watermarking & Side Outputs in Apache Flink, Tumbling/Sliding/Session windows, Gorilla XOR compression & LTTB downsampling, Multi-tier TSDB storage (ClickHouse/TimescaleDB), Welford streaming anomaly detection, Digital Twins (Device Shadows) with optimistic concurrency.
* [11 Healthcare & Telemedicine](11-healthcare-and-telemedicine.md)
  * *Domain:* HIPAA-Compliant Medical Records, Telehealth & IoMT (Epic Systems, Teladoc, Cerner/Oracle Health).
  * *Key Concepts:* Healthcare Engineering Trilemma, 18 HIPAA Safe Harbor identifiers, Envelope Encryption with KMS/DEK/KEK, Dynamic Attribute-Based Access Control (ABAC & OPA Rego), "Break-Glass" emergency override protocol, Tamper-evident Merkle Tree audit ledgers with WORM storage, MLLP TCP socket streaming ingestion, HL7 v2 to FHIR R4 ETL & Terminology Normalization (SNOMED-CT, LOINC, ICD-10), WebRTC SFU with client-side Insertable Streams E2EE, IoMT high-frequency vitals streaming with Flink NEWS2 sepsis alerting & alarm fatigue deadbanding, Cloud DICOMweb (WADO-RS, QIDO-RS, STOW-RS) medical imaging with progressive tile streaming and tag de-identification.
* [14 Event-Driven Architecture Patterns](14-event-driven-architecture-patterns.md)
  * *Domain:* Microservice Coordination & Distributed Systems.
  * *Key Concepts:* Transactional Outbox Pattern with Debezium CDC, Saga Orchestration (Temporal / Cadence) vs Choreography, Command Query Responsibility Segregation (CQRS), Idempotent consumers with unique constraints, Dead Letter Queues (DLQ).

---
*Created as a comprehensive learning resource for advanced backend systems and architecture.*
