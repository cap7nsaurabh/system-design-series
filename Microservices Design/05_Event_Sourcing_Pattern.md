# Event Sourcing Pattern

## 1. Pattern Overview and Intent
The **Event Sourcing Pattern** is a software architecture pattern where every change to the state of an application is captured as an event and stored in an append-only log, known as the Event Store. Instead of storing the current state of a domain entity, the application stores a sequence of events. The current state is then derived by replaying these events sequentially.

**Intent:**
- Guarantee absolute certainty about what happened and when.
- Provide a robust audit trail out-of-the-box.
- Decouple state persistence from current state querying.
- Enable temporal queries (e.g., "what did this entity look like at time T?").

## 2. Detailed Explanation of How it Works

In a traditional CRUD system, an update replaces the previous state. In Event Sourcing:
1. **Command:** The system receives a command (e.g., `UpdateCart`).
2. **Event Creation:** If the command is valid, it generates an event (e.g., `ItemAddedToCart`).
3. **Appending:** This event is appended to an **Event Store**. The event store is the single source of truth.
4. **State Reconstruction:** To read the state, the system queries the event store for all events related to the entity and applies them to an initial state sequentially (called replaying).
5. **Snapshots:** Replaying all events can be slow. A snapshotting mechanism periodically saves the reconstructed state. Future replays start from the latest snapshot and apply only subsequent events.

### Architecture Diagram

```mermaid
flowchart TD
    Client[Client] -->|1. Command (e.g. AddItem)| API[API Gateway]
    API -->|2. Route Command| CmdSvc[Command Service]
    CmdSvc -->|3. Validate & Generate Event| EventStore[(Event Store / Event Log)]
    EventStore -->|4. Store Event| EventStore
    EventStore -->|5. Publish Event| MessageBus((Message Bus / Kafka))
    MessageBus -->|6. Consume Event| ReadModelUpdater[Projector / Updater]
    ReadModelUpdater -->|7. Update View| ReadDB[(Read Database)]
    Client -->|8. Query State| QuerySvc[Query Service]
    QuerySvc -->|9. Fetch Data| ReadDB
```

## 3. Real-World Example: E-Commerce Shopping Cart

Imagine an e-commerce platform where a user interacts with a shopping cart.

### The Events
Instead of storing `{ cart_id: 1, items: ['apple', 'banana'], total: 5.00 }`, we store:
- `CartCreated` (timestamp: T1, user: Alice)
- `ItemAdded` (timestamp: T2, item: Apple, qty: 1, price: $2)
- `ItemAdded` (timestamp: T3, item: Banana, qty: 1, price: $3)
- `ItemRemoved` (timestamp: T4, item: Apple, qty: 1)

### Code Snippet (Pseudocode)

```python
class Event:
    pass

class CartCreated(Event):
    def __init__(self, cart_id):
        self.cart_id = cart_id

class ItemAdded(Event):
    def __init__(self, cart_id, item, qty):
        self.cart_id = cart_id
        self.item = item
        self.qty = qty

class ShoppingCart:
    def __init__(self):
        self.cart_id = None
        self.items = {}

    def apply(self, event):
        if isinstance(event, CartCreated):
            self.cart_id = event.cart_id
        elif isinstance(event, ItemAdded):
            self.items[event.item] = self.items.get(event.item, 0) + event.qty

def reconstruct_cart(events):
    cart = ShoppingCart()
    for event in events:
        cart.apply(event)
    return cart

# Usage
events_from_store = get_events_for_cart("cart-123")
current_cart = reconstruct_cart(events_from_store)
```

## 4. Pros and Cons

### Pros
- **Complete Audit Trail:** You never lose data; you know exactly how the system reached its current state.
- **Time Travel (Temporal Queries):** You can query the system's state at any point in time.
- **Performance:** Appending to a log is incredibly fast compared to updating indexed records in a traditional DB.
- **Integration with CQRS:** Fits perfectly with CQRS, allowing read models to be built from the event stream.

### Cons
- **Complexity:** Higher learning curve. It changes how developers think about persistence.
- **Eventual Consistency:** The read model is often updated asynchronously, leading to eventual consistency.
- **Event Schema Evolution:** Changing the structure of an event (versioning) is difficult because historical events cannot be modified.
- **Storage Size:** Event stores can grow indefinitely.

## 5. When to Use / When NOT to Use

**When to Use:**
- When a robust audit log is a business requirement (e.g., banking, healthcare).
- Systems where the history of an entity is just as important as its current state.
- Complex domains where state transitions are intricate.

**When NOT to Use:**
- Simple CRUD applications.
- When eventual consistency is unacceptable.
- Applications with a very tight deadline and a team unfamiliar with the pattern.

## 6. Common Interview Questions
1. **How do you handle event schema changes (versioning) in Event Sourcing?**
   *Answer:* Use upcasting. When reading an old event, an upcaster converts it into the new format before passing it to the application logic. Alternatively, store multiple versions of the event handler.
2. **How does Event Sourcing deal with performance issues during state reconstruction?**
   *Answer:* Using Snapshots. A snapshot is a serialized version of the state at a specific point. To reconstruct, the system loads the latest snapshot and replays only the events that occurred after the snapshot.
3. **What is the relationship between Event Sourcing and CQRS?**
   *Answer:* They often go hand-in-hand. Event sourcing handles the write side (commands generate events), while CQRS handles the read side (projectors listen to the events and build optimized read models).
