# 🔄 Data Pipelines & Analytics

> *"Data is only as valuable as the insights you can derive from it — and only as fast as the pipeline that delivers it."*

---

## 📋 Table of Contents
1. [Batch vs Stream Processing](#batch-vs-stream)
2. [Apache Spark — Batch Processing](#spark)
3. [Apache Flink — Stream Processing](#flink)
4. [The Lambda & Kappa Architectures](#lambda-kappa)
5. [Data Lakehouse Architecture](#lakehouse)
6. [Apache Kafka as a Data Backbone](#kafka-backbone)
7. [ETL vs ELT](#etl-vs-elt)
8. [Change Data Capture (CDC)](#cdc)
9. [OLAP vs OLTP & Columnar Storage](#olap)
10. [Real-Life Scenario](#real-life-scenario)
11. [🔥 Big Tech Story: Uber's Real-Time Analytics](#big-tech-story)
12. [Trade-offs & Best Practices](#trade-offs)
13. [Key Takeaways](#key-takeaways)

---

## ⚡ Batch vs Stream Processing

| Dimension | Batch Processing | Stream Processing |
|---|---|---|
| **Data** | Bounded (has a defined end) | Unbounded (infinite, continuous) |
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Throughput** | Very high | Moderate to high |
| **Complexity** | Lower | Higher |
| **Use case** | Historical reports, nightly aggregations | Real-time dashboards, fraud detection, live recommendations |
| **Failure handling** | Simple (re-run the job) | Complex (exactly-once processing) |
| **Examples** | Hadoop MapReduce, Spark batch | Flink, Kafka Streams, Spark Streaming |

**When to use batch:**
- Nightly sales reports
- Monthly invoice generation
- ML model training on historical data
- Database migration
- Data warehouse ETL

**When to use stream:**
- Real-time fraud detection (must stop a transaction NOW)
- Live sports score feeds
- IoT sensor monitoring and alerting
- Real-time recommendation updates ("Users who viewed this also bought...")
- Financial market data processing
- Live monitoring dashboards

---

## 🔥 Apache Spark — Batch Processing Deep Dive

Spark is the industry standard for large-scale distributed data processing.

### Core Architecture

```
Driver Program (coordinator):
  - Analyzes your code
  - Creates a DAG (Directed Acyclic Graph) of operations
  - Schedules tasks to Executors

Cluster Manager (YARN/Kubernetes/Standalone):
  - Allocates resources (CPUs, memory) for Executors

Executors (workers):
  - Each runs on a worker node
  - Executes tasks in parallel
  - Cache data in memory (RDDs, DataFrames)

DAG Scheduler → Task Scheduler → Executor
```

### Spark Execution Model

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder \
    .appName("SalesAnalysis") \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", "4") \
    .getOrCreate()

# Read from S3 (lazy evaluation — nothing executes yet)
orders_df = spark.read.parquet("s3://data-lake/orders/2024/")
products_df = spark.read.parquet("s3://data-lake/products/")

# Transform (still lazy)
result = orders_df \
    .filter(F.col("status") == "COMPLETED") \
    .join(products_df, "product_id") \
    .groupBy("category", F.date_format("created_at", "yyyy-MM").alias("month")) \
    .agg(
        F.sum("amount").alias("total_revenue"),
        F.count("*").alias("order_count"),
        F.avg("amount").alias("avg_order_value")
    ) \
    .orderBy("month", F.desc("total_revenue"))

# Action (triggers actual execution)
result.write.mode("overwrite").parquet("s3://data-warehouse/monthly_revenue/")
```

**Why Spark is fast:**
1. **In-memory processing:** Data is cached in executors' memory between stages. MapReduce wrote to disk between every stage.
2. **DAG optimization:** Spark optimizes the execution plan (Catalyst optimizer). Pushes filters down to minimize data read.
3. **Columnar execution (Tungsten):** Processes data in columnar format for vectorized operations (SIMD CPU instructions).
4. **Lazy evaluation:** Builds the full plan before executing → enables end-to-end optimization.

### Partitioning (Critical for Performance)

Data is split into partitions processed in parallel across executors.

```python
# Bad: default 200 partitions for 1TB of data → huge partitions, slow
df = spark.read.parquet("s3://data-lake/events/")

# Good: Repartition based on data size and cluster capacity
# Rule of thumb: partition size 100-200MB
df = spark.read.parquet("s3://data-lake/events/") \
    .repartition(2000)  # 1TB / 2000 = 512MB per partition... still large

# Even better: repartition by key to avoid shuffle during join
df = df.repartitionByRange(2000, "user_id")

# Check partition distribution
df.rdd.getNumPartitions()
```

**Data skew problem:** If one partition has 10x more data than others (e.g., user_id=1 is a bot with 1 billion events), that partition becomes the bottleneck.

**Solutions:**
1. Salting: `user_id_salted = user_id + "_" + random(0,9)` — splits skewed key across 10 keys
2. Broadcast join: for small tables, broadcast to all executors (no shuffle needed)

### Spark Structured Streaming

Spark also supports streaming by treating a stream as an infinite DataFrame with micro-batches.

```python
# Read from Kafka as a stream
stream_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "orders") \
    .load()

# Parse and aggregate
result = stream_df \
    .select(F.from_json(F.col("value").cast("string"), order_schema).alias("order")) \
    .select("order.*") \
    .groupBy(F.window("created_at", "1 minute"), "category") \
    .agg(F.sum("amount").alias("revenue_per_minute"))

# Write to console (or Kafka, Delta Lake, etc.)
query = result.writeStream \
    .format("delta") \
    .option("checkpointLocation", "s3://checkpoints/revenue-stream/") \
    .trigger(processingTime="30 seconds") \  # Micro-batch every 30s
    .start("s3://data-lake/realtime-revenue/")
```

---

## 🌊 Apache Flink — True Stream Processing

Flink treats streams as the primary abstraction. Batch is a special case of streaming (bounded stream).

**Key advantages over Spark Streaming:**
- True event-time processing (not micro-batches)
- Exactly-once stateful processing natively
- Lower latency (no mini-batch delay)

### Event Time vs Processing Time

```
Event time:      When the event actually occurred (timestamp in the event data)
Processing time: When the event is processed by Flink

These differ! A mobile app event created at 10:00am may arrive at Flink at 10:05am
due to network latency or the phone being offline.

With processing time:
  The 10:00am event processed at 10:05am is counted in the 10:05am window — WRONG

With event time:
  The 10:00am event (late by 5 min) is counted in the 10:00am window — CORRECT
  But: how long do you wait for late events? (watermarks)
```

### Watermarks

Watermarks tell Flink how late events are expected to be.

```java
DataStream<Event> events = env.fromSource(kafkaSource, ...)
    .assignTimestampsAndWatermarks(
        WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
        .withTimestampAssigner((event, ts) -> event.getTimestamp())
    );

// Flink waits for events up to 5 seconds late
// When watermark advances past window end + 5 seconds → window is closed and computed
```

### Flink State Management

Flink's killer feature: **managed, fault-tolerant, exactly-once state**.

```java
// Stateful stream processing: count events per user
DataStream<Tuple2<String, Long>> userCounts = events
    .keyBy(event -> event.getUserId())
    .process(new KeyedProcessFunction<String, Event, Tuple2<String, Long>>() {
        
        // State: one counter per user, stored in RocksDB (persistent)
        private ValueState<Long> countState;
        
        @Override
        public void open(Configuration config) {
            countState = getRuntimeContext().getState(
                new ValueStateDescriptor<>("count", Long.class, 0L)
            );
        }
        
        @Override
        public void processElement(Event event, Context ctx, Collector<Tuple2<String, Long>> out) throws Exception {
            Long count = countState.value();
            count++;
            countState.update(count);
            out.collect(Tuple2.of(event.getUserId(), count));
        }
    });
```

State is stored in **RocksDB** on local disk, with periodic snapshots (**checkpoints**) to S3. On failure, Flink restores from the last checkpoint and replays Kafka messages from that offset — achieving **exactly-once** semantics.

---

## Λ The Lambda & Kappa Architectures

### Lambda Architecture

Combines batch and stream processing to get both accuracy and low latency.

```
Incoming Data
    │
    ├──► Batch Layer (Spark/Hadoop):
    │      - Processes ALL historical data
    │      - Computes accurate, complete results
    │      - Output: Batch views (updated hourly/daily)
    │
    └──► Speed Layer (Flink/Kafka Streams):
           - Processes only recent data
           - Provides low-latency approximations
           - Output: Real-time views (seconds old)
           
Serving Layer:
    - Merges Batch views + Real-time views for queries
    - Batch view for old data (accurate)
    - Real-time view for recent data (approximate until batch catches up)
```

**Problems with Lambda:**
- Two separate codebases for same business logic (batch and stream) → complexity
- Operational burden of maintaining two systems
- Batch views can lag by hours; real-time view is eventually replaced

### Kappa Architecture

Simplify by using ONLY stream processing. Reprocess historical data by replaying Kafka.

```
All data → Kafka (retain forever or long retention) → Flink (single stream processor)

For historical reprocessing:
  1. Deploy new version of Flink job with new logic
  2. Flink reads Kafka from the beginning (time offset 0)
  3. Flink produces new output
  4. Once caught up → swap the "live" output table

No batch layer! Single codebase. Simpler operations.
```

**When Kappa works:** When your data volume is manageable in Kafka (weeks/months of retention) and stream processing can reprocess within acceptable time.

**When Lambda is needed:** When historical data is too large for Kafka retention (years of data) or when batch jobs are needed for complex machine learning pipelines.

---

## 🏠 Data Lakehouse Architecture

The evolution of data architecture: combining the best of Data Warehouses and Data Lakes.

### Data Warehouse (Traditional)

```
Structured data → ETL → Data Warehouse (Redshift, BigQuery, Snowflake)

✅ ACID transactions, schema enforcement, SQL interface
✅ Very fast analytical queries (columnar storage)
❌ Expensive, proprietary format, can't store raw/unstructured data
❌ Schema changes are painful (must re-ETL)
```

### Data Lake

```
All data (raw) → S3/GCS/ADLS → Query with Spark/Athena

✅ Cheap storage (S3)
✅ Any file format (JSON, CSV, Parquet, images, video)
✅ Schema-on-read flexibility
❌ No ACID transactions → "data swamp"
❌ No versioning → accidentally overwrite data
❌ Slow queries (no statistics, no indexing)
```

### Data Lakehouse (Delta Lake, Apache Iceberg, Apache Hudi)

Open table formats that add data warehouse features ON TOP of object storage.

```
Data Lake Storage (S3) + Open Table Format (Delta Lake/Iceberg) = Lakehouse

Features added by table formats:
  ✅ ACID transactions (atomic commits to the table)
  ✅ Time travel (query data as it was 30 days ago)
  ✅ Schema evolution (add/rename columns without rewriting data)
  ✅ Z-ordering (co-locate related data for faster filtering)
  ✅ Partition pruning (skip irrelevant partitions)
  ✅ Bloom filters (skip files that don't contain a key)
  ✅ Concurrent reads and writes
```

**Delta Lake (Databricks):**

```python
from delta.tables import DeltaTable
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.0.0") \
    .getOrCreate()

# Write as Delta format
df.write.format("delta").save("s3://data-lake/delta/orders/")

# ACID UPDATE (not possible in raw Parquet!)
delta_table = DeltaTable.forPath(spark, "s3://data-lake/delta/orders/")
delta_table.update(
    condition="status = 'PENDING' AND created_at < '2024-01-01'",
    set={"status": "'ABANDONED'"}
)

# Time travel: query data as it was 7 days ago
historical_df = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-08") \
    .load("s3://data-lake/delta/orders/")

# Or by version number
spark.read.format("delta").option("versionAsOf", "10").load("...")

# MERGE (upsert): common pattern for CDC
delta_table.alias("target") \
    .merge(updates_df.alias("source"), "target.order_id = source.order_id") \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()
```

**Apache Iceberg (Netflix's contribution, now Apache):**

Similar to Delta Lake but with more flexible spec. Supported by many engines (Spark, Flink, Presto, Trino, Hive, AWS Athena, Snowflake).

---

## 🔗 Kafka as a Data Backbone

In modern data architectures, Kafka is the central nervous system for data movement.

```
                        ┌─── Kafka ───────────────────────────────────────────────────┐
                        │ Topics: orders, payments, user_events, inventory_updates     │
                        └─────────────────────────────────────────────────────────────┘
                              ▲               │
         ┌────────────────────┘               └──────────────────────────────────┐
         │                                                                        │
Sources: OLTP DBs (via CDC)                  Consumers:
         Apps                                  - Flink (real-time analytics)
         IoT sensors                           - Spark (batch analytics → S3)
         Clickstream                           - Elasticsearch (search index)
                                               - Data Warehouse (Redshift/BigQuery)
                                               - ML training data lake
```

Kafka serves as the **single source of truth** for events. Each consumer group independently reads the data at its own pace. Replay is possible by resetting consumer offsets.

---

## 🔄 ETL vs ELT

**ETL (Extract-Transform-Load) — Traditional:**
```
Source DB → [Extract] → [Transform in pipeline] → [Load] → Data Warehouse
              ↑ raw data     ↑ cleaned, structured     ↑ clean data
```

Transformation happens BEFORE loading. Works well when warehouse storage is expensive (must load only clean data). Requires complex pipeline code.

**ELT (Extract-Load-Transform) — Modern Cloud:**
```
Source DB → [Extract] → [Load raw] → Data Warehouse/Lakehouse → [Transform with SQL]
              ↑ raw          ↑ raw data stored           ↑ dbt transforms SQL
```

Load ALL data first (cheap cloud storage). Transform inside the warehouse using SQL (very powerful engines). 

**dbt (Data Build Tool):** The standard ELT transformation tool. Write SQL models that reference each other; dbt generates the DAG and runs them in order.

```sql
-- models/fact_orders.sql (dbt model)
{{config(materialized='incremental', unique_key='order_id')}}

SELECT
    o.order_id,
    o.user_id,
    u.country,
    p.category,
    o.amount,
    o.status,
    o.created_at
FROM {{ ref('stg_orders') }} o  -- ref() creates dependency graph
JOIN {{ ref('stg_users') }} u ON o.user_id = u.user_id
JOIN {{ ref('stg_products') }} p ON o.product_id = p.product_id
{% if is_incremental() %}
WHERE o.created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

`dbt run` → executes all models in dependency order. `dbt test` → validates data quality. `dbt docs generate` → produces data lineage documentation.

---

## 📸 Change Data Capture (CDC)

**CDC** is the technique of capturing every change (INSERT, UPDATE, DELETE) to a database and streaming those changes to downstream systems.

### Why CDC?

Instead of batch ETL (nightly full-table dumps), CDC provides:
- Real-time data synchronization
- Only changes are transferred (no full table scans)
- Complete audit trail of every change
- Ability to rebuild any downstream view from the change log

### How CDC Works (Debezium)

```
PostgreSQL WAL (Write-Ahead Log):
  Every INSERT/UPDATE/DELETE is written to WAL before being applied

Debezium (CDC tool):
  - Reads WAL as a logical replication stream
  - Converts WAL entries to structured change events
  - Publishes to Kafka

Kafka topics:
  db.myschema.orders: { "op": "u", "before": {...}, "after": {...} }
  db.myschema.users:  { "op": "c", "after": {...} }
  db.myschema.products: { "op": "d", "before": {...} }
```

```json
// Debezium change event for an order update
{
  "op": "u",                    // u=update, c=create, d=delete, r=read(snapshot)
  "ts_ms": 1705312000000,
  "before": {
    "order_id": 456,
    "status": "PENDING",
    "amount": 99.99
  },
  "after": {
    "order_id": 456,
    "status": "SHIPPED",       // ← what changed
    "amount": 99.99
  },
  "source": {
    "db": "myapp",
    "table": "orders",
    "lsn": 14592            // PostgreSQL WAL position
  }
}
```

**Downstream consumers:**
```
Kafka orders topic
  → Elasticsearch: update order document for search
  → Data Lakehouse: upsert into Delta Lake orders table
  → Analytics DB: sync to Redshift/BigQuery
  → Notification service: trigger "your order shipped" email
  → Inventory service: update reserved/available stock
```

CDC is how you keep multiple systems in sync in a microservices architecture without direct DB connections between services.

---

## 🗃️ OLAP vs OLTP & Columnar Storage

### OLTP (Online Transaction Processing)
- Many small, concurrent read/write transactions
- Row-oriented storage (all columns of a row together)
- Low latency per query (ms)
- Examples: PostgreSQL, MySQL, DynamoDB

### OLAP (Online Analytical Processing)
- Few large, complex queries reading millions of rows
- Column-oriented storage (all values of one column together)
- High throughput per query (may take seconds, but over petabytes)
- Examples: Redshift, BigQuery, Snowflake, ClickHouse, DuckDB

### Why Columnar Storage is Faster for Analytics

```
Row-oriented (PostgreSQL):
  Row 1: [order_id=1, user_id=100, product_id=50, amount=99.99, status="SHIPPED", ...]
  Row 2: [order_id=2, user_id=101, product_id=50, amount=149.99, status="PENDING", ...]

Query: SELECT SUM(amount) FROM orders WHERE product_id = 50
  → Must read every row (all columns) to get the amount value
  → For 1 billion rows: read 1 billion full rows

Column-oriented (BigQuery):
  amount column:   [99.99, 149.99, 299.99, ...]  (all amounts together)
  product_id column: [50, 50, 51, ...]            (all product_ids together)

Query: SELECT SUM(amount) FROM orders WHERE product_id = 50
  → Read only product_id column (find matching rows)
  → Read only amount column (for matching rows)
  → Skip all other columns entirely!
  → 10-100x less data read → 10-100x faster
```

**Additional columnar optimizations:**
- **Compression:** Same data type + sorted → better compression (run-length encoding, dictionary encoding)
- **Vectorized execution:** Process 1024 values at once using SIMD CPU instructions
- **Predicate pushdown:** Push filters to storage layer (skip entire files/row groups that don't match)
- **Min/max statistics:** Skip row groups where min > max_value_we're_filtering_for

**ClickHouse** is the fastest open-source columnar DB. Can scan billions of rows per second on a single server. Used by: Cloudflare, ByteDance, Uber, Yandex for real-time analytics.

---

## 📱 Real-Life Scenario: E-Commerce Analytics Platform

**The requirement:** Build an analytics platform that provides:
1. Real-time revenue dashboard (< 5 second delay)
2. Daily product performance reports (next-day)
3. Ad-hoc analysis by business analysts (SQL queries)
4. ML training data for recommendation engine

**Architecture:**

```
OLTP Database (PostgreSQL)
    │ Debezium CDC
    ▼
Apache Kafka (orders, users, products, events topics)
    │
    ├──► Flink Real-Time Job:
    │      - Computes revenue per minute per category
    │      - Detects fraudulent orders in real-time
    │      - Writes to: Redis (for dashboard), Kafka (fraud alerts)
    │
    ├──► Spark Batch Job (every 4 hours):
    │      - Reads from Kafka (last 4 hours)
    │      - Enriches with product/user data
    │      - Writes Parquet to Delta Lake (S3)
    │
    └──► Raw ingestion (Kafka Connect → S3):
           - All events in JSON, partitioned by day
           - Long-term retention
           
Delta Lake (S3):
    │ dbt transforms
    ▼
Data Warehouse (BigQuery):
    - Cleaned fact_orders, dim_products, dim_users tables
    - Business analysts query with SQL
    - Looker/Tableau dashboards

ML Platform:
    - Reads from Delta Lake (S3) directly with Spark
    - Trains collaborative filtering model weekly
    - Publishes recommendations to Redis
```

**Query examples:**
```sql
-- Daily revenue by category (BigQuery)
SELECT
    DATE(created_at) as date,
    category,
    SUM(amount) as revenue,
    COUNT(*) as orders,
    AVG(amount) as avg_order_value
FROM `myproject.analytics.fact_orders`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND status = 'COMPLETED'
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC

-- Ad-hoc: cohort retention
WITH cohorts AS (
    SELECT user_id, DATE_TRUNC(first_order_date, MONTH) as cohort_month
    FROM dim_users WHERE first_order_date IS NOT NULL
),
activities AS (
    SELECT user_id, DATE_TRUNC(created_at, MONTH) as activity_month
    FROM fact_orders WHERE status = 'COMPLETED'
    GROUP BY 1, 2
)
SELECT
    cohort_month,
    DATE_DIFF(activity_month, cohort_month, MONTH) as months_since_first_order,
    COUNT(DISTINCT a.user_id) / COUNT(DISTINCT c.user_id) as retention_rate
FROM cohorts c
JOIN activities a USING (user_id)
GROUP BY 1, 2
```

---

## 🔥 Big Tech Story: Uber's Real-Time Analytics

### The Problem (2015-2016)

Uber was growing explosively — millions of trips per day across hundreds of cities. Their data team needed to answer questions like:
- "What's the current surge pricing multiplier in Manhattan?"
- "How many drivers are available in San Francisco right now?"
- "What's the P99 ETA accuracy for Chicago in the last 5 minutes?"

These questions needed answers in **seconds**, on data that was **seconds old**. Traditional batch pipelines (hourly/daily) were useless for operational decisions.

### The Challenge

Uber's data volume was massive:
- Hundreds of millions of GPS pings per day
- Millions of ride events per day
- Data from hundreds of microservices

Traditional SQL databases couldn't handle this query load. BigQuery/Redshift had minutes of latency for the ETL pipeline. They needed something new.

### The Solution: Pinot + Kafka

Uber (along with LinkedIn, where Pinot was originally built) developed **Apache Pinot** — a real-time OLAP data store.

**How Pinot works:**
```
Kafka (ride_events, driver_locations, fare_calculations)
    │ real-time ingestion (< 1 second)
    ▼
Pinot Real-Time Segments (columnar, in-memory)
    │ merged with
Pinot Offline Segments (columnar, from HDFS/S3 batch)
    │
SQL queries → Pinot Broker → Pinot Servers → Return in < 100ms
```

**Pinot's key properties:**
- Ingests from Kafka with < 1 second latency
- Columnar storage (fast aggregations)
- Pre-built inverted indexes, star-tree indexes for fast GROUP BY
- Scales to hundreds of billions of rows
- P99 query latency < 100ms even on massive datasets

**The results:** Uber could now answer "What is surge pricing across all US cities right now?" in under 50ms, on data less than 30 seconds old.

### The Real-Time Analytics Ecosystem

Uber's architecture became the template for real-time analytics at many companies:

```
Kafka → Pinot → User-facing analytics (Uber Eats restaurant dashboards)
     → Flink → Fraud detection (real-time risk scoring)
     → Spark → ML model training (demand forecasting)
     → Delta Lake → Historical analysis (business intelligence)
```

### Other Systems in This Space

The real-time OLAP space has evolved rapidly:

| System | Latency | Throughput | Best For |
|---|---|---|---|
| **Apache Pinot** (LinkedIn/Uber) | < 100ms | Very high | User-facing real-time analytics |
| **Apache Druid** | < 1s | Very high | Time-series dashboards, monitoring |
| **ClickHouse** | < 1s | Extreme | Interactive analytics, observability |
| **Apache Doris** | < 1s | High | OLAP + some OLTP (good SQL coverage) |
| **Rockset** | < 100ms | High | Managed, operational analytics |

### The Lessons

1. **Different analytics use cases need different tools.** Historical BI (Redshift/BigQuery), operational analytics (Pinot/ClickHouse), real-time stream processing (Flink) — there's no single tool that does all well.

2. **Kafka is the data backbone.** Every analytics system at Uber ultimately reads from Kafka. This decouples data producers from consumers and enables multiple independent analytics systems.

3. **Real-time analytics changes products.** When Uber could show restaurant owners their live order volume and acceptance rate in the Uber Eats Partner app, it changed driver/restaurant behavior. The latency of analytics determines what products you can build.

4. **Invest in data infrastructure early.** Uber's data team was a strategic competitive advantage. Real-time surge pricing, demand forecasting, and fraud detection all depended on the data platform.

---

## ⚖️ Trade-offs & Best Practices

### Choosing a Processing Framework

| Requirement | Recommended |
|---|---|
| Batch ETL, ML training | Apache Spark |
| True event-time streaming, exactly-once | Apache Flink |
| Simple Kafka consumer logic | Kafka Streams |
| Real-time OLAP queries | ClickHouse or Apache Pinot |
| Managed, no-ops | BigQuery, Snowflake, AWS Athena |

### Best Practices

1. **Use Parquet (or ORC) for columnar storage on S3.** Parquet is 10-50x faster than CSV for analytical queries (via Spark/Athena) due to columnar format and compression.

2. **Partition your data lake by date + other common filters.** `s3://lake/events/year=2024/month=01/day=15/` allows query engines to skip irrelevant partitions entirely.

3. **Use Delta Lake or Iceberg for ACID operations.** Raw Parquet in S3 has no ACID guarantees — concurrent writers can corrupt data. Table formats fix this.

4. **Implement CDC for real-time synchronization.** Debezium + Kafka is the standard pattern for replicating OLTP changes to analytics systems.

5. **Separate hot and cold data paths.** Use Flink/Pinot for the last 24 hours. Use BigQuery/Redshift for historical queries. Don't force one system to do both.

6. **Use dbt for SQL transformations.** dbt brings software engineering practices (version control, testing, documentation, modularity) to SQL.

7. **Monitor data pipeline latency.** Alert when Kafka consumer lag grows unexpectedly (indicates processing is falling behind).

8. **Design for late-arriving data.** With event-time processing, always define your acceptable lateness (watermarks) and handle late events explicitly.

9. **Avoid small files on S3.** Many small files (< 1MB) are slow in Spark (one task per file). Compact to 100-500MB files. Delta Lake's `OPTIMIZE` command does this automatically.

10. **Data quality gates.** Add `dbt test` assertions on your data models: no null order_ids, amounts > 0, user_ids exist in dim_users. Fail pipelines early if data quality drops.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **Batch vs Stream** | Batch = bounded, high throughput; Stream = real-time, complex. Choose based on required latency. |
| **Spark** | Industry standard for batch; DAG + in-memory + Catalyst optimizer = fast |
| **Flink** | True event-time streaming; RocksDB state; exactly-once via checkpoints |
| **Lambda arch** | Batch accuracy + stream speed; but two codebases = complexity |
| **Kappa arch** | Stream-only; replay via Kafka. Simpler but requires long Kafka retention |
| **Data Lakehouse** | S3 cost + warehouse features via Delta Lake/Iceberg (ACID, time travel) |
| **CDC** | Real-time DB sync via WAL (Debezium + Kafka); eliminates batch ETL |
| **Columnar storage** | 10-100x faster for analytical queries vs row-oriented |
| **ELT over ETL** | Load raw first; transform with SQL (dbt) inside the warehouse |
| **Uber's lesson** | Real-time analytics requires specialized real-time OLAP (Pinot/Druid/ClickHouse) |

---

*Next: [15_High_Availability_and_Disaster_Recovery.md](15_High_Availability_and_Disaster_Recovery.md) — RPO/RTO, multi-region active-active, backup strategies, and the AWS us-east-1 outage story.*
