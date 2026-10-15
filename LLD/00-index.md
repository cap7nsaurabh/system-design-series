# Low-Level Design (LLD) Master Index

Welcome to the **Low-Level Design (LLD) & Object-Oriented Software Architecture** knowledgebase. This series provides an end-to-end, industry-grade guide to designing maintainable, extensible, high-performance, and thread-safe software systems in **Java**.

---

## 📚 Curriculum Structure

```
LLD/
│
├── 00-index.md                                    # This master index
│
├── ── Principles & Architecture ──────────────────
│   ├── 01-lld-approaches-and-architectures.md     # Paradigms, OOAD, Clean Arch, DDD, UML & Metrics
│   ├── 02-solid-principles.md                     # Single Responsibility, Open/Closed, Liskov, ISP, DIP & more
│   └── 03-thread-safety-and-concurrency.md        # JMM, Locks, CAS, Concurrent Collections, Thread Pools
│
├── ── Creational Design Patterns ─────────────────
│   ├── 04-creational-design-patterns.md           # Category Index & Decision Flowchart
│   ├── 04-01-singleton-pattern.md                 # Singleton (DCL, Bill Pugh, Enum, Attack Defenses)
│   ├── 04-02-factory-method-pattern.md            # Factory Method (Payment Gateway)
│   ├── 04-03-abstract-factory-pattern.md          # Abstract Factory (Cross-Cloud Infrastructure)
│   ├── 04-04-builder-pattern.md                   # Builder (Fluent, Staged/Wizard, Invariant Validation)
│   ├── 04-05-prototype-pattern.md                 # Prototype (Deep vs Shallow Copy, Network Registry)
│   └── 04-06-object-pool-pattern.md               # Object Pool (Thread-Safe BlockingQueue Pool)
│
├── ── Structural Design Patterns ─────────────────
│   ├── 05-structural-design-patterns.md           # Category Index & Decision Tree
│   ├── 05-01-adapter-pattern.md                   # Adapter (XML to JSON Payment Adapter)
│   ├── 05-02-bridge-pattern.md                    # Bridge (Multi-Channel Notification Abstraction)
│   ├── 05-03-composite-pattern.md                 # Composite (Recursive Virtual File System)
│   ├── 05-04-decorator-pattern.md                 # Decorator (HTTP Processing Pipeline)
│   ├── 05-05-facade-pattern.md                    # Facade (E-Commerce Order Checkout Facade)
│   ├── 05-06-flyweight-pattern.md                 # Flyweight (Rich Text Glyph Formatting Engine)
│   └── 05-07-proxy-pattern.md                     # Proxy (Virtual, Security RBAC, Java Dynamic Proxy)
│
├── ── Behavioral Design Patterns ─────────────────
│   ├── 06-behavioral-design-patterns.md           # Category Index, Master Taxonomy & Decision Tree
│   ├── 06-01-chain-of-responsibility-pattern.md   # Chain of Responsibility (API Filter Pipeline)
│   ├── 06-02-command-pattern.md                   # Command (Transactional DB with Undo/Redo & Queue)
│   ├── 06-03-interpreter-pattern.md               # Interpreter (SQL WHERE Clause AST Parser)
│   ├── 06-04-iterator-pattern.md                  # Iterator (BST In-Order, Fail-Fast vs Fail-Safe)
│   ├── 06-05-mediator-pattern.md                  # Mediator (Air Traffic Control Tower Coordinator)
│   ├── 06-06-memento-pattern.md                   # Memento (Rich Text Editor State Snapshots)
│   ├── 06-07-observer-pattern.md                  # Observer (Stock Market Ticker with CopyOnWrite)
│   ├── 06-08-state-pattern.md                     # State (Order Lifecycle State Machine with CAS)
│   ├── 06-09-strategy-pattern.md                  # Strategy (Dynamic Pricing Engine with Lambdas)
│   ├── 06-10-template-method-pattern.md           # Template Method (Enterprise ETL Mining Pipeline)
│   └── 06-11-visitor-pattern.md                   # Visitor (Shopping Cart Double Dispatch Visitor)
│
└── ── Real-World LLD Problems & Case Studies ──────
    ├── 07-problem-parking-lot.md                  # Multi-floor parking, spot allocation strategies, concurrent gates
    ├── 08-problem-elevator-system.md              # Multi-car elevator dispatcher, LOOK algorithm, state machine
    ├── 09-problem-rate-limiter.md                 # Token Bucket, Leaky Bucket, Sliding Window Log/Counter, lock-free
    ├── 10-problem-splitwise-expense-sharing.md    # Expense splits, balances, Min Cash Flow graph simplification
    ├── 11-problem-in-memory-cache.md              # LRU/LFU cache with TTL eviction, ReadWriteLock, segmented striping
    ├── 12-problem-movie-ticket-booking.md         # Cinema seat locking with TTL, race prevention, transaction flow
    └── 13-problem-thread-safe-music-player.md     # Multi-threaded audio player, State machine, buffer backpressure, Observer
```

---

## 1. Architectural Foundations & LLD Approaches
* **File:** [`01-lld-approaches-and-architectures.md`](./01-lld-approaches-and-architectures.md)
* **Key Topics:**
  * **LLD vs. HLD:** Micro-architectural decisions, class hierarchies, boundary encapsulation, memory models.
  * **Design Paradigms:** Object-Oriented Analysis & Design (OOAD), Functional idioms in Java, Domain-Driven Design (DDD Aggregates, Entities, Value Objects, Domain Events, Repositories).
  * **Architectural Styles at Code Level:** Layered Architecture, Clean Architecture, Hexagonal (Ports & Adapters), Onion Architecture.
  * **Unified Modeling Language (UML):** Class diagrams (Association, Aggregation, Composition, Inheritance, Dependency) and Sequence diagrams with Mermaid.
  * **Quality Metrics:** High Cohesion, Loose Coupling, Cyclomatic Complexity, Connascence.
  * **Decomposition Strategies:** Top-Down vs. Bottom-Up design with e-commerce fulfillment walkthrough.

---

## 2. SOLID Principles & Software Quality Standards
* **File:** [`02-solid-principles.md`](./02-solid-principles.md)
* **Key Topics:**
  * **Single Responsibility Principle (SRP):** God objects anti-pattern, cohesion decomposition, thread-safety isolation.
  * **Open/Closed Principle (OCP):** Polymorphism, Strategy pattern, plugin architectures, eliminating switch/if-else ladders.
  * **Liskov Substitution Principle (LSP):** Formal behavioral subtyping, Rectangle/Square violation, precondition/postcondition/invariant contracts.
  * **Interface Segregation Principle (ISP):** Fat interfaces anti-pattern, role interfaces, client-tailored contracts.
  * **Dependency Inversion Principle (DIP):** Inversion of Control (IoC), Dependency Injection (Constructor, Setter, Interface) in vanilla Java and Spring.
  * **Supplementary Principles:** DRY vs. Rule of Three, KISS, YAGNI, Law of Demeter (Least Knowledge), Composition Over Inheritance, Favor Immutability, Fail-Fast assertions.
  * *All principles feature side-by-side BAD (violation) vs. GOOD (refactored) Java implementations.*

---

## 3. Concurrency, Multithreading & Thread Safety in Java
* **File:** [`03-thread-safety-and-concurrency.md`](./03-thread-safety-and-concurrency.md)
* **Key Topics:**
  * **Memory Models & Hardware Realities:** Processes, OS threads, Stack vs Heap, ThreadLocal lifecycle and leak vectors.
  * **Java Memory Model (JMM):** CPU cache lines, store buffers, instruction reordering, memory barriers, visibility vs atomicity, the `happens-before` order.
  * **Synchronization Primitives:**
    * `volatile`: semantics, memory barriers (LoadLoad, LoadStore, StoreStore, StoreLoad).
    * `synchronized`: Object header (Mark Word) layout, Biased -> Lightweight (Thin/Spin) -> Heavyweight (OS Mutex) inflation, lock coarsening & elision.
    * Explicit Locks: `ReentrantLock` (fairness, `tryLock()`, `Condition` queues), `ReentrantReadWriteLock` (read starvation, lock downgrade), `StampedLock` (optimistic reads).
    * Lock-Free & CAS: CMPXCHG assembly instruction, ABA problem, `AtomicInteger`, `AtomicReference`, `AtomicStampedReference`, `LongAdder` cell striping.
  * **Concurrent Collections:** `ConcurrentHashMap` (bucket locks, TreeBins, resize orchestration), `CopyOnWriteArrayList`, `BlockingQueue` family (`ArrayBlockingQueue`, `LinkedBlockingQueue`, `SynchronousQueue`).
  * **Concurrency Hazards:** Deadlock (Coffman conditions, lock ordering, detection), Livelock, Starvation.
  * **Executor Framework:** `ThreadPoolExecutor` internal architecture, thread sizing formulas for CPU-bound vs I/O-bound tasks, rejection policies.
  * **Thread-Safe Patterns:** DCL Singleton, Producer-Consumer with Conditions and BlockingQueues, Work-Stealing ForkJoinPool.

---

## 4. Creational Design Patterns
* **Category Index:** [`04-creational-design-patterns.md`](./04-creational-design-patterns.md)
* **Individual Pattern Guides:**
  * [04-01: Singleton Pattern](./04-01-singleton-pattern.md) — Eager, Lazy Synchronized, Double-Checked Locking (DCL) with `volatile`, Bill Pugh Static Inner Holder, Enum Singleton; reflection, serialization, and cloning attack defenses.
  * [04-02: Factory Method Pattern](./04-02-factory-method-pattern.md) — Decoupled object instantiation, subclass polymorphism, parameterized factory, Open/Closed compliance; multi-provider payment gateway.
  * [04-03: Abstract Factory Pattern](./04-03-abstract-factory-pattern.md) — Families of related products; cross-cloud infrastructure toolkit (AWS vs. Azure Compute, Storage, and Database clusters).
  * [04-04: Builder Pattern](./04-04-builder-pattern.md) — Telescoping constructor antipattern, step-by-step wizard builder, immutability, thread-safe object construction, build-time validation.
  * [04-05: Prototype Pattern](./04-05-prototype-pattern.md) — Deep vs. Shallow copying, `Cloneable` interface pitfalls, copy constructors and serialization-based cloning in a network configuration registry.
  * [04-06: Object Pool Pattern](./04-06-object-pool-pattern.md) — Generic thread-safe object pool with `BlockingQueue`, lease timeouts, health validation, resource leak prevention.

---

## 5. Structural Design Patterns
* **Category Index:** [`05-structural-design-patterns.md`](./05-structural-design-patterns.md)
* **Individual Pattern Guides:**
  * [05-01: Adapter Pattern](./05-01-adapter-pattern.md) — Object Adapter vs. Class Adapter; legacy integration (XML to modern JSON payment adapter).
  * [05-02: Bridge Pattern](./05-02-bridge-pattern.md) — Decoupling abstraction from implementation, mitigating $M \times N$ class explosion (Notification types across SMS/Email/Slack).
  * [05-03: Composite Pattern](./05-03-composite-pattern.md) — Tree hierarchies, uniform treatment of leaves and composites (File System with Files, Folders, and Symlinks).
  * [05-04: Decorator Pattern](./05-04-decorator-pattern.md) — Dynamic behavioral wrapping without subclass explosion; HTTP processing pipeline (Auth, Logging, Compression, Encryption).
  * [05-05: Facade Pattern](./05-05-facade-pattern.md) — Simplifying complex subsystems, high-level API orchestrator (Order placement checkout facade with compensation logic).
  * [05-06: Flyweight Pattern](./05-06-flyweight-pattern.md) — Memory optimization via Intrinsic (shared) vs. Extrinsic (context) state separation; rich text formatting glyph engine.
  * [05-07: Proxy Pattern](./05-07-proxy-pattern.md) — Virtual Proxy (lazy loading), Protection/Security Proxy (RBAC), Remote Proxy, Logging Proxy, Java Dynamic Proxy (`java.lang.reflect.Proxy`).

---

## 6. Behavioral Design Patterns
* **Category Index:** [`06-behavioral-design-patterns.md`](./06-behavioral-design-patterns.md)
* **Individual Pattern Guides:**
  * [06-01: Chain of Responsibility Pattern](./06-01-chain-of-responsibility-pattern.md) — Pipeline processing (Auth -> RateLimit -> Validation -> Execution), atomic sliding window rate limiter, short-circuiting.
  * [06-02: Command Pattern](./06-02-command-pattern.md) — Encapsulated actions, undo/redo history stacks, macro commands, asynchronous thread-safe command queues.
  * [06-03: Interpreter Pattern](./06-03-interpreter-pattern.md) — Domain-specific grammar representation, AST parsing, SQL WHERE clause evaluator.
  * [06-04: Iterator Pattern](./06-04-iterator-pattern.md) — Aggregate traversal, custom Binary Search Tree iterator, Fail-Fast (`modCount`) vs. Fail-Safe / Fail-Weak concurrent iterators.
  * [06-05: Mediator Pattern](./06-05-mediator-pattern.md) — Centralized communication hub, eliminating $N \times N$ colleague dependencies (Air Traffic Control tower).
  * [06-06: Memento Pattern](./06-06-memento-pattern.md) — Encapsulated object state snapshots, Originator-Caretaker-Memento roles, immutable state capture, memory optimization.
  * [06-07: Observer Pattern](./06-07-observer-pattern.md) — One-to-many publish-subscribe, `CopyOnWriteArrayList` thread safety, asynchronous notification dispatch, Java Flow API.
  * [06-08: State Pattern](./06-08-state-pattern.md) — State-dependent behavior changes, state machine transitions (Order lifecycle), thread-safe CAS transitions.
  * [06-09: Strategy Pattern](./06-09-strategy-pattern.md) — Family of interchangeable algorithms, runtime selection (Dynamic pricing/discount engine), modern Java lambda strategies.
  * [06-10: Template Method Pattern](./06-10-template-method-pattern.md) — Invariant algorithm skeleton with subclass hook points (ETL Data Mining Pipeline), Hollywood Principle.
  * [06-11: Visitor Pattern](./06-11-visitor-pattern.md) — Double dispatch mechanism, adding operations to heterogeneous object structures without class modification (Shopping cart tax/shipping calculation).

---

## 7. Real-World LLD Problems & System Case Studies

Every problem includes:
1. Requirements Analysis (Functional & Non-Functional)
2. Capacity & Scale Constraints
3. Domain Model & Mermaid Class Diagrams
4. Sequence Diagrams & Interaction Flows
5. Design Pattern Mapping
6. Concurrency & Thread-Safety Analysis
7. Complete, Production-Grade Java Implementation
8. Step-by-Step Code Walkthrough
9. Edge Cases & Extensibility Points

| Problem | File | Key Patterns Used | Concurrency & Thread Safety Focus |
| :--- | :--- | :--- | :--- |
| **Multi-Floor Parking Lot** | [`07-problem-parking-lot.md`](./07-problem-parking-lot.md) | Strategy (Allocation), Factory (Vehicle/Fee), Singleton | Concurrent gate arrivals, atomic spot reservation, `ReentrantLock` per floor |
| **Multi-Car Elevator System** | [`08-problem-elevator-system.md`](./08-problem-elevator-system.md) | State (Car states), Strategy (LOOK/SCAN), Observer | Independent car worker threads, concurrent floor calls, `Condition` signaling |
| **High-Performance Rate Limiter** | [`09-problem-rate-limiter.md`](./09-problem-rate-limiter.md) | Strategy (Algorithms), Factory, Singleton | Lock-free Token Bucket, Sliding Window Log/Counter, `AtomicLong`, CAS loops |
| **Splitwise Expense Sharing** | [`10-problem-splitwise-expense-sharing.md`](./10-problem-splitwise-expense-sharing.md) | Strategy (Splits: Equal, Exact, %), Observer | Graph Debt Simplification (Min Cash Flow), concurrent balance updates, `ConcurrentHashMap` |
| **In-Memory Cache (LRU/LFU + TTL)** | [`11-problem-in-memory-cache.md`](./11-problem-in-memory-cache.md) | Strategy (Eviction), Template Method | `ReentrantReadWriteLock`, Segmented lock striping, background TTL cleaner |
| **Movie Ticket Booking (BookMyShow)** | [`12-problem-movie-ticket-booking.md`](./12-problem-movie-ticket-booking.md) | State (Seat lifecycle), Strategy (Pricing) | Seat hold TTL lock, race condition prevention under flash crowd, atomic state flip |
| **Thread-Safe Music Player** | [`13-problem-thread-safe-music-player.md`](./13-problem-thread-safe-music-player.md) | State, Strategy, Observer, Command, Decorator, Facade | Producer-consumer audio buffer, `ReentrantReadWriteLock` playlist, `Condition` pause/resume |

---

## 🚀 How to Study This Series

1. **Foundations First:** Read [`01-lld-approaches-and-architectures.md`](./01-lld-approaches-and-architectures.md) and [`02-solid-principles.md`](./02-solid-principles.md) to build clean object modeling intuition.
2. **Master Concurrency:** Study [`03-thread-safety-and-concurrency.md`](./03-thread-safety-and-concurrency.md). High-concurrency low-level design is expected in senior and staff engineering interviews.
3. **Deep-Dive Patterns:** Study category indexes ([`04`](./04-creational-design-patterns.md), [`05`](./05-structural-design-patterns.md), [`06`](./06-behavioral-design-patterns.md)) and their respective pattern files ([`04-01`](./04-01-singleton-pattern.md) to [`06-11`](./06-11-visitor-pattern.md)).
4. **Code Case Studies:** Solve and review problems [`07`](./07-problem-parking-lot.md) through [`13`](./13-problem-thread-safe-music-player.md). Try implementing them yourself first before inspecting the reference Java code.
