# Time-Series Databases: TimescaleDB & InfluxDB

## Overview
Time-Series data represents measurements, metrics, or events tracked, monitored, and recorded chronologically over time. Characterized by high write velocity, immutable append-only workloads, time-based range queries, and rapid decay of data value over time (recent data is hot; older data is summarized or purged), time-series workloads require specialized database architectures fundamentally different from traditional OLTP databases.

---

## Architectural Deep Dive: Hypertables vs TSM Engines

### 1. TimescaleDB: Relational Scalability on PostgreSQL
TimescaleDB is engineered as a PostgreSQL extension that transforms Postgres into a distributed time-series engine while preserving full SQL support and ACID compliance.

```
+-------------------------------------------------------------------------+
|                  TimescaleDB Hypertable Architecture                    |
+-------------------------------------------------------------------------+
 Hypertable: "sensor_telemetry" (Logical Postgres Table abstraction)
                   │
                   ▼ (Automatic 2D partitioning by Time + Space)
   ┌───────────────────────┬───────────────────────┬──────────────────────┐
   │ Chunk 1               │ Chunk 2               │ Chunk 3              │
   │ [2024-06-01 - 06-07]  │ [2024-06-08 - 06-14]  │ [2024-06-15 - 06-21] │
   │ (Compressed Columnar) │ (Compressed Columnar) │ (Row-Oriented Heap)  │
   └───────────────────────┴───────────────────────┴──────────────────────┘
```

- **Hypertables:** A single virtual table interface exposed to users that transparently routes writes into time-partitioned physical tables called **Chunks**.
- **Chunk Exclusion:** When a query filters by timestamp, the query planner prunes untouched chunks before scanning, reducing I/O to only relevant time windows.
- **Hybrid Row/Columnar Storage:** Recent chunks are written as fast row-based tuples. Once a chunk ages past a threshold (e.g., 7 days), TimescaleDB transforms it into an **on-disk columnar format**, achieving **$90\%+$ compression ratios**.
- **Continuous Aggregates:** Real-time materialized views that compute rollups incrementally as new rows arrive without recalculating historical buckets.

### 2. InfluxDB: Time-Structured Merge-Tree (TSM)
InfluxDB is a purpose-built, standalone time-series engine:
- **Data Model:** `Measurement` (Table), `Tags` (Indexed string metadata), `Fields` (Unindexed values/metrics), `Timestamp`.
- **TSM Engine:** Inspired by LSM trees, InfluxDB utilizes a Time-Structured Merge-Tree where data is grouped by series key and written sequentially to immutable columnar files.
- **The Series Cardinality Wall:** InfluxDB indexes all tag combinations in an in-memory inverted index (TSI). If high-cardinality values (like UUIDs or User IDs) are stored as tags, the index explodes in RAM, crashing the server.

---

## Real World Stories & Incidents

### 1. Tesla's Fleet Vehicle Telemetry
Tesla collects billions of data points daily from millions of connected electric vehicles (battery temperature, motor RPM, autopilot camera health, tire pressure):
- Using standard relational databases resulted in I/O degradation and storage exhaustion within weeks.
- Fleet engineers leverage time-series architectures with downsampling policies: raw sensor data is ingested at 10 Hz, aggregated into 1-minute averages after 24 hours, downsampled to 1-hour rollups after 30 days, and compressed into long-term cold storage.

### 2. Robinhood's Real-Time Stock Market Candlesticks
During the GameStop trading frenzy, Robinhood processed millions of stock price updates per second:
- Displaying real-time 1-minute, 5-minute, and 1-day candlestick charts requires instant aggregations (`time_bucket()`, `first()`, `last()`, `max()`, `min()`).
- By utilizing continuous aggregates, Robinhood serves pre-aggregated OHLC (Open, High, Low, Close) chart queries in sub-5ms directly from memory.

---

## Comprehensive TimescaleDB Operations & SQL Reference

### 1. Hypertables Creation & Indexing

```sql
-- Ensure extension is installed
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- 1. Standard PostgreSQL Table definition
CREATE TABLE device_metrics (
    recorded_at TIMESTAMPTZ NOT NULL,
    device_id UUID NOT NULL,
    firmware_version VARCHAR(16),
    cpu_usage FLOAT,
    temperature_celsius FLOAT,
    network_latency_ms INT
);

-- 2. Convert standard table to TimescaleDB Hypertable
-- Chunk time interval: Creates a new physical chunk every 7 days
SELECT create_hypertable(
    'device_metrics', 
    'recorded_at', 
    chunk_time_interval => INTERVAL '7 days'
);

-- 3. Composite Index on Device and Time for lightning-fast queries
CREATE INDEX idx_device_time ON device_metrics (device_id, recorded_at DESC);
```

---

### 2. Time-Bucket Aggregations & Analytical Functions

```sql
-- Downsample raw data into 5-minute buckets with gap filling
SELECT 
    time_bucket('5 minutes', recorded_at) AS five_min_bucket,
    device_id,
    AVG(cpu_usage) AS avg_cpu,
    MAX(temperature_celsius) AS peak_temp,
    -- First and Last value functions (OHLC analytics)
    first(temperature_celsius, recorded_at) AS opening_temp,
    last(temperature_celsius, recorded_at) AS closing_temp
FROM device_metrics
WHERE recorded_at >= NOW() - INTERVAL '24 hours'
GROUP BY five_min_bucket, device_id
ORDER BY five_min_bucket DESC;

-- LOCF (Last Observation Carried Forward) for handling missing sensor intervals
SELECT 
    time_bucket_gapfill('1 minute', recorded_at) AS minute_slot,
    device_id,
    locf(AVG(temperature_celsius)) AS imputed_temp
FROM device_metrics
WHERE recorded_at >= '2024-06-01 00:00:00Z' AND recorded_at < '2024-06-01 06:00:00Z'
GROUP BY minute_slot, device_id;
```

---

### 3. Continuous Aggregates (Real-Time Materialized Views)

```sql
-- Create automatically updating hourly summary
CREATE MATERIALIZED VIEW hourly_device_summary
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', recorded_at) AS bucket_hour,
    device_id,
    COUNT(*) AS total_samples,
    AVG(cpu_usage) AS avg_cpu,
    AVG(temperature_celsius) AS avg_temp,
    MAX(temperature_celsius) AS max_temp
FROM device_metrics
GROUP BY bucket_hour, device_id
WITH NO DATA;

-- Define continuous aggregate refresh policy (runs every 30 minutes in background)
SELECT add_continuous_aggregate_policy('hourly_device_summary',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '30 minutes'
);
```

---

### 4. Native Columnar Compression & Data Retention

```sql
-- Enable Columnar Compression on the Hypertable
ALTER TABLE device_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);

-- Automatically compress chunks older than 7 days
SELECT add_compression_policy('device_metrics', INTERVAL '7 days');

-- Automatic Data Retention: Drop uncompressed or compressed chunks older than 90 days
SELECT add_retention_policy('device_metrics', INTERVAL '90 days');
```

---

## Comprehensive InfluxDB (InfluxQL & Flux) Reference

### 1. InfluxQL Syntax

```sql
-- Insert Line Protocol format:
-- measurement,tag_set field_set timestamp
-- cpu_usage,host=server_01,region=us-west value=78.5 1717240000000000000

-- InfluxQL Query: Mean CPU grouped by 10-minute intervals
SELECT MEAN("value") AS "mean_cpu" 
FROM "cpu_usage" 
WHERE "region" = 'us-west' AND time >= now() - 12h 
GROUP BY time(10m), "host" 
FILL(previous);

-- Create Retention Policy
CREATE RETENTION POLICY "thirty_days" ON "telemetry_db" 
DURATION 30d REPLICATION 1 DEFAULT;
```

### 2. InfluxDB Flux Query Language (Functional Pipeline Syntax)

```javascript
// Flux Script: Filter, Window, Aggregate, and Pivot
from(bucket: "telemetry_bucket")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "device_metrics" and r._field == "temperature")
  |> filter(fn: (r) => r.region == "eu-central")
  |> aggregateWindow(every: 15m, fn: mean, createEmpty: false)
  |> yield(name: "mean_15m_temp")
```

---

## Architectural Comparison Matrix

| Feature | TimescaleDB | InfluxDB | Apache Cassandra | Prometheus |
|---|---|---|---|---|
| **Underlying Engine** | PostgreSQL | TSM / Apache Arrow | LSM-Tree Distributed | Custom append-only disk TSDB |
| **Query Language** | Full ANSI SQL | Flux / InfluxQL | CQL (Cassandra QL) | PromQL |
| **High Cardinality Support** | Excellent (B-Tree + Relational) | Poor (Explodes RAM) | High (Partition Key bounded) | Medium |
| **JOIN Capability** | Full SQL Joins | Very Limited | None | None |
| **Compression** | Up to $90\%+$ (Columnar) | Excellent (Gorilla/Snappy) | Good (ZSTD/Snappy) | Excellent (Double-delta) |
| **Best For** | IoT telemetry, FinTech, SQL teams | DevOps metrics, simple IoT | Petabyte-scale raw append | K8s / Server monitoring |
