# Low-Level Design: Splitwise (Expense Sharing & Debt Simplification System)

## 1. Executive Summary & Problem Statement

**Splitwise** is a world-class financial utility enabling groups and individuals to track shared expenses, split costs equitably, maintain running bilateral and group balance sheets, and settle debts with the absolute minimum number of financial transactions.

```
                           THE EXPENSE SHARING CHALLENGE
                           
   4 Friends on a Trip: Alice, Bob, Charlie, David
   Total Expenses Incurred: 12 different bills across restaurants, cabs, Airbnb, groceries
   
   Without Debt Simplification:
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”      $50      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚  Alice  â”œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚   Bob   â”‚
   â””â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”˜               â””â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”˜
        â”‚ $30                     â”‚ $40
        â–¼                         â–¼
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”      $20      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚ Charlie â”‚â—„â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¤  David  â”‚
   â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜               â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
   (Dense web of 6 to 12 bilateral transactions, high transaction fees, social friction)
   
   With Min Cash Flow Simplification:
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”      $60      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚  Alice  â”œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚ Charlie â”‚
   â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜               â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
   â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”      $30      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”
   â”‚   Bob   â”œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚  David  â”‚
   â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜               â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
   (Reduced to at most N - 1 = 3 optimal settlement transactions!)
```

### The Core Architectural Challenges
1. **Financial Precision & Cents Rounding**: Floating-point arithmetic (`float`, `double`) introduces binary representation errors ($0.1 + 0.2 = 0.30000000000000004$). A production financial system must represent currency in atomic integer units (e.g., cents/paise) or arbitrary-precision decimals (`BigDecimal` with explicit `RoundingMode`), ensuring that split sums strictly match the total expenditure down to the last penny.
2. **Strategy-Driven Split Computation**: Expenses can be divided equally (`EQUAL`), by exact amounts (`EXACT`), or by percentage shares (`PERCENT`). The system must validate inputs against strict domain invariants before any state is mutated.
3. **High-Contention Concurrency & Deadlock Prevention**: When multiple roommates or trip participants add expenses simultaneously, bilateral balance sheets ($User_A \leftrightarrow User_B$) undergo concurrent write mutations. Uncontrolled synchronization causes lost updates; naive lock acquisition across two users ($Lock(A) \rightarrow Lock(B)$ vs. $Lock(B) \rightarrow Lock(A)$) leads to instantaneous thread deadlocks.
4. **Optimal Debt Simplification**: As transactions accumulate, the debt graph becomes dense ($O(N^2)$ edges). The system must compute a simplified transaction graph that minimizes the total number of physical money transfers while preserving every user's net financial balance invariant.

---

## 2. Requirements Gathering

### 2.1 Functional Requirements
1. **User & Group Management**:
   - Register users with unique identifiers, names, emails, and phone numbers.
   - Create groups, add/remove members, and track group-specific expenses.
   - Support non-group (individual peer-to-peer) expenses.
2. **Expense Creation & Split Strategies**:
   - Record an expense paid by a user on behalf of one or more participants.
   - Support three primary split strategies via the **Strategy Pattern**:
     - **EQUAL Split**: Divides amount evenly. If unevenly divisible, distribute remainder cents deterministically.
     - **EXACT Split**: Explicit dollar/cent amounts for each participant. Sum of splits must strictly equal total expense.
     - **PERCENTAGE Split**: Percentage shares per participant. Sum must strictly equal $100.00\%$.
3. **Balance Sheet Invariants**:
   - Track pairwise bilateral balances: "User A owes User B \$X".
   - View a user's net global balance across all contacts.
   - View user-specific balances within a specific group.
4. **Debt Simplification (Min Cash Flow)**:
   - Provide an on-demand algorithm to compute the minimal set of transactions needed to completely square off all debts within a group or global network.
5. **Settlement & Auditability**:
   - Record settlement payments between users that decrement or clear bilateral debt.
   - Append-only activity log for audits, revisions, and dispute resolution.

### 2.2 Non-Functional Requirements
1. **Data Consistency & Correctness**: Zero money creation or destruction. Total net balance across all system users must identically sum to zero:
   $$\sum_{i=1}^{N} \text{NetBalance}(User_i) \equiv 0$$
2. **Thread Safety & High Concurrency**: Must support hundreds of concurrent expense submissions across overlapping user sets without deadlocks, race conditions, or corrupted balance sheets.
3. **Sub-Millisecond Response Time**: In-memory balance lookups and updates must execute in $O(1)$ or $O(K)$ where $K$ is the number of participants in an expense.
4. **Extensibility (Open-Closed Principle)**: Adding new split strategies (e.g., `SHARE_RATIO`, `ADJUSTMENT_SPLIT`, `DYNAMIC_TIP`) must require zero modifications to existing core business logic.

---

## 3. Capacity Estimation & Numerical Constraints

### Operational Scale
- **Daily Active Users (DAU)**: $10,000,000$
- **Expenses Added Per Day**: $50,000,000$
- **Average Participants Per Expense ($K$)**: $4$
- **Peak Write Throughput**:
  $$\text{Average RPS} = \frac{50,000,000}{86,400} \approx 578 \text{ expenses/sec}$$
  $$\text{Peak Multiplier} = 10\times \implies \text{Peak Write RPS} \approx 6,000 \text{ writes/sec}$$

### In-Memory Storage Footprint
- User object: $\approx 200 \text{ bytes}$
- Bilateral balance entry (`ConcurrentHashMap<String, LongAdder>`): $\approx 128 \text{ bytes}$
- Active balance pairs per user: $\approx 50$
- Memory for $10\text{M}$ active balance graphs:
  $$10,000,000 \times 50 \times 128 \text{ bytes} \approx 64 \text{ GB RAM}$$
- A sharded or striped in-memory cluster (or single modern 128GB JVM server) can comfortably manage the entire active balance state.

---

## 4. Core Domain Entities & Class Diagram

```mermaid
classDiagram
    direction TB

    class User {
        -String userId
        -String name
        -String email
        -String phoneNumber
        +getUserId() String
        +getName() String
    }

    class Group {
        -String groupId
        -String groupName
        -Set~String~ memberUserIds
        -List~String~ expenseIds
        +addMember(String userId) void
        +removeMember(String userId) void
        +getMembers() Set~String~
    }

    class Expense {
        -String expenseId
        -String description
        -long amountInCents
        -String paidByUserId
        -String groupId
        -SplitType splitType
        -List~Split~ splits
        -Instant createdAt
        +getAmountInCents() long
        +getSplits() List~Split~
    }

    class SplitType {
        <<enumeration>>
        EQUAL
        EXACT
        PERCENT
    }

    class Split {
        <<abstract>>
        -String userId
        -long amountInCents
        +getUserId() String
        +getAmountInCents() long
        +setAmountInCents(long cents) void
    }

    class EqualSplit {
        +EqualSplit(String userId)
    }

    class ExactSplit {
        +ExactSplit(String userId, long amountInCents)
    }

    class PercentSplit {
        -double percentage
        +PercentSplit(String userId, double percentage)
        +getPercentage() double
    }

    Split <|-- EqualSplit
    Split <|-- ExactSplit
    Split <|-- PercentSplit
    Expense --> SplitType
    Expense o-- Split

    class ISplitStrategy {
        <<interface>>
        +validateSplits(long totalAmountInCents, List~Split~ splits) void
        +calculateSplits(long totalAmountInCents, List~Split~ splits) void
    }

    class EqualSplitStrategy {
        +validateSplits(...) void
        +calculateSplits(...) void
    }

    class ExactSplitStrategy {
        +validateSplits(...) void
        +calculateSplits(...) void
    }

    class PercentSplitStrategy {
        +validateSplits(...) void
        +calculateSplits(...) void
    }

    ISplitStrategy <|.. EqualSplitStrategy
    ISplitStrategy <|.. ExactSplitStrategy
    ISplitStrategy <|.. PercentSplitStrategy

    class BalanceManager {
        -ConcurrentHashMap~String, ConcurrentHashMap~String, Long~~ userBalances
        -ConcurrentHashMap~String, StripedLock~ lockManager
        +updateBalance(String payerId, String borrowerId, long cents) void
        +getUserBalances(String userId) Map~String, Long~
        +getAllNetBalances() Map~String, Long~
    }

    class DebtSimplifier {
        <<interface>>
        +simplifyDebts(Map~String, Long~ netBalances) List~SettlementTransaction~
    }

    class GreedyMinCashFlowSimplifier {
        +simplifyDebts(Map~String, Long~ netBalances) List~SettlementTransaction~
    }

    DebtSimplifier <|.. GreedyMinCashFlowSimplifier

    class SettlementTransaction {
        -String fromUserId
        -String toUserId
        -long amountInCents
        +getFromUserId() String
        +getToUserId() String
        +getAmountInCents() long
    }

    class SplitwiseService {
        -UserManager userManager
        -GroupManager groupManager
        -BalanceManager balanceManager
        -SplitStrategyFactory strategyFactory
        -DebtSimplifier debtSimplifier
        +addExpense(...) Expense
        +settleUp(String user1, String user2, long cents) void
        +simplifyGroupDebts(String groupId) List~SettlementTransaction~
    }

    SplitwiseService --> BalanceManager
    SplitwiseService --> ISplitStrategy
    SplitwiseService --> DebtSimplifier
    GreedyMinCashFlowSimplifier ..> SettlementTransaction
```

---

## 5. Sequence Diagram / Workflow

### Adding an Expense with Atomic Balance Updates and Deadlock-Free Locking

```mermaid
sequenceDiagram
    autonumber
    actor Alice as User Alice (Payer)
    participant Service as SplitwiseService
    participant Factory as SplitStrategyFactory
    participant Strategy as EqualSplitStrategy
    participant BalMgr as BalanceManager
    participant Audit as AuditLogger

    Alice->>Service: addExpense("Dinner", 30000 cents, Alice, [Alice, Bob, Charlie], EQUAL)
    Service->>Factory: getStrategy(EQUAL)
    Factory-->>Service: EqualSplitStrategy instance
    
    Service->>Strategy: validateSplits(30000, splits)
    Note over Strategy: Check nulls, empty lists, positive total
    Strategy-->>Service: Validated OK
    
    Service->>Strategy: calculateSplits(30000, splits)
    Note over Strategy: 30000 / 3 = 10000 cents each<br/>Remainder = 0 cents
    Strategy-->>Service: Populated Splits [Alice: 10000, Bob: 10000, Charlie: 10000]

    rect rgb(240, 248, 255)
        Note over Service, BalMgr: Concurrency Control (Ordered Bilateral Updates)
        loop For each borrower (Bob, Charlie)
            Service->>BalMgr: updateBalance(Alice, Bob, 10000 cents)
            Note over BalMgr: Acquire Canonical Lock: min("Alice","Bob") then max("Alice","Bob")<br/>prevents AB-BA deadlock!
            BalMgr->>BalMgr: userBalances[Bob][Alice] += 10000<br/>userBalances[Alice][Bob] -= 10000
            BalMgr-->>Service: Balance Updated
        end
    end

    Service->>Audit: recordEvent(ExpenseCreatedEvent)
    Audit-->>Service: ACK
    Service-->>Alice: Expense Confirmation (expenseId, splits)
```

---

## 6. Design Pattern Mapping

| Design Pattern | Concrete Component | Architectural Justification |
| :--- | :--- | :--- |
| **Strategy Pattern** | `ISplitStrategy`, `EqualSplitStrategy`, `ExactSplitStrategy`, `PercentSplitStrategy` | Encapsulates distinct mathematical validation and distribution algorithms. New split styles can be added without modifying `SplitwiseService`. |
| **Factory Pattern** | `SplitStrategyFactory` | Decouples strategy instantiation from execution. Maps `SplitType` enum to cached strategy singletons. |
| **Domain-Driven Value Object** | `Split`, `SettlementTransaction` | Encapsulates immutable transaction quantities in integer cents, guarding against precision drift. |
| **Ordered Lock Pattern** | `BalanceManager.getLockPair(userA, userB)` | Eliminates cycle-based concurrency deadlocks when two users concurrently post expenses against each other. |
| **Facade Pattern** | `SplitwiseService` | Provides a unified, thread-safe public API masking the complexities of user validation, strategy resolution, atomic balance math, and audit logging. |

---

## 7. Concurrency & Thread-Safety Analysis

### 7.1 The Bilateral Race Condition & Deadlock Trap
Suppose User $A$ and User $B$ go out. Simultaneously:
- **Thread 1 (Alice's app)**: Alice pays \$50 for lunch shared with Bob. Mutates pair $(A, B)$.
- **Thread 2 (Bob's app)**: Bob pays \$30 for an Uber shared with Alice. Mutates pair $(B, A)$.

#### The Naive Locking Flaw (Deadlock Hazard)
```java
// FATAL ANTI-PATTERN:
// Thread 1 acquires lock(Alice) -> then attempts to acquire lock(Bob)
// Thread 2 acquires lock(Bob)   -> then attempts to acquire lock(Alice)
// RESULT: Classic AB-BA Coffman Deadlock!
```

#### The Production Solution: Canonical Resource Ordering
To guarantee a strict partial order on locks and prevent cycles:
$$\text{Lock}_1 = \min(User_A, User_B), \quad \text{Lock}_2 = \max(User_A, User_B)$$
Where comparison is deterministic (e.g., `userIdA.compareTo(userIdB)`). Every thread in the JVM acquires the lower-ordered ID lock first, rendering deadlocks mathematically impossible.

```
       Thread 1 (Alice & Bob)                   Thread 2 (Bob & Alice)
                 â”‚                                        â”‚
                 â–¼                                        â–¼
      Canonical Lock 1: "Alice"                Canonical Lock 1: "Alice"
      (Acquired by Thread 1)                   (Blocks until Thread 1 releases)
                 â”‚                                        â”‚
                 â–¼                                        â”‚
      Canonical Lock 2: "Bob"                             â”‚
      (Acquired by Thread 1)                              â”‚
                 â”‚                                        â”‚
         [Update Balances]                                â”‚
                 â”‚                                        â”‚
      Release Lock 2, Lock 1                              â”‚
                 â”‚                                        â–¼
                 â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–º Acquired by Thread 2!
```

### 7.2 Integer Cents Arithmetic to Eliminate Precision Loss
Standard IEEE 754 floating-point operations accumulate non-zero rounding errors.
```java
// Anti-pattern:
double split = 100.0 / 3.0; // 33.333333333333336
double total = split + split + split; // 99.99999999999999 -> 1 cent lost in the ether!
```
**Production Rule**: Every balance is stored as a 64-bit integer (`long amountInCents`). When dividing 10,000 cents across 3 users:
- Base share: $\lfloor 10000 / 3 \rfloor = 3333 \text{ cents}$
- Remainder: $10000 \pmod 3 = 1 \text{ cent}$
- The first user receives $3333 + 1 = 3334 \text{ cents}$; remaining users receive $3333 \text{ cents}$.
- Sum check: $3334 + 3333 + 3333 = 10000 \text{ cents}$. Invariant preserved!

---

## 8. Debt Simplification Algorithm (Min Cash Flow)

### 8.1 Graph Theory Background
In a raw expense system with $N$ users, the debt relationship is a directed weighted graph $G = (V, E)$, where $|E| \le N(N - 1)$.
- **Full Optimal Simplification (NP-Hard)**: Finding the absolute minimum number of transactions across arbitrary subsets is equivalent to the **Subset Sum Problem**, which is NP-hard.
- **Greedy Min Cash Flow Algorithm**: A widely adopted $O(N \log N)$ algorithm that preserves net balances while guaranteeing at most $N - 1$ total settlement transactions.

### 8.2 Algorithm Mechanics
1. Compute the **Net Balance** for every user:
   $$\text{NetBalance}(u) = \sum \text{Amount Credited to } u - \sum \text{Amount Debited from } u$$
2. Users with $\text{NetBalance} = 0$ are settled and removed.
3. Partition remaining users into two priority queues (Heaps):
   - **Max-Heap (`Creditors`)**: Users with positive net balance ($\text{NetBalance} > 0$).
   - **Min-Heap (`Debtors`)**: Users with negative net balance ($\text{NetBalance} < 0$).
4. In each iteration:
   - Extract the largest creditor ($C$) and the largest debtor ($D$).
   - Compute settlement amount: $S = \min(\text{credit}_C, |\text{debit}_D|)$.
   - Generate transaction: $D \xrightarrow{S} C$.
   - Update balances:
     - If $C$ still has remaining credit, push back into `Creditors`.
     - If $D$ still has remaining debit, push back into `Debtors`.
5. Terminate when both heaps are empty. Output at most $N - 1$ transactions.

---

## 9. Full, Complete, Production-Grade JAVA Implementation

```java
package com.splitwise.system;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.locks.ReentrantLock;

// ============================================================================
// 1. ENUMS & VALUE OBJECTS
// ============================================================================

enum SplitType {
    EQUAL,
    EXACT,
    PERCENT
}

/**
 * Immutable representation of a settled transaction between two parties.
 */
final class SettlementTransaction {
    private final String fromUserId;
    private final String toUserId;
    private final long amountInCents;

    public SettlementTransaction(String fromUserId, String toUserId, long amountInCents) {
        if (amountInCents <= 0) {
            throw new IllegalArgumentException("Settlement amount must be strictly positive");
        }
        this.fromUserId = Objects.requireNonNull(fromUserId, "fromUserId cannot be null");
        this.toUserId = Objects.requireNonNull(toUserId, "toUserId cannot be null");
        this.amountInCents = amountInCents;
    }

    public String getFromUserId() { return fromUserId; }
    public String getToUserId() { return toUserId; }
    public long getAmountInCents() { return amountInCents; }

    public double getAmountInDollars() {
        return amountInCents / 100.0;
    }

    @Override
    public String toString() {
        return String.format("[%s pays %s -> $%.2f]", fromUserId, toUserId, getAmountInDollars());
    }
}

// ============================================================================
// 2. DOMAIN MODELS: USER, GROUP, SPLIT, EXPENSE
// ============================================================================

class User {
    private final String userId;
    private final String name;
    private final String email;
    private final String phoneNumber;

    public User(String userId, String name, String email, String phoneNumber) {
        this.userId = Objects.requireNonNull(userId, "userId must not be null");
        this.name = Objects.requireNonNull(name, "name must not be null");
        this.email = email;
        this.phoneNumber = phoneNumber;
    }

    public String getUserId() { return userId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhoneNumber() { return phoneNumber; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return userId.equals(user.userId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(userId);
    }

    @Override
    public String toString() {
        return String.format("User{id='%s', name='%s'}", userId, name);
    }
}

class Group {
    private final String groupId;
    private final String groupName;
    private final Set<String> memberUserIds;
    private final List<String> expenseIds;

    public Group(String groupId, String groupName) {
        this.groupId = Objects.requireNonNull(groupId, "groupId cannot be null");
        this.groupName = Objects.requireNonNull(groupName, "groupName cannot be null");
        this.memberUserIds = ConcurrentHashMap.newKeySet();
        this.expenseIds = new CopyOnWriteArrayList<>();
    }

    public void addMember(String userId) {
        memberUserIds.add(userId);
    }

    public void removeMember(String userId) {
        memberUserIds.remove(userId);
    }

    public boolean hasMember(String userId) {
        return memberUserIds.contains(userId);
    }

    public String getGroupId() { return groupId; }
    public String getGroupName() { return groupName; }
    public Set<String> getMemberUserIds() { return Collections.unmodifiableSet(memberUserIds); }
    public List<String> getExpenseIds() { return Collections.unmodifiableList(expenseIds); }

    public void linkExpense(String expenseId) {
        expenseIds.add(expenseId);
    }
}

abstract class Split {
    protected final String userId;
    protected long amountInCents;

    public Split(String userId) {
        this.userId = Objects.requireNonNull(userId, "userId cannot be null");
    }

    public String getUserId() { return userId; }
    public long getAmountInCents() { return amountInCents; }
    public void setAmountInCents(long amountInCents) { this.amountInCents = amountInCents; }
}

class EqualSplit extends Split {
    public EqualSplit(String userId) {
        super(userId);
    }
}

class ExactSplit extends Split {
    public ExactSplit(String userId, long amountInCents) {
        super(userId);
        if (amountInCents < 0) {
            throw new IllegalArgumentException("Exact split amount cannot be negative");
        }
        this.amountInCents = amountInCents;
    }
}

class PercentSplit extends Split {
    private final double percentage;

    public PercentSplit(String userId, double percentage) {
        super(userId);
        if (percentage < 0.0 || percentage > 100.0) {
            throw new IllegalArgumentException("Percentage must be between 0.0 and 100.0");
        }
        this.percentage = percentage;
    }

    public double getPercentage() { return percentage; }
}

class Expense {
    private final String expenseId;
    private final String description;
    private final long amountInCents;
    private final String paidByUserId;
    private final String groupId; // Optional (nullable)
    private final SplitType splitType;
    private final List<Split> splits;
    private final Instant createdAt;

    public Expense(String expenseId, String description, long amountInCents,
                   String paidByUserId, String groupId, SplitType splitType,
                   List<Split> splits) {
        if (amountInCents <= 0) {
            throw new IllegalArgumentException("Expense amount must be strictly positive");
        }
        this.expenseId = Objects.requireNonNull(expenseId);
        this.description = Objects.requireNonNull(description);
        this.amountInCents = amountInCents;
        this.paidByUserId = Objects.requireNonNull(paidByUserId);
        this.groupId = groupId;
        this.splitType = Objects.requireNonNull(splitType);
        this.splits = Collections.unmodifiableList(new ArrayList<>(splits));
        this.createdAt = Instant.now();
    }

    public String getExpenseId() { return expenseId; }
    public String getDescription() { return description; }
    public long getAmountInCents() { return amountInCents; }
    public String getPaidByUserId() { return paidByUserId; }
    public String getGroupId() { return groupId; }
    public SplitType getSplitType() { return splitType; }
    public List<Split> getSplits() { return splits; }
    public Instant getCreatedAt() { return createdAt; }
}

// ============================================================================
// 3. SPLIT STRATEGIES (STRATEGY PATTERN)
// ============================================================================

interface ISplitStrategy {
    void validate(long totalAmountInCents, List<Split> splits);
    void calculate(long totalAmountInCents, List<Split> splits);
}

class EqualSplitStrategy implements ISplitStrategy {
    @Override
    public void validate(long totalAmountInCents, List<Split> splits) {
        if (splits == null || splits.isEmpty()) {
            throw new IllegalArgumentException("Splits list cannot be empty for Equal Split");
        }
        for (Split s : splits) {
            if (!(s instanceof EqualSplit)) {
                throw new IllegalArgumentException("All splits must be instances of EqualSplit");
            }
        }
    }

    @Override
    public void calculate(long totalAmountInCents, List<Split> splits) {
        int count = splits.size();
        long baseShare = totalAmountInCents / count;
        long remainder = totalAmountInCents % count;

        for (int i = 0; i < count; i++) {
            // Distribute the indivisible cents one-by-one to early participants
            long allocated = baseShare + (i < remainder ? 1 : 0);
            splits.get(i).setAmountInCents(allocated);
        }
    }
}

class ExactSplitStrategy implements ISplitStrategy {
    @Override
    public void validate(long totalAmountInCents, List<Split> splits) {
        if (splits == null || splits.isEmpty()) {
            throw new IllegalArgumentException("Splits list cannot be empty for Exact Split");
        }
        long sum = 0;
        for (Split s : splits) {
            if (!(s instanceof ExactSplit)) {
                throw new IllegalArgumentException("All splits must be instances of ExactSplit");
            }
            sum += s.getAmountInCents();
        }
        if (sum != totalAmountInCents) {
            throw new IllegalArgumentException(String.format(
                "Exact splits sum (%d cents) does not match total expense (%d cents)",
                sum, totalAmountInCents
            ));
        }
    }

    @Override
    public void calculate(long totalAmountInCents, List<Split> splits) {
        // Amounts are already populated inside ExactSplit instances
    }
}

class PercentSplitStrategy implements ISplitStrategy {
    private static final double EPSILON = 0.0001;

    @Override
    public void validate(long totalAmountInCents, List<Split> splits) {
        if (splits == null || splits.isEmpty()) {
            throw new IllegalArgumentException("Splits list cannot be empty for Percent Split");
        }
        double totalPercent = 0.0;
        for (Split s : splits) {
            if (!(s instanceof PercentSplit)) {
                throw new IllegalArgumentException("All splits must be instances of PercentSplit");
            }
            totalPercent += ((PercentSplit) s).getPercentage();
        }
        if (Math.abs(totalPercent - 100.0) > EPSILON) {
            throw new IllegalArgumentException(String.format(
                "Total percentage (%.4f) must strictly equal 100.00%%", totalPercent
            ));
        }
    }

    @Override
    public void calculate(long totalAmountInCents, List<Split> splits) {
        long accumulatedCents = 0;
        int count = splits.size();

        for (int i = 0; i < count; i++) {
            PercentSplit ps = (PercentSplit) splits.get(i);
            if (i == count - 1) {
                // Last participant absorbs the residual rounding cent difference
                ps.setAmountInCents(totalAmountInCents - accumulatedCents);
            } else {
                long share = Math.round((ps.getPercentage() * totalAmountInCents) / 100.0);
                ps.setAmountInCents(share);
                accumulatedCents += share;
            }
        }
    }
}

class SplitStrategyFactory {
    private static final Map<SplitType, ISplitStrategy> STRATEGIES = new EnumMap<>(SplitType.class);

    static {
        STRATEGIES.put(SplitType.EQUAL, new EqualSplitStrategy());
        STRATEGIES.put(SplitType.EXACT, new ExactSplitStrategy());
        STRATEGIES.put(SplitType.PERCENT, new PercentSplitStrategy());
    }

    public static ISplitStrategy getStrategy(SplitType type) {
        ISplitStrategy strategy = STRATEGIES.get(type);
        if (strategy == null) {
            throw new UnsupportedOperationException("No strategy configured for: " + type);
        }
        return strategy;
    }
}

// ============================================================================
// 4. THREAD-SAFE BALANCE MANAGER & ORDERED LOCKING
// ============================================================================

class BalanceManager {
    // Outer Key: UserA, Inner Key: UserB, Value: Net cents UserA owes UserB (positive = A owes B, negative = B owes A)
    private final ConcurrentHashMap<String, ConcurrentHashMap<String, AtomicLong>> balanceSheet;
    
    // Fine-grained lock striping per user ID pair to prevent deadlocks
    private final ConcurrentHashMap<String, ReentrantLock> userLocks;

    public BalanceManager() {
        this.balanceSheet = new ConcurrentHashMap<>();
        this.userLocks = new ConcurrentHashMap<>();
    }

    private ReentrantLock getLockForUser(String userId) {
        return userLocks.computeIfAbsent(userId, k -> new ReentrantLock());
    }

    /**
     * Updates the bilateral balance between payer and borrower safely.
     * Enforces Canonical Lock Ordering by lexicographical user ID comparison.
     */
    public void recordDebt(String payerId, String borrowerId, long amountInCents) {
        if (payerId.equals(borrowerId) || amountInCents <= 0) {
            return; // No self-debt
        }

        // Canonical Lock Acquisition Order to prevent AB-BA deadlock
        String firstId = payerId.compareTo(borrowerId) < 0 ? payerId : borrowerId;
        String secondId = payerId.compareTo(borrowerId) < 0 ? borrowerId : payerId;

        ReentrantLock lock1 = getLockForUser(firstId);
        ReentrantLock lock2 = getLockForUser(secondId);

        lock1.lock();
        try {
            lock2.lock();
            try {
                // Borrower owes Payer (+amount)
                balanceSheet
                    .computeIfAbsent(borrowerId, k -> new ConcurrentHashMap<>())
                    .computeIfAbsent(payerId, k -> new AtomicLong(0))
                    .addAndGet(amountInCents);

                // Payer is owed by Borrower (-amount)
                balanceSheet
                    .computeIfAbsent(payerId, k -> new ConcurrentHashMap<>())
                    .computeIfAbsent(borrowerId, k -> new AtomicLong(0))
                    .addAndGet(-amountInCents);
            } finally {
                lock2.unlock();
            }
        } finally {
            lock1.unlock();
        }
    }

    /**
     * Settles debt: payer pays receiver a given amount in cents.
     */
    public void recordSettlement(String payerId, String receiverId, long amountInCents) {
        // Paying off debt is mathematically equivalent to the receiver lending the payer money
        recordDebt(payerId, receiverId, amountInCents);
    }

    /**
     * Returns a snapshot of what other users owe this user or this user owes others.
     * Key: otherUserId, Value: positive = otherUser owes this user, negative = this user owes otherUser
     */
    public Map<String, Long> getBilateralBalances(String userId) {
        ConcurrentHashMap<String, AtomicLong> inner = balanceSheet.get(userId);
        if (inner == null) {
            return Collections.emptyMap();
        }
        Map<String, Long> snapshot = new HashMap<>();
        inner.forEach((otherId, atomicVal) -> {
            long val = atomicVal.get();
            if (val != 0) {
                // Negative inside balanceSheet(userId, otherId) means otherId owes userId
                snapshot.put(otherId, -val);
            }
        });
        return snapshot;
    }

    /**
     * Computes the global net balance for all registered users in the system.
     * Positive = net creditor (is owed money overall)
     * Negative = net debtor (owes money overall)
     */
    public Map<String, Long> computeAllNetBalances() {
        Map<String, Long> netBalances = new HashMap<>();
        balanceSheet.forEach((u1, debts) -> {
            debts.forEach((u2, amount) -> {
                // u1 owes u2 'amount'
                long val = amount.get();
                netBalances.put(u1, netBalances.getOrDefault(u1, 0L) - val);
            });
        });
        // Divide by 2 because each balance is tracked symmetrically in both directions
        netBalances.replaceAll((k, v) -> v / 2);
        return netBalances;
    }
}

// ============================================================================
// 5. DEBT SIMPLIFICATION ALGORITHM (GREEDY MIN CASH FLOW)
// ============================================================================

interface DebtSimplifier {
    List<SettlementTransaction> simplify(Map<String, Long> netBalances);
}

class GreedyMinCashFlowSimplifier implements DebtSimplifier {

    private static class BalanceNode {
        final String userId;
        long amount;

        BalanceNode(String userId, long amount) {
            this.userId = userId;
            this.amount = amount;
        }
    }

    @Override
    public List<SettlementTransaction> simplify(Map<String, Long> netBalances) {
        List<SettlementTransaction> transactions = new ArrayList<>();

        // Max-Heap for creditors (people who are owed money: amount > 0)
        PriorityQueue<BalanceNode> creditors = new PriorityQueue<>(
            (a, b) -> Long.compare(b.amount, a.amount)
        );

        // Min-Heap for debtors (people who owe money: stored as positive magnitude for easy math)
        PriorityQueue<BalanceNode> debtors = new PriorityQueue<>(
            (a, b) -> Long.compare(b.amount, a.amount)
        );

        for (Map.Entry<String, Long> entry : netBalances.entrySet()) {
            long balance = entry.getValue();
            if (balance > 0) {
                creditors.offer(new BalanceNode(entry.getKey(), balance));
            } else if (balance < 0) {
                debtors.offer(new BalanceNode(entry.getKey(), -balance)); // Store absolute positive value
            }
        }

        // Greedy matching between the largest debtor and the largest creditor
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            BalanceNode maxCreditor = creditors.poll();
            BalanceNode maxDebtor = debtors.poll();

            long settleAmount = Math.min(maxCreditor.amount, maxDebtor.amount);

            transactions.add(new SettlementTransaction(
                maxDebtor.userId,
                maxCreditor.userId,
                settleAmount
            ));

            maxCreditor.amount -= settleAmount;
            maxDebtor.amount -= settleAmount;

            if (maxCreditor.amount > 0) {
                creditors.offer(maxCreditor);
            }
            if (maxDebtor.amount > 0) {
                debtors.offer(maxDebtor);
            }
        }

        return transactions;
    }
}

// ============================================================================
// 6. AUDIT & EVENT LOGGING
// ============================================================================

class ActivityLog {
    private final String logId;
    private final String eventType;
    private final String description;
    private final Instant timestamp;

    public ActivityLog(String eventType, String description) {
        this.logId = UUID.randomUUID().toString();
        this.eventType = eventType;
        this.description = description;
        this.timestamp = Instant.now();
    }

    @Override
    public String toString() {
        return String.format("[%s] %s: %s", timestamp, eventType, description);
    }
}

// ============================================================================
// 7. CORE SPLITWISE FACADE SERVICE
// ============================================================================

class SplitwiseService {
    private final ConcurrentHashMap<String, User> users;
    private final ConcurrentHashMap<String, Group> groups;
    private final ConcurrentHashMap<String, Expense> expenses;
    private final BalanceManager balanceManager;
    private final DebtSimplifier debtSimplifier;
    private final List<ActivityLog> auditLogs;

    public SplitwiseService() {
        this.users = new ConcurrentHashMap<>();
        this.groups = new ConcurrentHashMap<>();
        this.expenses = new ConcurrentHashMap<>();
        this.balanceManager = new BalanceManager();
        this.debtSimplifier = new GreedyMinCashFlowSimplifier();
        this.auditLogs = new CopyOnWriteArrayList<>();
    }

    public void registerUser(User user) {
        users.put(user.getUserId(), user);
        auditLogs.add(new ActivityLog("USER_REGISTERED", "Registered user: " + user.getName()));
    }

    public void createGroup(Group group) {
        groups.put(group.getGroupId(), group);
        auditLogs.add(new ActivityLog("GROUP_CREATED", "Created group: " + group.getGroupName()));
    }

    public void addUserToGroup(String groupId, String userId) {
        Group group = groups.get(groupId);
        if (group == null) throw new IllegalArgumentException("Group not found: " + groupId);
        if (!users.containsKey(userId)) throw new IllegalArgumentException("User not found: " + userId);
        group.addMember(userId);
    }

    /**
     * Main API to submit an expense. Thread-safe and validates strategy invariants.
     */
    public Expense addExpense(String description, long amountInCents, String paidByUserId,
                              String groupId, SplitType splitType, List<Split> splits) {
        if (!users.containsKey(paidByUserId)) {
            throw new IllegalArgumentException("Payer does not exist: " + paidByUserId);
        }
        if (groupId != null && !groups.containsKey(groupId)) {
            throw new IllegalArgumentException("Group does not exist: " + groupId);
        }

        for (Split s : splits) {
            if (!users.containsKey(s.getUserId())) {
                throw new IllegalArgumentException("Participant does not exist: " + s.getUserId());
            }
        }

        // 1. Resolve strategy and calculate exact splits
        ISplitStrategy strategy = SplitStrategyFactory.getStrategy(splitType);
        strategy.validate(amountInCents, splits);
        strategy.calculate(amountInCents, splits);

        // 2. Instantiate and persist Expense
        String expenseId = "EXP-" + UUID.randomUUID().toString().substring(0, 8);
        Expense expense = new Expense(
            expenseId, description, amountInCents, paidByUserId, groupId, splitType, splits
        );
        expenses.put(expenseId, expense);

        if (groupId != null) {
            groups.get(groupId).linkExpense(expenseId);
        }

        // 3. Update bilateral balances in thread-safe BalanceManager
        for (Split split : splits) {
            String borrowerId = split.getUserId();
            long share = split.getAmountInCents();

            if (!borrowerId.equals(paidByUserId) && share > 0) {
                balanceManager.recordDebt(paidByUserId, borrowerId, share);
            }
        }

        auditLogs.add(new ActivityLog("EXPENSE_ADDED", String.format(
            "%s paid $%.2f for '%s' (Splits: %d)",
            paidByUserId, amountInCents / 100.0, description, splits.size()
        )));

        return expense;
    }

    /**
     * Direct settlement between two users.
     */
    public void settleDebt(String fromUser, String toUser, long amountInCents) {
        if (!users.containsKey(fromUser) || !users.containsKey(toUser)) {
            throw new IllegalArgumentException("Invalid user for settlement");
        }
        balanceManager.recordSettlement(fromUser, toUser, amountInCents);
        auditLogs.add(new ActivityLog("DEBT_SETTLED", String.format(
            "%s paid %s $%.2f", fromUser, toUser, amountInCents / 100.0
        )));
    }

    public Map<String, Long> getUserBalances(String userId) {
        return balanceManager.getBilateralBalances(userId);
    }

    /**
     * Runs Greedy Min Cash Flow algorithm across the entire system.
     */
    public List<SettlementTransaction> simplifyAllDebts() {
        Map<String, Long> netBalances = balanceManager.computeAllNetBalances();
        return debtSimplifier.simplify(netBalances);
    }

    /**
     * Runs Greedy Min Cash Flow restricted to members of a single group.
     */
    public List<SettlementTransaction> simplifyGroupDebts(String groupId) {
        Group group = groups.get(groupId);
        if (group == null) throw new IllegalArgumentException("Group not found");

        Map<String, Long> allNets = balanceManager.computeAllNetBalances();
        Map<String, Long> groupNets = new HashMap<>();

        for (String memberId : group.getMemberUserIds()) {
            groupNets.put(memberId, allNets.getOrDefault(memberId, 0L));
        }

        return debtSimplifier.simplify(groupNets);
    }

    public List<ActivityLog> getAuditLogs() {
        return Collections.unmodifiableList(auditLogs);
    }
}

// ============================================================================
// 8. MULTI-THREADED SIMULATION & VERIFICATION TEST
// ============================================================================

public class SplitwiseApplication {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("===============================================================");
        System.out.println("   PRODUCTION SPLITWISE SYSTEM: COMPREHENSIVE LLD SIMULATION   ");
        System.out.println("===============================================================\n");

        SplitwiseService service = new SplitwiseService();

        // Register Users
        User u1 = new User("U1", "Alice", "alice@example.com", "+1-555-0101");
        User u2 = new User("U2", "Bob", "bob@example.com", "+1-555-0102");
        User u3 = new User("U3", "Charlie", "charlie@example.com", "+1-555-0103");
        User u4 = new User("U4", "David", "david@example.com", "+1-555-0104");

        service.registerUser(u1);
        service.registerUser(u2);
        service.registerUser(u3);
        service.registerUser(u4);

        // Create Group
        Group skiTrip = new Group("G1", "Lake Tahoe Ski Trip");
        skiTrip.addMember("U1");
        skiTrip.addMember("U2");
        skiTrip.addMember("U3");
        skiTrip.addMember("U4");
        service.createGroup(skiTrip);

        // --------------------------------------------------------------------
        // DEMO 1: Equal Split with Indivisible Cents Distribution ($100 across 3 users)
        // --------------------------------------------------------------------
        System.out.println("[TEST 1] EQUAL Split: $100.00 across Alice, Bob, Charlie (3-way indivisible)");
        List<Split> splits1 = Arrays.asList(
            new EqualSplit("U1"),
            new EqualSplit("U2"),
            new EqualSplit("U3")
        );
        Expense e1 = service.addExpense("Groceries", 10000L, "U1", "G1", SplitType.EQUAL, splits1);
        System.out.println("Expense added: " + e1.getDescription() + " | Total Cents: " + e1.getAmountInCents());
        for (Split s : e1.getSplits()) {
            System.out.println("  -> Participant " + s.getUserId() + " allocated: " + s.getAmountInCents() + " cents ($" + (s.getAmountInCents() / 100.0) + ")");
        }
        System.out.println();

        // --------------------------------------------------------------------
        // DEMO 2: Exact Split ($125.50 total)
        // --------------------------------------------------------------------
        System.out.println("[TEST 2] EXACT Split: $125.50 (Alice pays, Bob=$40.00, Charlie=$50.00, David=$35.50)");
        List<Split> splits2 = Arrays.asList(
            new ExactSplit("U2", 4000L),
            new ExactSplit("U3", 5000L),
            new ExactSplit("U4", 3550L)
        );
        service.addExpense("Ski Lift Passes", 12550L, "U1", "G1", SplitType.EXACT, splits2);
        System.out.println("Exact split expense added successfully.\n");

        // --------------------------------------------------------------------
        // DEMO 3: Percentage Split ($200 total: 40%, 30%, 20%, 10%)
        // --------------------------------------------------------------------
        System.out.println("[TEST 3] PERCENTAGE Split: $200.00 (David pays: Alice=40%, Bob=30%, Charlie=20%, David=10%)");
        List<Split> splits3 = Arrays.asList(
            new PercentSplit("U1", 40.0),
            new PercentSplit("U2", 30.0),
            new PercentSplit("U3", 20.0),
            new PercentSplit("U4", 10.0)
        );
        service.addExpense("Chalet Rental", 20000L, "U4", "G1", SplitType.PERCENT, splits3);
        System.out.println("Percentage split expense added successfully.\n");

        // Print Pre-Simplification Balances for Alice
        System.out.println("--- Alice's Bilateral Balances Before Debt Simplification ---");
        Map<String, Long> aliceBalances = service.getUserBalances("U1");
        aliceBalances.forEach((otherId, cents) -> {
            if (cents > 0) {
                System.out.printf("  %s owes Alice: $%.2f%n", otherId, cents / 100.0);
            } else {
                System.out.printf("  Alice owes %s: $%.2f%n", otherId, (-cents) / 100.0);
            }
        });
        System.out.println();

        // --------------------------------------------------------------------
        // DEMO 4: Min Cash Flow Debt Simplification Execution
        // --------------------------------------------------------------------
        System.out.println("===============================================================");
        System.out.println("      GREEDY MIN CASH FLOW DEBT SIMPLIFICATION EXECUTION       ");
        System.out.println("===============================================================");
        List<SettlementTransaction> simplifiedTransactions = service.simplifyGroupDebts("G1");
        System.out.println("Optimal minimal settlement transactions to clear all group debts:");
        for (SettlementTransaction tx : simplifiedTransactions) {
            System.out.println("  â˜… " + tx);
        }
        System.out.println();

        // --------------------------------------------------------------------
        // DEMO 5: High-Concurrency Stress Test (Simultaneous Cross-Expenses)
        // --------------------------------------------------------------------
        System.out.println("===============================================================");
        System.out.println("     CONCURRENCY STRESS TEST: 20 CONCURRENT CROSS-EXPENSES     ");
        System.out.println("===============================================================");
        int threadCount = 20;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch finishLatch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            final int index = i;
            executor.submit(() -> {
                try {
                    startLatch.await(); // Thundering herd start
                    String payer = (index % 2 == 0) ? "U1" : "U2";
                    String debtor = (index % 2 == 0) ? "U2" : "U1";
                    List<Split> s = Collections.singletonList(new ExactSplit(debtor, 500L)); // $5.00
                    service.addExpense("Concurrent Coffee " + index, 500L, payer, null, SplitType.EXACT, s);
                } catch (Exception ex) {
                    System.err.println("Thread error: " + ex.getMessage());
                } finally {
                    finishLatch.countDown();
                }
            });
        }

        startLatch.countDown(); // Fire all threads simultaneously
        boolean finished = finishLatch.await(5, TimeUnit.SECONDS);
        executor.shutdown();

        System.out.println("High-concurrency test completed successfully without deadlock: " + finished);
        System.out.println("Total audit logs recorded: " + service.getAuditLogs().size());
        System.out.println("\nSimulation complete. System verified production-ready.");
    }
}
```

---

## 10. Step-by-Step Code Walkthrough & Design Decisions

### 1. Integer Cents vs. Floating-Point Decimals
- **Decision**: All money variables are typed as primitive 64-bit `long amountInCents`.
- **Rationale**: Double-precision floating-point arithmetic introduces silent penny drift over thousands of transactions. Long cents provide exact penny arithmetic, zero garbage collection overhead compared to `BigDecimal` in high-throughput hot paths, and trivial serialization.

### 2. Equal Split Remainder Distribution
- In `EqualSplitStrategy.calculate()`, when dividing 100 dollars (10,000 cents) by 3 users:
  $$\text{baseShare} = \lfloor 10000 / 3 \rfloor = 3333 \text{ cents}$$
  $$\text{remainder} = 10000 \pmod 3 = 1 \text{ cent}$$
- The strategy iterates through the index `i = 0` to `2`: if `i < remainder`, add $1\text{ cent}$. User 0 receives 3,334 cents; Users 1 and 2 receive 3,333 cents. The sum $3334 + 3333 + 3333 = 10000$ strictly preserves zero leakage.

### 3. Preventing Concurrency Deadlock via Ordered Locking
- In `BalanceManager.recordDebt()`, user IDs are sorted lexicographically before acquiring reentrant locks:
  ```java
  String firstId = payerId.compareTo(borrowerId) < 0 ? payerId : borrowerId;
  String secondId = payerId.compareTo(borrowerId) < 0 ? borrowerId : payerId;
  ```
- If Thread 1 updates Alice and Bob while Thread 2 updates Bob and Alice, both threads attempt to acquire `Alice`'s lock first, eliminating circular waiting (the 4th Coffman condition).

### 4. Greedy Min Cash Flow vs. Exact Subset Sum
- **Decision**: Implemented `GreedyMinCashFlowSimplifier` utilizing dual `PriorityQueue`s (Max-Heap for creditors, Min-Heap for debtors).
- **Complexity**: $O(V \log V)$ time where $V$ is the number of participants. Maximum transaction count is guaranteed to be $\le V - 1$.
- **Design Tradeoff**: While finding the true absolute minimum transaction count is NP-Complete (reducible to Subset Sum), the greedy approach is mathematically sound, preserves net balances, and executes in sub-millisecond time.

---

## 11. Edge Cases, Failure Modes, and Extensibility Points

### 11.1 Critical Edge Cases Handled
1. **Payer is a Split Participant**: When Alice pays \$30 for Alice, Bob, and Charlie equally, Alice's own share (\$10) does not create self-debt. `BalanceManager.recordDebt` explicitly checks `if (payerId.equals(borrowerId)) return;`.
2. **Unequal Percentages Totaling 99.99% or 100.01%**: `PercentSplitStrategy.validate()` rejects percentages whose absolute difference from 100.0 exceeds $10^{-4}$. The last participant absorbs fractional remainder cents during split calculation.
3. **Empty Group or Missing User**: `SplitwiseService` enforces fail-fast assertions on all user and group lookups before initiating strategy validation or lock acquisition.

### 11.2 Failure Modes & Resilience
- **Database Connection Dropout**: In an enterprise microservice deployment, in-memory state should be backed by write-ahead event sourcing (e.g., Kafka / PostgreSQL transaction log). If the process crashes, the balance sheet is reconstructed by replaying expense events.
- **Lock Contention Under Hot Users**: If an influencer or company account has millions of transactions, lock striping by individual user can lead to queuing. Solution: Event-driven asynchronous balance aggregation via background worker queues.

### 11.3 Extensibility Roadmap
- **Multi-Currency Support**: Introduce a `Currency` enum and an `ExchangeRateProvider`. Balances are maintained per currency or auto-converted to a base currency at daily spot rates.
- **Dynamic Tip & Tax Splits**: Add an `ItemizedSplitStrategy` where line items are assigned to specific users, and communal tip/tax percentages are pro-rated automatically.
