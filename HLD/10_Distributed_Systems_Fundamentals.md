# 🔬 Distributed Systems Fundamentals & Consensus

> *"A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable."* — Leslie Lamport

---

## 📋 Table of Contents

1. [What Makes a System Distributed?](#what-is-distributed)
2. [The 8 Fallacies of Distributed Computing](#8-fallacies)
3. [CAP Theorem Deep Dive](#cap-theorem)
4. [PACELC Theorem](#pacelc)
5. [Consistency Models](#consistency-models)
6. [BASE vs ACID](#base-vs-acid)
7. [Distributed Clocks](#distributed-clocks)
8. [Consensus: Paxos & Raft](#consensus)
9. [Split-Brain & Quorums](#split-brain)
10. [Two-Phase Commit (2PC)](#2pc)
11. [Three-Phase Commit (3PC)](#3pc)
12. [Gossip Protocol](#gossip-protocol)
13. [Failure Detection](#failure-detection)
14. [Real-Life Scenario](#real-life-scenario)
15. [🔥 Big Tech Story: Google Chubby and ZooKeeper](#big-tech-story)
16. [Trade-offs & Best Practices](#trade-offs)
17. [Key Takeaways](#key-takeaways)

---

## 🌐 What Makes a System Distributed?

A **distributed system** is a collection of independent computers that appears to users as a single coherent system.

**Core challenges:**
1. **Partial failures:** Individual components can fail while others continue operating (unlike single-machine where all-or-nothing)
2. **No global clock:** Computers have independent clocks that drift; no single notion of "now"
3. **Unreliable communication:** Messages can be lost, delayed, reordered, or duplicated
4. **No shared memory:** Components communicate only via message passing

These challenges make distributed systems fundamentally harder than single-machine systems.

---

## 🚫 The 8 Fallacies of Distributed Computing

Articulated by L Peter Deutsch and James Gosling, these are false assumptions engineers make when building distributed systems.

| Fallacy | Reality |
|---|---|
| 1. The network is reliable | Networks drop packets, connections time out, switches fail |
| 2. Latency is zero | Network calls add 1-100ms; within a data center ~0.5ms; cross-region 100ms+ |
| 3. Bandwidth is infinite | Network links have limits; bandwidth bottlenecks are real |
| 4. The network is secure | Networks can be intercepted, packets spoofed, DNS poisoned |
| 5. Topology doesn't change | Servers are added/removed, IPs change, load balancers rotate |
| 6. There is one administrator | Multiple teams, clouds, vendors — coordination is messy |
| 7. Transport cost is zero | Serialization, compression, protocol overhead are non-trivial |
| 8. The network is homogeneous | Different MTUs, protocols, speeds across the path |

**The implications:** Your code MUST handle failures gracefully. Every network call must have a timeout. Every operation must be idempotent (safe to retry). You cannot assume the response to a request tells you what happened — the request may have been processed even if you never got the response.

---

## 🔺 CAP Theorem Deep Dive

Formally proved by Eric Brewer (2002) based on his conjecture from 2000: A distributed data store can guarantee at most **two** of three properties simultaneously.

### C — Consistency (Linearizability)

Every read receives the most recent write OR an error. If you write a value and another node immediately reads the same key, it sees your write.

```
Node A: write X = 5
Node B: read X → MUST return 5 (not old value, not an error about uncertainty)
```

This is **linearizability** — the strongest consistency model. Operations appear to execute instantaneously at a single point in time.

### A — Availability

Every request receives a **non-error response** (though it may not be the most recent data). The system keeps accepting reads and writes even during failures.

No node can reject a request saying "I can't serve you."

### P — Partition Tolerance

The system continues to operate even when the **network is partitioned** — some messages between nodes are lost or delayed indefinitely.

**Network partition:** A network failure that splits nodes into two groups that cannot communicate.

```
Before partition:         During partition:
  Node A ↔ Node B           Node A    |    Node B
  (can communicate)         (cannot reach each other)
```

### Why P is Non-Negotiable

In any real distributed system running on a network, **partitions WILL happen**. Network cables are cut, switches fail, routers have bugs. If you cannot tolerate partitions, your system cannot be distributed.

Therefore, the real choice is:
- **CP:** When partition occurs → system may become unavailable (refuses requests to maintain consistency)
- **AP:** When partition occurs → system remains available but returns potentially stale data

### Real Database Examples

**CP Systems (sacrifice Availability):**

| System | Behavior During Partition |
|---|---|
| HBase | Stops accepting writes; requires ZooKeeper quorum |
| ZooKeeper | Requires majority quorum; rejects writes if quorum lost |
| CockroachDB | Uses Raft; requires majority; rejects if can't achieve quorum |
| Etcd | Raft-based; requires quorum for writes |

**AP Systems (sacrifice Consistency):**

| System | Behavior During Partition |
|---|---|
| Cassandra (default) | Continues accepting writes; may return stale reads |
| DynamoDB (eventual) | Accepts writes; eventually converges |
| Riak | Always available; eventual consistency |
| CouchDB | Always available; sync on reconnect |

**CA Systems (sacrifice Partition Tolerance):**

Not truly distributed systems. Single-node PostgreSQL, MySQL are "CA" — they don't have to worry about partitions because there's only one node.

### Nuances of CAP

The CAP theorem is often misunderstood as a binary choice. In reality:

1. **Partitions are rare, not constant.** Most of the time, your system runs in normal operation without partitions. During normal operation, you can have both C and A. The CAP choice only matters DURING a partition.

2. **Consistency is a spectrum.** The "C" in CAP is specifically linearizability (the strongest form). Many systems offer weaker consistency guarantees that still provide useful properties.

3. **Available is binary.** A system is either available (returns non-error responses) or not.

---

## 📊 PACELC Theorem

Proposed by Daniel Abadi (2012) as an extension of CAP:

**If there is a Partition (P), choose between:**
- **Availability (A)**
- **Consistency (C)**

**Else (E) when the system is running normally, choose between:**
- **Latency (L)** — fast responses (acknowledge before full replication)
- **Consistency (C)** — correct responses (wait for full replication before ack)

PACELC captures the **performance vs correctness tradeoff** that exists even without partitions.

```
System replicated to 3 nodes:

For fast writes (EL choice):
  Write → acknowledge immediately → replicate async
  Latency: 5ms
  Risk: node crashes before replication → data loss

For correct writes (EC choice):
  Write → replicate to all 3 nodes → acknowledge
  Latency: 25ms (round trip to furthest replica)
  Guarantee: data never lost
```

| System | P: A or C | E: L or C |
|---|---|---|
| DynamoDB | A | L |
| Cassandra | A | L |
| MongoDB (default) | C | L |
| CRDB (CockroachDB) | C | C |
| PostgreSQL (sync) | C | C |
| MySQL (async replication) | A | L |

---

## 📏 Consistency Models

Consistency exists on a spectrum from strongest to weakest.

### 1. Linearizability (Strongest)

Every operation appears to execute instantaneously at some point between invocation and completion. All clients see operations in the same total order.

```
Timeline:
t=1: Client A writes X=5 (takes 10ms, completes at t=11)
t=5: Client B reads X → MUST return 5 (write was "linearized" before t=5 conceptually)

This requires strong synchronization across all nodes.
```

Used by: ZooKeeper, etcd, Google Spanner (with TrueTime)

**Cost:** Every operation must coordinate globally → high latency

### 2. Sequential Consistency

Operations appear to execute in the same sequential order for all clients, but not necessarily in real-time order.

```
Client A: write X=1, write X=2
Client B: read X, read X

Sequential consistency allows:
  B sees: X=1, X=2 (same order as A's writes, even if delayed)
  OR:     X=2, X=2 (sees only the latest writes)

But does NOT allow:
  B sees: X=2, X=1 (violates A's write order)
```

### 3. Causal Consistency

Operations that are causally related are seen in the correct order by all nodes. Concurrent operations (no causal relationship) can be seen in any order.

```
Client A: write X=5
Client B: reads X=5, then writes Y=10  (causally after seeing X=5)

All clients must see Y=10 AFTER X=5 (causal order preserved)

But if Client C independently writes Z=99 (no causal relation to X/Y):
  C's write and A's writes can be seen in any order → that's ok
```

Used by: Facebook's Cassandra tuning, some DynamoDB configurations

### 4. Eventual Consistency (Weakest Practical)

If no new updates are made, all replicas will eventually converge to the same value.

```
Write X=5 → reaches only Replica 1 initially
Read from Replica 2 → returns stale value (X=0 or previous value)
...some time later...
Replication catches up → Replica 2 also shows X=5
```

**Provides:** Availability, Partition tolerance  
**Lacks:** Immediate consistency guarantees

"Eventually" could be milliseconds or seconds depending on replication lag.

Subtypes:
- **Read-Your-Writes:** You always see your own writes (even if others don't yet)
- **Monotonic Reads:** Once you see a value, you never see an older value
- **Monotonic Writes:** Your writes are processed in order

### 5. Consistency Model Comparison

| Model | Guarantee | Performance | Common Use |
|---|---|---|---|
| Linearizability | Global total order, real-time | Slowest | ZooKeeper, Spanner |
| Sequential | Global total order, not real-time | Slow | Some RDBMS configs |
| Causal | Causally related ops ordered | Medium | Social feeds, comments |
| Read-Your-Writes | See your own writes | Medium | User profile updates |
| Eventual | Converge eventually | Fastest | Cassandra default, DNS |

---

## 🆚 BASE vs ACID

### ACID

- **Atomic:** All or nothing
- **Consistent:** Always valid state
- **Isolated:** Concurrent transactions don't interfere
- **Durable:** Committed writes survive crashes

ACID is what relational databases provide. It's the guarantee that allows you to say "if COMMIT returned successfully, the data is there and correct."

### BASE

- **Basically Available:** System appears available most of the time (with degradation)
- **Soft state:** State of the system may change over time (due to eventual consistency)
- **Eventually consistent:** System will become consistent over time

BASE is what most NoSQL and large-scale distributed systems provide. It trades strict correctness for availability and performance.

### Which to Choose?

| Requirement | Choose |
|---|---|
| Financial transactions, inventory | ACID (PostgreSQL, CockroachDB) |
| User preferences, settings | ACID fine; BASE acceptable |
| Social feed likes, view counts | BASE (slight inaccuracy ok) |
| DNS records | BASE (propagation takes time, that's ok) |
| IoT sensor data | BASE (some data loss tolerable) |
| Booking system (double-booking risk) | ACID critical |

---

## ⏰ Distributed Clocks

**Problem:** Each machine has its own clock (quartz oscillator). Even with NTP synchronization, clocks can drift apart by milliseconds to seconds. There is no "global now" in a distributed system.

### Why Wall Clocks Fail

```
Node A time: 10:00:00.100
Node B time: 10:00:00.050  (50ms behind)

Node A writes X=5 at "10:00:00.100"
Node B writes X=3 at "10:00:00.200"

Question: Which write happened "later"?
By wall clock: Node B's write (200 > 100)
But Node B's clock is behind — B's write may have happened BEFORE A's in reality!
```

If you use wall clock timestamps to determine order, you get incorrect results.

### Lamport Timestamps

Leslie Lamport's solution (1978): logical timestamps that capture causal ordering.

**Algorithm:**
1. Each node maintains a counter (Lamport clock) initialized to 0
2. On any local event: `counter++`
3. Before sending a message: `counter++`; include counter in message
4. On receiving a message: `counter = max(local_counter, message_counter) + 1`

```
Node A: counter=0
Node A sends message (counter→1, sends 1)
Node B receives message (counter=max(0,1)+1=2)
Node B sends reply (counter→3, sends 3)
Node A receives reply (counter=max(1,3)+1=4)
```

**Property:** If A happened-before B, then A.timestamp < B.timestamp
**Limitation:** A.timestamp < B.timestamp does NOT mean A happened-before B (events may be concurrent)

### Vector Clocks

Lamport timestamps can't detect concurrency. Vector clocks track causality per-node.

```
3 nodes: A, B, C
Each maintains a vector: [A_counter, B_counter, C_counter]

Node A sends message: A's vector = [1,0,0]
Node B receives (has [0,0,0]):
  B's vector = [max(0,1), max(0,0)+1, max(0,0)] = [1,1,0]
Node C sends message independently: C's vector = [0,0,1]
Node B receives C's message (has [1,1,0]):
  B's vector = [max(1,0), max(1,0), max(0,1)+1] = [1,1,2]

Comparing [1,0,0] vs [0,0,1]:
  Neither dominates the other (first has A>A in other; second has C>C in first)
  → Concurrent events! Conflict needs resolution.

Comparing [0,0,0] vs [1,1,0]:
  [0,0,0] ≤ [1,1,0] element-wise
  → [0,0,0] happened-before [1,1,0]
```

Used by: Amazon DynamoDB (internal vector clocks for conflict detection), Riak

**Limitation:** Vector size grows with number of nodes. In large systems, vector clocks are too expensive. Use version vectors instead (summarized per server).

### TrueTime (Google Spanner)

Google solved the distributed clock problem with **hardware**: GPS receivers + atomic clocks in every data center.

TrueTime API returns not a single timestamp but an interval: `[earliest, latest]` — the guaranteed bounds on actual current time.

```
TrueTime.now() → {earliest: 10:00:00.050, latest: 10:00:00.057}
(7ms uncertainty window due to clock synchronization)

Spanner's commit wait:
  When transaction commits:
    1. TrueTime.now() → {earliest: T, latest: T+ε}
    2. Wait until real time > T+ε (wait out the uncertainty)
    3. Then commit
  
  Result: Commit timestamps are guaranteed to be in strictly increasing order
  globally, across all data centers.
```

This enables **external consistency** — globally ordered transactions across geographically distributed data centers.

No other public cloud database offers this. It requires physical atomic clock infrastructure.

---

## 🤝 Consensus: Paxos & Raft

**Consensus:** Getting multiple nodes to agree on a single value, even in the presence of failures.

Required for: leader election, distributed locks, replicated state machines, database leader.

### Paxos (1989/1998)

Invented by Leslie Lamport. The foundational consensus algorithm. Notoriously difficult to understand and implement correctly.

**Basic Paxos (single-decree):**

Three roles: Proposers, Acceptors, Learners

**Phase 1: Prepare**
```
Proposer → all Acceptors: Prepare(n)  [n = proposal number]
Acceptor responds: Promise to not accept any proposal numbered < n
                   Returns highest-numbered proposal it has already accepted (if any)
```

**Phase 2: Accept**
```
If Proposer receives Promise from majority:
  Proposer → Acceptors: Accept(n, v)  [v = value (or highest already-accepted value)]
  Acceptor: Accepts if no higher-numbered Prepare has been received
```

**Phase 3: Committed**
```
If Proposer receives Accept from majority:
  Value v is chosen/committed
  Learners are notified
```

**Multi-Paxos** extends this for a sequence of values (replicated log). Used by: Google Chubby, Apache Zookeeper (ZAB protocol, a Paxos variant), Google Spanner.

**The problem with Paxos:** Lamport himself called it difficult. Implementing it correctly has caused numerous subtle bugs in production systems. Enter Raft.

### Raft (2014)

Designed specifically for **understandability**. Diego Ongaro and John Ousterhout's PhD work: "In Search of an Understandable Consensus Algorithm."

Raft decomposes consensus into three sub-problems:
1. **Leader election:** Select one leader at a time
2. **Log replication:** Leader accepts entries, replicates to followers
3. **Safety:** A committed entry is never overridden

**Raft Terms:**
- **Term:** An election period. Terms are numbered monotonically. Each term starts with an election.
- **Leader:** The single node that accepts client requests and replicates log entries.
- **Follower:** Passive; responds to leader and candidate requests.
- **Candidate:** A node trying to become leader.

**Leader Election:**
```
1. All nodes start as followers.
2. Follower doesn't hear from leader for election timeout (150-300ms) → becomes Candidate
3. Candidate increments term, votes for itself, sends RequestVote to all other nodes
4. Other nodes vote for the first candidate that requests in this term (if they haven't voted yet)
5. Candidate that receives votes from majority (N/2 + 1) → becomes Leader
6. Leader sends periodic heartbeat (AppendEntries with no entries) to prevent new elections

Split vote: No candidate wins majority → election timeout → new election (new term)
Randomized timeouts prevent continuous split votes.
```

**Log Replication:**
```
1. Client sends command to Leader
2. Leader appends command to its log (uncommitted)
3. Leader sends AppendEntries to all followers concurrently
4. Followers append to their log, respond OK
5. Once Leader receives OK from majority:
   a. Leader commits the entry (applies to state machine)
   b. Returns result to client
   c. Notifies followers of commit in next AppendEntries
6. Followers commit the entry (apply to their state machines)
```

**Log consistency:**
- Leader never overwrites existing committed entries — only appends
- Followers can't have a "gap" in their log (Raft ensures this via consistency check in AppendEntries)
- If a follower's log diverges → leader finds the divergence point and overwrites follower's conflicting entries

**Safety guarantee:** Only a node with the most up-to-date log can win an election (ensured via log comparison in RequestVote).

**Practical Raft implementations:** etcd (Kubernetes), CockroachDB, TiKV, InfluxDB, MongoDB Raft (since 4.0)

```
Raft in a 5-node cluster:
  Quorum: 3 nodes (majority)
  Can tolerate: 2 node failures
  
  Rule: N nodes, (N-1)/2 failures tolerated
  5 nodes → 2 failures tolerated
  3 nodes → 1 failure tolerated
```

---

## 🔀 Split-Brain & Quorums

### Split-Brain Problem

Network partition splits a cluster into two groups that cannot communicate. Both groups think the other is dead.

```
Cluster: Node A, Node B, Node C, Node D, Node E

Partition:
  Group 1: [A, B]
  Group 2: [C, D, E]
  
Without quorum: Both groups might elect their own leader and accept writes
→ Data diverges → inconsistency → very hard to reconcile
```

### Quorums — The Solution

A **quorum** is a majority of nodes. For N nodes, quorum is ⌊N/2⌋ + 1.

```
5 nodes → quorum = 3
3 nodes → quorum = 2

During partition:
  Group 1 [A, B]: size 2 < quorum (3) → CANNOT elect leader → becomes read-only or refuses writes
  Group 2 [C, D, E]: size 3 = quorum → CAN elect leader → accepts writes

Only one group can have quorum at any time!
```

**Quorum writes and reads:**

DynamoDB and Cassandra use configurable quorums:
```
N = 3 (replication factor)
W = 2 (write quorum: write to 2/3 replicas before ack)
R = 2 (read quorum: read from 2/3 replicas)

W + R > N → Strong consistency (W=2, R=2, N=3: 2+2=4>3 ✓)
```

If W + R > N, at least one replica is in the intersection of reads and writes → you always read the latest write.

For higher availability: `W=1, R=1` → writes and reads fast, but no consistency guarantee.

---

## 💳 Two-Phase Commit (2PC)

2PC is a protocol for achieving atomicity across multiple nodes in a distributed transaction.

```
Participants: Database on Node 1, Database on Node 2
Coordinator: Transaction Manager

Phase 1 — Prepare:
  Coordinator → Node 1: "Prepare to commit transaction X"
  Coordinator → Node 2: "Prepare to commit transaction X"
  Node 1: Writes to WAL, acquires locks, responds "PREPARED" (can commit, will not abort)
  Node 2: Writes to WAL, acquires locks, responds "PREPARED"

Phase 2 — Commit:
  Coordinator: Both PREPARED → "COMMIT"
  Coordinator → Node 1: "Commit"
  Coordinator → Node 2: "Commit"
  Nodes commit and release locks

If any PREPARE fails (node says "ABORT"):
  Coordinator → all nodes: "ABORT"
  Nodes discard changes
```

### The Blocking Problem

**Fatal flaw:** If the coordinator crashes AFTER phase 1 but BEFORE phase 2, participants are stuck in "prepared" state indefinitely:
- They've locked resources (can't accept other transactions)
- They can't commit (need coordinator's decision)
- They can't abort (might violate atomicity if coordinator intended to commit)

This is the **blocking** problem — 2PC is a blocking protocol.

**Mitigation:**
- Coordinator recovery: write coordinator decision to durable log; on restart, re-send decision
- Timeout: after long wait, participants can ask other participants what the decision was ("cooperative termination") — but this doesn't always work during a partition

### When to Use 2PC

Despite its flaws, 2PC is used in:
- XA transactions (Java EE, .NET distributed transactions)
- RDBMS with distributed transactions (PostgreSQL's `postgres_fdw`, MySQL XA)
- AWS RDS Proxy for Aurora global distributed transactions

Use 2PC when: you need atomic commit across multiple independent databases and can tolerate the blocking risk (usually acceptable in enterprise environments with reliable infrastructure).

---

## 🔄 Three-Phase Commit (3PC)

3PC adds a third phase to avoid the blocking problem of 2PC.

```
Phase 1 — canCommit?:
  Coordinator → Participants: "Can you commit?"
  Participants: respond Yes/No (no locks acquired yet)

Phase 2 — preCommit:
  If all Yes: Coordinator → "PreCommit" 
  Participants: acquire locks, write to WAL, respond "ACK"
  
Phase 3 — doCommit:
  Coordinator → "DoCommit"
  Participants: commit and release locks
```

**The key addition:** If coordinator crashes after PreCommit but before DoCommit, participants can recover:
- Timeout → participants can commit autonomously (safe because all agreed in phase 2)

**However:** 3PC still doesn't work perfectly under network partitions. If a partition occurs during phase 2, participants may disagree on whether to commit.

**In practice:** 3PC is rarely used in production. The Raft/Paxos family of algorithms is preferred for replication, and application-level sagas for distributed transactions.

---

## 🗣️ Gossip Protocol

How do nodes in a large cluster (thousands of nodes) share information efficiently?

**Gossip (epidemic protocol):** Each node periodically picks a random peer and exchanges information with it. Information spreads exponentially.

```
Round 1: Node A knows info I, tells Node B and Node D
Round 2: A+B+D each tell 2 random nodes
Round 3: Multiple nodes spread further
...
After log(N) rounds: ALL nodes have info I
```

For 1000 nodes: 10 rounds to reach all nodes.

**Used for:**
- **Membership:** Which nodes are alive/dead? Cassandra uses gossip for cluster membership
- **Node state:** Health information, load metrics
- **Failure detection:** Disseminate failure information through the cluster
- **Data anti-entropy:** DynamoDB/Cassandra gossip to ensure replicas are in sync

**Cassandra Gossip:**
```
Every second, each node:
  1. Picks 1-3 random nodes
  2. Exchanges HeartbeatState and ApplicationState
     (token ranges, schema version, load info)
  3. Updates local view of cluster

HeartbeatState: {generation, version}
  generation: when the node last restarted
  version: monotonically increasing heartbeat counter

If a node's heartbeat stops increasing → suspected dead
If confirmed dead → removed from cluster membership
```

### Properties of Gossip

- **Convergence:** Information reaches all nodes in O(log N) rounds
- **Resilience:** No single point of failure; works even if many nodes are down
- **Scalability:** Communication overhead is O(log N) per round, not O(N)
- **Eventual consistency:** Information is eventually consistent (not immediate)

---

## 💓 Failure Detection

How does a system determine that a node has failed (vs temporarily slow)?

### Heartbeats

The simplest approach. Nodes send periodic heartbeat messages. If no heartbeat for a threshold time → suspected failure.

```
Node A sends heartbeat every 1 second
Node B monitors: if no heartbeat for 5 seconds → Node A suspected dead
```

**Problem:** How do you choose the threshold?
- Too short → false positives (healthy but slow node declared dead)
- Too long → slow failure detection (real failures take too long to detect)

### Phi Accrual Failure Detector

Used by Cassandra. Instead of a binary (alive/dead) decision, maintains a **suspicion level** (phi, φ) that increases as heartbeats are missed.

```
φ(t) = -log₁₀(Probability of getting heartbeat at time t given history)

φ > threshold → suspect failure
High φ → very likely dead; Low φ → likely alive

Threshold tunable:
  φ = 1 → 10% chance of mistake (aggressive)
  φ = 5 → 0.001% chance of mistake (conservative)
  φ = 8 → 0.0000001% chance (very conservative, for Cassandra)
```

**Adapts to network conditions:** If network is normally slow, the expected heartbeat interval is adjusted accordingly → fewer false positives.

---

## 🔢 Real-Life Scenario: Distributed Counter

**The challenge:** Build a hit counter for a website that remains accurate across multiple server nodes, under network partitions.

**Approach 1: Single database (no distribution)**
```
Every request: UPDATE page_views SET count = count + 1 WHERE page='home'
```
Works perfectly. But: database bottleneck at high traffic, single point of failure.

**Approach 2: In-memory per-server counters (AP approach)**
```
Each server maintains local counter
Server 1: count=5, Server 2: count=3, Server 3: count=7

Total ≈ 15 (sum across servers)
Problem: sum requires querying all servers; exact count impossible during partition
```

**Approach 3: Redis distributed counter**
```
INCR page_views:home
# Redis is single-threaded → INCR is atomic
# With Redis Cluster: route page_views:home to specific shard
```
High throughput (Redis handles 100K+ ops/sec). But: Redis is a single SPOF unless clustered.

**Approach 4: CRDTs (Conflict-free Replicated Data Types)**

Grow-only counter (G-Counter): Each node maintains its own count. Total = sum across all nodes. No conflicts possible.

```
Node A: {A: 5, B: 0, C: 0}  (incremented 5 times on A)
Node B: {A: 0, B: 3, C: 0}  (incremented 3 times on B)
Node C: {A: 0, B: 0, C: 7}  (incremented 7 times on C)

After gossip/merge:
All nodes: {A: 5, B: 3, C: 7} → Total = 15

Merge function: take max of each element
{A: 5, B: 0, C: 0} merge {A: 0, B: 3, C: 0} = {A: 5, B: 3, C: 0}
```

No coordination needed. Always available. Eventually consistent. Used by Riak for counters.

**Trade-offs:**
| Approach | Consistency | Availability | Complexity |
|---|---|---|---|
| Single DB | Strong | Low (SPOF) | Low |
| Per-server | Eventual (approx) | High | Medium |
| Redis cluster | Strong | Medium-High | Medium |
| CRDT | Eventual (exact) | Very High | High |

The right choice depends on how exact the count needs to be and what downtime is acceptable.

---

## 🔥 Big Tech Story: Google Chubby and the Birth of ZooKeeper

### The Problem Google Faced (2003)

Google was running thousands of machines in their data centers for services like Google Search, MapReduce, and GFS (Google File System). These distributed systems needed:

1. **Leader election:** Which GFS master is authoritative?
2. **Distributed locking:** Only one MapReduce master should schedule work
3. **Configuration storage:** Small but critical configuration data (cluster membership, schema)

Without a reliable source of truth for these, distributed systems either needed to implement their own consensus or ran with subtle correctness bugs.

### Google Chubby (2004-2006)

Mike Burrows at Google designed **Chubby** — a distributed lock service based on Paxos consensus.

Core abstraction: A **file system interface** where files represent locks and small pieces of configuration.

```
/chubby/gfs/master    → contains IP of current GFS master
/chubby/bigtable/lock → GFS lock for BigTable master
```

Any service that needed to know the GFS master queried Chubby:
- Read `/chubby/gfs/master` → get master's IP
- Chubby guaranteed: everyone reading this file gets the same IP (linearizability)
- If the master fails → GFS master election protocol writes a new IP to Chubby

**Chubby's design:**
- 5 replicas using Paxos for consensus
- Can tolerate 2 failures
- Clients cache leases (reducing load on Chubby)
- Small data, high reliability

**Google's published Chubby paper** (OSDI 2006) by Mike Burrows became one of the most influential papers in distributed systems.

### The Birth of ZooKeeper

Yahoo! engineers (including Benjamin Reed and Flavio Junqueira) read the Chubby paper and realized they had the same problem. Rather than reimplementing Paxos from scratch (extremely error-prone), they built **ZooKeeper** — an open-source distributed coordination service.

ZooKeeper's differences from Chubby:
- Open source (Chubby was internal to Google)
- Used **ZAB (ZooKeeper Atomic Broadcast)** protocol — a Paxos variant focused on primary-backup replication
- Richer API: znodes (like files), watches (notifications on data changes), ephemeral nodes (deleted when client disconnects)

**ZooKeeper's key features:**
```
znodes: hierarchical namespace (like filesystem)
  /services/kafka/brokers/[1,2,3,4,5]  ← Kafka brokers register here

Ephemeral nodes: exist only while client is connected
  /services/kafka/brokers/1  (deleted when broker 1 disconnects)
  → Other services watching /services/kafka/brokers/* are notified immediately

Watches: Receive notification when a znode changes
  Service watches /leader → if leader changes, service is notified instantly
```

**Ecosystem built on ZooKeeper:**
- Kafka (original): Broker leader election, consumer group coordination, topic metadata
- HBase: Master election, region server assignment
- Hadoop YARN: ResourceManager HA and leader election
- Apache Storm: Nimbus leader election

### The Ironies and Learnings

1. **ZooKeeper itself became a SPOF.** Kafka required ZooKeeper so tightly that a ZooKeeper outage could destabilize a Kafka cluster. The Kafka team eventually built **KRaft** (Kafka Raft Metadata) to eliminate ZooKeeper dependency.

2. **Etcd replaced ZooKeeper for cloud-native.** Kubernetes chose etcd (a Raft-based store, not ZAB-based). Etcd is simpler to operate and has a better API for Kubernetes's needs.

3. **The abstraction level matters.** ZooKeeper's file-system-like API was too low-level for many use cases, leading developers to implement complex patterns on top. Consul (service mesh + coordination), etcd (simple key-value with watch), and Raft-native databases provide better abstractions for their specific use cases.

**The legacy:** Google Chubby → ZooKeeper → etcd/Consul is a direct lineage. The coordination service pattern (a small, highly-consistent cluster that stores critical distributed system metadata) is now ubiquitous in distributed systems. Every Kubernetes cluster, every Kafka cluster, every HDFS cluster depends on this concept.

---

## ⚖️ Trade-offs & Best Practices

### Trade-offs

| Concept | Pros | Cons |
|---|---|---|
| CP systems | Correct, consistent | May become unavailable during partitions |
| AP systems | Always available | May return stale data |
| Raft consensus | Understandable, correct | Leader bottleneck, requires majority |
| Gossip | Scalable, resilient | Eventual convergence, approximate knowledge |
| 2PC | Atomic across systems | Blocking on coordinator failure |
| Vector clocks | Precise causality | Storage grows with nodes |

### Best Practices

1. **Design for failure.** Assume any network call can fail. Every service call must have a timeout. Every operation must be idempotent.

2. **Choose your consistency model explicitly.** Don't let it be an accident. If your business requires no double-charging, you need strong consistency for that operation specifically.

3. **Prefer AP for user-facing reads, CP for writes.** Serve stale data (AP) for read-heavy features. Enforce consistency (CP) only where business logic requires it (inventory, payments).

4. **Understand your system's failure modes.** Test network partitions explicitly (chaos engineering). Know what your system does when the coordination service goes down.

5. **Monitor replication lag.** In AP systems, replication lag is your staleness window. If it exceeds acceptable thresholds, alert.

6. **Use quorums thoughtfully.** Cassandra's `QUORUM` consistency level is a strong guarantee. Use it for critical reads/writes. Use `ONE` only where eventual consistency is explicitly acceptable.

7. **Don't implement your own consensus.** Use Raft via etcd, ZooKeeper, or CockroachDB. Consensus algorithms are notoriously difficult to implement correctly.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **8 Fallacies** | The network is NOT reliable, zero-latency, or secure. Design for failure. |
| **CAP Theorem** | P is mandatory; choose C (CP) or A (AP) during partitions |
| **PACELC** | Even without partitions: higher consistency = higher latency |
| **Consistency models** | Spectrum from linearizability (strongest) to eventual (weakest) |
| **Lamport timestamps** | Logical clocks capture causal ordering without physical clocks |
| **Raft** | Understandable consensus: leader election + log replication + safety |
| **Quorums** | W + R > N ensures reading latest write; prevents split-brain |
| **2PC** | Atomic across systems but blocking on coordinator failure |
| **Gossip** | Scalable O(log N) information dissemination; used in Cassandra |
| **Google's lesson** | Coordination services (Chubby→ZooKeeper→etcd) are the nervous system of distributed systems |

---

*Next: [11_Storage_Systems.md](11_Storage_Systems.md) — Object storage, block storage, file systems, and the internals of how databases store data.*
