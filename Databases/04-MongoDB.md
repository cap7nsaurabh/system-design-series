# MongoDB: The Document Database Giant

## Overview
MongoDB is an open-source, cross-platform NoSQL document database that stores data in flexible, JSON-like documents (BSON — Binary JSON). It is designed for high availability, horizontal scalability, and developer agility — particularly suited to applications with rapidly evolving schemas or deeply nested, hierarchical data.

> [!NOTE]
> MongoDB's fundamental unit is a **document** (analogous to a row), stored in a **collection** (analogous to a table), inside a **database**. Unlike rows, documents can have completely different structures within the same collection.

## Features
- **Schema Flexibility:** Documents in the same collection can have different fields and structures. Ideal for rapidly evolving product requirements.
- **Rich Query Language:** Supports a powerful, expressive query language with operators for filtering, projection, sorting, and aggregation.
- **Aggregation Pipeline:** A powerful multi-stage data processing framework (similar to Unix pipes) for transformations, joins, grouping, and reshaping data.
- **Horizontal Scaling (Sharding):** Built-in support for distributing data across multiple machines (shards) using a shard key.
- **Replica Sets:** A group of MongoDB instances that maintain the same dataset for high availability and automatic failover.
- **ACID Transactions:** Full multi-document, multi-collection ACID transactions (introduced in v4.0), closing the gap with relational databases.
- **Change Streams:** Real-time event notifications for data changes, enabling reactive applications and ETL pipelines.

## Limitations
- **Memory Usage:** MongoDB's WiredTiger storage engine aggressively caches working set data in RAM. High memory usage is a common operational challenge.
- **No Joins (By Design):** While `$lookup` provides join-like functionality, it is generally a signal of incorrect data modeling. Data should be embedded or denormalized.
- **Document Size Limit:** A single document cannot exceed **16 MB**. For larger data (images, videos), GridFS must be used.
- **Transactions Have a Cost:** Multi-document transactions add significant latency overhead compared to single-document operations. They should be used sparingly.

## Locking Techniques
- **Document-Level Locking:** WiredTiger uses optimistic concurrency control at the document level, only locking a document during an in-flight write, which allows very high concurrency.
- **Intent Locks:** MongoDB uses Intent locks at the database and collection level to coordinate with document-level operations.
- **Snapshot Isolation:** Reads within a session use a consistent snapshot of the data, preventing dirty reads.

## Backend Principles
- **WiredTiger Storage Engine:** MongoDB's default storage engine, featuring document-level concurrency, compression (snappy, zlib, zstd), and a separate write-ahead log (journal).
- **BSON:** Documents are stored internally as BSON (Binary JSON), which is more efficient than plain JSON for storage and traversal and supports additional data types (ObjectId, Date, Binary, Decimal128).
- **ObjectId:** MongoDB's default `_id` field type. A 12-byte value encoding a timestamp (4 bytes), machine identifier (5 bytes), and a random counter (3 bytes) — globally unique and sortable by insertion time.

## Real World Stories

### 1. Why MongoDB Was Created
MongoDB was created in 2007 by Dwight Merriman and Eliot Horowitz, the founders of DoubleClick (an ad tech company later acquired by Google). They experienced firsthand the limitations of relational databases for ad targeting — the schema was constantly changing, and scaling Oracle was prohibitively expensive. They built MongoDB to solve their own problem: a database that was as scalable as key-value stores but as powerful as relational databases.

### 2. Real World Incident — The Ebay Schema Migration Nightmare
eBay's classifieds platform (Kijiji) originally used PostgreSQL but faced extreme difficulty adding new attributes to listings (a car listing needs different fields than an apartment listing). Every new attribute category required a schema migration that locked tables for hours. They migrated to MongoDB, where each listing is a document that can have arbitrary attributes without affecting other documents or requiring schema locks.

### 3. Real World Solution — Coinbase's Real-Time Crypto Data
Coinbase uses MongoDB to store trade history, order books, and user portfolio data. The document model allows them to store rich, nested trade objects (with nested fee breakdown, asset info, and metadata) as a single atomic document, eliminating the 6-table JOIN that the equivalent relational query would require. Their `$lookup` aggregations power the portfolio analytics dashboard.

## Scaling Features and Principles
- **Replica Sets:** A primary node handles all writes; secondary nodes replicate asynchronously. If the primary fails, an election automatically promotes a secondary.
- **Sharding:** MongoDB uses a **shard key** to distribute documents across shards. A bad shard key (low cardinality, monotonically increasing like timestamp) causes hotspots. Good shard keys have high cardinality and even distribution (e.g., `user_id` hashed).
- **Atlas:** MongoDB's managed cloud service handles sharding, backups, and scaling automatically with a few clicks.

---

## Complete MongoDB Operations Reference

### 1. CRUD — Insert Operations

```javascript
// ─────────────────────────────────────────────────────────────────────────────
// insertOne() — Insert a single document
// ─────────────────────────────────────────────────────────────────────────────
// MongoDB auto-generates _id (ObjectId) if not provided
db.users.insertOne({
  name: "Alice Kumar",
  email: "alice@example.com",
  age: 29,
  roles: ["admin", "user"],
  address: {
    city: "Mumbai",
    country: "India"
  },
  createdAt: new Date()
});
// Returns: { acknowledged: true, insertedId: ObjectId("...") }

// ─────────────────────────────────────────────────────────────────────────────
// insertMany() — Insert multiple documents atomically
// ─────────────────────────────────────────────────────────────────────────────
// ordered: true (default) — stops on first error
// ordered: false — continues inserting even if one fails (partial success)
db.products.insertMany([
  { name: "iPhone 15", price: 79999, stock: 100, category: "electronics" },
  { name: "AirPods Pro", price: 24999, stock: 500, category: "electronics" },
  { name: "MacBook Air", price: 114999, stock: 50, category: "computers" }
], { ordered: false });
// Returns: { acknowledged: true, insertedCount: 3, insertedIds: { ... } }
```

---

### 2. CRUD — Find / Read Operations

```javascript
// ─────────────────────────────────────────────────────────────────────────────
// find() — Query documents. Returns a cursor.
// ─────────────────────────────────────────────────────────────────────────────
// Projection: 1 to include, 0 to exclude (_id is included by default)
db.users.find(
  { age: { $gte: 18 } },          // filter
  { name: 1, email: 1, _id: 0 }  // projection (exclude _id)
);

// ─── COMPARISON OPERATORS ────────────────────────────────────────────────────
db.products.find({ price: { $eq: 79999 } });       // equals
db.products.find({ price: { $ne: 79999 } });       // not equals
db.products.find({ price: { $gt: 50000 } });       // greater than
db.products.find({ price: { $gte: 50000 } });      // greater than or equal
db.products.find({ price: { $lt: 30000 } });       // less than
db.products.find({ price: { $lte: 30000 } });      // less than or equal
db.products.find({ category: { $in: ["electronics", "computers"] } });   // in array
db.products.find({ category: { $nin: ["clothing", "furniture"] } });     // not in array

// ─── LOGICAL OPERATORS ───────────────────────────────────────────────────────
// $and — all conditions must match
db.products.find({ $and: [{ price: { $lt: 100000 } }, { category: "electronics" }] });
// Shorthand (implicit AND with multiple conditions):
db.products.find({ price: { $lt: 100000 }, category: "electronics" });

// $or — at least one condition must match
db.products.find({ $or: [{ category: "electronics" }, { price: { $lt: 10000 } }] });

// $nor — no conditions must match (inverse of $or)
db.products.find({ $nor: [{ stock: 0 }, { category: "discontinued" }] });

// $not — negates a single condition
db.products.find({ price: { $not: { $gt: 100000 } } });

// ─── ELEMENT OPERATORS ───────────────────────────────────────────────────────
db.users.find({ phone: { $exists: true } });           // field exists
db.users.find({ phone: { $exists: false } });          // field does not exist
db.users.find({ age: { $type: "int" } });              // field is of type int
db.users.find({ age: { $type: ["int", "double"] } }); // field is int OR double

// ─── EVALUATION OPERATORS ────────────────────────────────────────────────────
// $regex — pattern matching (use indexes only with anchored prefixes: /^pattern/)
db.users.find({ email: { $regex: /.*@gmail\.com$/, $options: "i" } });

// $where — JavaScript expression (slow, no index, avoid in production)
db.users.find({ $where: "this.age > 25 && this.age < 35" });

// ─── ARRAY OPERATORS ─────────────────────────────────────────────────────────
// $all — array must contain all specified values
db.users.find({ roles: { $all: ["admin", "user"] } });

// $elemMatch — at least one array element must match all conditions
// Use when you need multiple conditions on the SAME array element
db.orders.find({ items: { $elemMatch: { product: "iPhone 15", qty: { $gte: 2 } } } });

// $size — array has exactly N elements
db.posts.find({ tags: { $size: 3 } });

// ─── SORT, LIMIT, SKIP ───────────────────────────────────────────────────────
db.products.find()
  .sort({ price: -1, name: 1 })  // -1 = descending, 1 = ascending
  .skip(20)                       // skip first 20 (page 2 with limit 20)
  .limit(20);                     // return at most 20 documents

// ─────────────────────────────────────────────────────────────────────────────
// findOne() — Returns the first matching document (not a cursor)
// ─────────────────────────────────────────────────────────────────────────────
const user = db.users.findOne({ email: "alice@example.com" });
```

---

### 3. CRUD — Update Operations

```javascript
// ─────────────────────────────────────────────────────────────────────────────
// updateOne() / updateMany() — Modify documents using update operators
// ─────────────────────────────────────────────────────────────────────────────

// ─── FIELD UPDATE OPERATORS ──────────────────────────────────────────────────
db.users.updateOne(
  { email: "alice@example.com" },
  {
    $set: { "address.city": "Bangalore", lastLogin: new Date() }, // set/overwrite fields
    $unset: { temporaryToken: "" },   // remove a field entirely
    $inc: { loginCount: 1 },          // increment a numeric field
    $mul: { price: 1.1 },             // multiply a numeric field by factor (10% price increase)
    $rename: { "oldField": "newField" }, // rename a field
    $min: { lowestScore: 42 },        // update only if new value is LOWER than current
    $max: { highScore: 9999 },        // update only if new value is HIGHER than current
    $currentDate: { lastModified: true } // set field to current date
  }
);

// upsert: true — insert if no document matches the filter
db.users.updateOne(
  { email: "new@example.com" },
  { $set: { name: "New User", createdAt: new Date() } },
  { upsert: true }  // If no match, insert the document
);

// ─── ARRAY UPDATE OPERATORS ──────────────────────────────────────────────────
// $push — append to an array
db.posts.updateOne(
  { _id: postId },
  { $push: { comments: { user: "bob", text: "Great post!", timestamp: new Date() } } }
);

// $push with modifiers: $each, $sort, $slice, $position
db.leaderboard.updateOne(
  { gameId: "chess" },
  {
    $push: {
      topScores: {
        $each: [{ user: "alice", score: 9500 }, { user: "bob", score: 8800 }],
        $sort: { score: -1 },  // keep sorted by score descending
        $slice: 10             // keep only top 10 entries
      }
    }
  }
);

// $addToSet — add to array ONLY IF the value doesn't already exist (no duplicates)
db.users.updateOne({ _id: userId }, { $addToSet: { tags: "premium" } });

// $pull — remove matching elements from an array
db.users.updateOne({ _id: userId }, { $pull: { tags: "trial" } });
// $pull with condition:
db.orders.updateOne(
  { _id: orderId },
  { $pull: { items: { status: "cancelled" } } }
);

// $pop — remove first (-1) or last (1) element of an array
db.users.updateOne({ _id: userId }, { $pop: { recentSearches: -1 } }); // remove oldest
db.users.updateOne({ _id: userId }, { $pop: { recentSearches: 1 } });  // remove newest

// $bit — bitwise update
db.items.updateOne({ _id: id }, { $bit: { flags: { or: NumberInt(4) } } });

// ─── findOneAndUpdate / findOneAndReplace / findOneAndDelete ─────────────────
// Returns the document (before or after update). Atomic.
const updatedDoc = db.products.findOneAndUpdate(
  { _id: productId, stock: { $gt: 0 } },
  { $inc: { stock: -1 } },
  {
    returnDocument: "after",  // "before" (default) or "after"
    projection: { name: 1, stock: 1 }
  }
);
// If stock was 0, returns null. If > 0, atomically decrements and returns new doc.
// This is the correct pattern for inventory management — no race condition.
```

---

### 4. CRUD — Delete Operations

```javascript
// deleteOne() — Delete the first matching document
db.users.deleteOne({ email: "spam@example.com" });

// deleteMany() — Delete ALL matching documents
db.sessions.deleteMany({ expiresAt: { $lt: new Date() } }); // delete expired sessions

// findOneAndDelete() — Atomically find, delete, and return the document
const deletedUser = db.users.findOneAndDelete(
  { _id: userId },
  { projection: { name: 1, email: 1 } }
);
```

---

### 5. Bulk Write Operations

```javascript
// bulkWrite() — Send multiple different write operations in a single round trip
// ordered: true (default) — stops on first error; ordered: false — continues on error
db.inventory.bulkWrite([
  // insertOne
  { insertOne: { document: { sku: "WIDGET-X", qty: 100 } } },
  // updateOne
  { updateOne: { filter: { sku: "WIDGET-Y" }, update: { $inc: { qty: -5 } }, upsert: false } },
  // updateMany
  { updateMany: { filter: { qty: { $lt: 10 } }, update: { $set: { status: "low_stock" } } } },
  // replaceOne
  { replaceOne: { filter: { sku: "WIDGET-Z" }, replacement: { sku: "WIDGET-Z", qty: 50, status: "active" } } },
  // deleteOne
  { deleteOne: { filter: { sku: "DISCONTINUED" } } },
  // deleteMany
  { deleteMany: { filter: { status: "archived", createdAt: { $lt: cutoffDate } } } }
], { ordered: false });
```

---

### 6. Aggregation Pipeline — All Major Stages

The aggregation pipeline processes documents through a sequence of stages (like Unix pipes). Each stage transforms the stream of documents.

```javascript
// ─── $match — Filter documents (like WHERE). USE EARLY to reduce data. ────────
// Uses indexes when placed as the first stage.
db.orders.aggregate([
  { $match: { status: "COMPLETED", createdAt: { $gte: new Date("2024-01-01") } } }
]);

// ─── $project — Reshape documents (include/exclude/add computed fields) ───────
db.orders.aggregate([
  { $project: {
    _id: 0,
    orderId: "$_id",
    total: { $sum: "$items.price" },                    // computed field
    year: { $year: "$createdAt" },                      // date expression
    isLargeOrder: { $gt: [{ $sum: "$items.price" }, 10000] }  // boolean computed field
  }}
]);

// ─── $group — Group documents and calculate aggregates (like GROUP BY) ────────
db.orders.aggregate([
  { $group: {
    _id: "$userId",                              // group by userId
    totalSpent: { $sum: "$totalAmount" },        // sum
    avgOrderValue: { $avg: "$totalAmount" },     // average
    orderCount: { $sum: 1 },                     // count
    maxOrder: { $max: "$totalAmount" },          // max
    minOrder: { $min: "$totalAmount" },          // min
    firstOrder: { $first: "$createdAt" },        // first value in group
    lastOrder: { $last: "$createdAt" },          // last value in group
    orderIds: { $push: "$_id" },                 // push all values into array
    uniqueStatuses: { $addToSet: "$status" }     // unique values into set
  }}
]);

// ─── $unwind — Deconstruct an array. Outputs one doc per array element. ──────
// Essential before grouping by array elements.
db.orders.aggregate([
  { $unwind: { path: "$items", preserveNullAndEmpty: false } },
  // Now each document represents one ORDER ITEM, not one ORDER
  { $group: { _id: "$items.category", totalRevenue: { $sum: "$items.price" } } }
]);

// ─── $lookup — Left outer join with another collection ───────────────────────
// Simple join (like SQL LEFT JOIN ... ON)
db.orders.aggregate([
  { $lookup: {
    from: "users",          // collection to join
    localField: "userId",   // field from the left collection (orders)
    foreignField: "_id",    // field from the right collection (users)
    as: "userDetails"       // output array field name
  }},
  { $unwind: "$userDetails" }  // flatten the 1-element array to an object
]);

// Advanced $lookup with pipeline (correlated subquery — join with conditions)
db.orders.aggregate([
  { $lookup: {
    from: "products",
    let: { orderItems: "$items.productId", minQty: 2 },
    pipeline: [
      { $match: { $expr: { $in: ["$_id", "$$orderItems"] } } },
      { $project: { name: 1, price: 1, category: 1 } }
    ],
    as: "productDetails"
  }}
]);

// ─── $addFields — Add new computed fields without removing existing ones ──────
db.users.aggregate([
  { $addFields: {
    fullName: { $concat: ["$firstName", " ", "$lastName"] },
    ageGroup: {
      $switch: {
        branches: [
          { case: { $lt: ["$age", 18] }, then: "minor" },
          { case: { $lt: ["$age", 65] }, then: "adult" }
        ],
        default: "senior"
      }
    }
  }}
]);

// ─── $sort, $limit, $skip ────────────────────────────────────────────────────
db.products.aggregate([
  { $sort: { price: -1, name: 1 } },  // sort by price desc, then name asc
  { $skip: 0 },
  { $limit: 10 }
]);

// ─── $count — Count documents passing through ────────────────────────────────
db.orders.aggregate([
  { $match: { status: "PENDING" } },
  { $count: "pendingOrderCount" }
]);

// ─── $sample — Randomly sample N documents ───────────────────────────────────
db.products.aggregate([{ $sample: { size: 5 } }]);

// ─── $facet — Multi-faceted search (multiple aggregation pipelines in one) ───
// Used for search results with filters + stats + pagination simultaneously
db.products.aggregate([
  { $facet: {
    "paginatedResults": [
      { $match: { category: "electronics" } },
      { $sort: { price: 1 } },
      { $skip: 0 }, { $limit: 20 }
    ],
    "totalCount": [
      { $match: { category: "electronics" } },
      { $count: "count" }
    ],
    "priceStats": [
      { $match: { category: "electronics" } },
      { $group: { _id: null, avg: { $avg: "$price" }, max: { $max: "$price" }, min: { $min: "$price" } } }
    ],
    "categoryBreakdown": [
      { $group: { _id: "$subcategory", count: { $sum: 1 } } }
    ]
  }}
]);

// ─── $bucket / $bucketAuto — Categorize into price ranges ───────────────────
db.products.aggregate([
  { $bucket: {
    groupBy: "$price",
    boundaries: [0, 10000, 50000, 100000, 500000],
    default: "Other",
    output: { count: { $sum: 1 }, avgPrice: { $avg: "$price" } }
  }}
]);

// ─── $graphLookup — Recursive graph traversal (friends-of-friends, org charts) ─
db.employees.aggregate([
  { $match: { name: "Alice" } },
  { $graphLookup: {
    from: "employees",
    startWith: "$reportsTo",      // start from this employee's manager
    connectFromField: "reportsTo", // follow this field recursively
    connectToField: "name",        // match against this field
    as: "reportingHierarchy",      // output array
    maxDepth: 5,                   // max levels to traverse
    depthField: "level"            // add depth level to each result
  }}
]);

// ─── $merge / $out — Write pipeline results to a collection ──────────────────
// $out: replaces the target collection entirely
db.orders.aggregate([
  { $group: { _id: "$userId", totalSpent: { $sum: "$totalAmount" } } },
  { $out: "user_spending_summary" }  // creates/replaces the collection
]);

// $merge: upsert/merge results into target collection (more flexible)
db.orders.aggregate([
  { $group: { _id: { userId: "$userId", month: { $month: "$createdAt" } }, total: { $sum: "$totalAmount" } } },
  { $merge: {
    into: "monthly_spending",
    on: "_id",
    whenMatched: "merge",  // merge fields into existing docs
    whenNotMatched: "insert"
  }}
]);

// ─── $replaceRoot — Replace the document root with a nested sub-document ─────
db.orders.aggregate([
  { $unwind: "$items" },
  { $replaceRoot: { newRoot: "$items" } }
  // Now each document is an item, not an order
]);
```

---

### 7. Aggregation Expressions

```javascript
// Arithmetic: $add, $subtract, $multiply, $divide, $mod, $abs, $ceil, $floor, $round, $sqrt, $pow, $log, $exp
{ $project: { profit: { $subtract: ["$revenue", "$cost"] } } }

// String: $concat, $substr, $toLower, $toUpper, $trim, $ltrim, $rtrim, $split, $indexOfCP, $strLenCP, $replaceAll
{ $project: { slug: { $toLower: { $replaceAll: { input: "$name", find: " ", replacement: "-" } } } } }

// Date: $year, $month, $dayOfMonth, $hour, $minute, $second, $dayOfWeek, $week, $dateToString, $dateFromString, $dateDiff, $dateAdd
{ $project: { year: { $year: "$createdAt" }, formatted: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } } } }

// Conditional: $cond, $ifNull, $switch
{ $project: {
  discount: { $cond: { if: { $gte: ["$totalAmount", 10000] }, then: 0.1, else: 0 } },
  displayName: { $ifNull: ["$username", "$email"] }  // use email if username is null
}}

// Array: $map (transform), $filter (subset), $reduce (fold), $size, $first, $last, $arrayElemAt, $concatArrays, $setIntersection, $setUnion, $setDifference, $in (check membership)
{ $project: {
  expensiveItems: { $filter: { input: "$items", as: "item", cond: { $gt: ["$$item.price", 1000] } } },
  itemNames: { $map: { input: "$items", as: "item", in: "$$item.name" } },
  totalQty: { $reduce: { input: "$items", initialValue: 0, in: { $add: ["$$value", "$$this.qty"] } } }
}}
```

---

### 8. Indexes

```javascript
// ─── Basic B-Tree Index ───────────────────────────────────────────────────────
db.users.createIndex({ email: 1 });                     // ascending
db.users.createIndex({ createdAt: -1 });                // descending

// ─── Compound Index (order matters! follows ESR rule: Equality, Sort, Range) ──
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 });

// ─── Unique Index ────────────────────────────────────────────────────────────
db.users.createIndex({ email: 1 }, { unique: true });
// Sparse unique — allows multiple docs with missing field (but not duplicate values)
db.users.createIndex({ phone: 1 }, { unique: true, sparse: true });

// ─── Partial Index — Index only a subset of documents (saves space!) ─────────
// Only index active users — queries on inactive users do a full scan
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
);

// ─── TTL Index — Automatically expire documents after a set time ──────────────
// Sessions expire 1 hour after `expiresAt`
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
// OR: expire 1 hour after the document was created
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 });

// ─── Text Index — Full-text search ───────────────────────────────────────────
db.articles.createIndex({ title: "text", body: "text" }, { weights: { title: 10, body: 1 } });
// Query with text index:
db.articles.find({ $text: { $search: "mongodb indexing performance" } },
                 { score: { $meta: "textScore" } })
           .sort({ score: { $meta: "textScore" } });

// ─── 2dsphere Index — Geospatial queries ────────────────────────────────────
db.restaurants.createIndex({ location: "2dsphere" });
// Find restaurants within 2km of a point:
db.restaurants.find({
  location: { $near: { $geometry: { type: "Point", coordinates: [77.5946, 12.9716] }, $maxDistance: 2000 } }
});

// ─── Wildcard Index — Index all fields in a document ─────────────────────────
// Useful when you don't know which fields will be queried (e.g., custom attributes)
db.products.createIndex({ "attributes.$**": 1 });

// ─── Hashed Index — For hash-based sharding ──────────────────────────────────
db.users.createIndex({ userId: "hashed" });

// ─── Multikey Index — Automatically created when indexing an array field ─────
db.articles.createIndex({ tags: 1 });  // indexes each tag string separately

// ─── EXPLAIN — Understand query execution ────────────────────────────────────
db.orders.find({ userId: "user123", status: "COMPLETED" })
         .sort({ createdAt: -1 })
         .explain("executionStats");
// Look for: "stage": "IXSCAN" (good) vs "COLLSCAN" (bad, full collection scan)
// Check: "totalKeysExamined" vs "totalDocsExamined" vs "nReturned"

// hint() — Force use of a specific index
db.orders.find({ userId: "user123" }).hint({ userId: 1, status: 1, createdAt: -1 });
```

---

### 9. Multi-Document ACID Transactions

```javascript
// Transactions are needed for multi-collection writes requiring atomicity
// e.g., transferring funds between two accounts

const session = db.getMongo().startSession();
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});

try {
  const accountsCollection = session.getDatabase("bank").accounts;

  // Debit sender
  const senderUpdate = await accountsCollection.updateOne(
    { _id: senderId, balance: { $gte: amount } }, // Check sufficient funds!
    { $inc: { balance: -amount } },
    { session }
  );

  if (senderUpdate.matchedCount === 0) {
    throw new Error("Insufficient funds or sender not found");
  }

  // Credit receiver
  await accountsCollection.updateOne(
    { _id: receiverId },
    { $inc: { balance: amount } },
    { session }
  );

  // Write to transaction ledger
  await session.getDatabase("bank").transactions.insertOne({
    from: senderId, to: receiverId, amount,
    timestamp: new Date(), status: "completed"
  }, { session });

  await session.commitTransaction();
  console.log("Transfer successful!");
} catch (error) {
  await session.abortTransaction(); // Rolls back ALL changes
  console.error("Transfer failed, rolled back:", error.message);
} finally {
  session.endSession();
}
```

---

### 10. Change Streams — Real-Time Event Subscriptions

```javascript
// Change Streams allow applications to react to data changes in real time
// Use case: Sync MongoDB to Elasticsearch, trigger notifications, audit logging

// Watch changes on a single collection
const changeStream = db.orders.watch([
  // Optional pipeline to filter events (reduces network traffic)
  { $match: { "operationType": { $in: ["insert", "update"] } } },
  { $match: { "fullDocument.status": "PAYMENT_CONFIRMED" } }
], {
  fullDocument: "updateLookup"  // Include the full updated document (not just the diff)
});

changeStream.on("change", (change) => {
  console.log("Operation:", change.operationType); // insert, update, delete, replace
  console.log("Document:", change.fullDocument);
  console.log("Changed fields:", change.updateDescription?.updatedFields);

  // Example: When a new order is confirmed, trigger inventory update
  if (change.operationType === "insert") {
    triggerInventoryReduction(change.fullDocument);
    sendOrderConfirmationEmail(change.fullDocument.userId);
  }
});

// Resume a change stream after a disconnect using resume token
const resumeToken = changeStream.resumeToken;
const resumedStream = db.orders.watch([], { resumeAfter: resumeToken });
```

---

### 11. GridFS — Storing Large Files (>16 MB)

```javascript
// GridFS splits large files into 255 KB chunks stored in two collections:
// fs.files (metadata) and fs.chunks (binary data)

const { MongoClient, GridFSBucket } = require("mongodb");
const client = await MongoClient.connect(uri);
const db = client.db("myApp");
const bucket = new GridFSBucket(db, { bucketName: "uploads" });

// Upload a file
const fs = require("fs");
const uploadStream = bucket.openUploadStream("profile-photo.jpg", {
  metadata: { userId: "user123", contentType: "image/jpeg" }
});
fs.createReadStream("/local/path/photo.jpg").pipe(uploadStream);
uploadStream.on("finish", () => console.log("Upload complete! ID:", uploadStream.id));

// Download a file
const downloadStream = bucket.openDownloadStreamByName("profile-photo.jpg");
downloadStream.pipe(fs.createWriteStream("/output/photo.jpg"));

// List files with metadata
const files = await bucket.find({ "metadata.userId": "user123" }).toArray();
```

---

## Interview Quick Reference

| Question | Answer |
|----------|--------|
| When to embed vs reference? | Embed when data is read together frequently and child data is bounded in size. Reference when child data is large, shared by multiple parents, or independently queried. |
| What is the 16 MB limit? | Max single document size. Use GridFS for larger binary data. |
| What makes `$lookup` slow? | It can't use indexes on the joined collection in many cases, causing collection scans. Embed data to avoid joins. |
| Difference between `$match` position? | Early `$match` uses indexes. `$match` after `$unwind` cannot use the original collection index. |
| What is a hotspot shard? | When all writes go to one shard (e.g., using a monotonically increasing shard key like timestamp). Use hashed shard keys or a high-cardinality field. |
| ACID transactions cost? | Multi-document transactions have 2-5x latency overhead vs single-document ops. Use sparingly. |
