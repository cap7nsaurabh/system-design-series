# Elasticsearch: The Distributed Search & Analytics Engine

## Overview
Elasticsearch is a distributed, RESTful, JSON-based search and analytics engine built on Apache Lucene. Developed by Elastic, it powers full-text search, structured log aggregation (the ELK stack: Elasticsearch, Logstash, Kibana), application observability, and real-time analytical business intelligence across petabytes of semi-structured data.

---

## Architecture & Search Engine Internals

```
+-------------------------------------------------------------------------+
|                    Elasticsearch Cluster Architecture                   |
+-------------------------------------------------------------------------+
 Master Nodes (Cluster state, shard allocation, consensus via Raft)
                   │
                   ▼
 Data Nodes (Hold Shards: Inverted Indexes, Doc Values, Segment Merging)
   ┌───────────────────────────────────┬──────────────────────────────────┐
   │ Primary Shard 0 (Lucene Index)   │ Replica Shard 1 (Lucene Index)   │
   │ - Segment 1, Segment 2, Segment 3 │ - Translog (Crash durability)    │
   │ - Inverted Index + FST Dictionary │ - Doc Values (Columnar storage)  │
   └───────────────────────────────────┴──────────────────────────────────┘
                   │
                   ▼
 Coordinating Nodes (Client request routing, scatter-gather query aggregation)
```

### 1. The Inverted Index & Lucene Segments
- **Inverted Index:** Maps every unique word/token to a sorted list of Document IDs where it appears (the "Posting List").
- **Finite State Transducers (FST):** Inverted index term dictionaries are loaded into memory as memory-compact FSTs, enabling sub-millisecond prefix, fuzzy, and regex matching.
- **Immutable Segments:** Lucene writes new documents into immutable on-disk index files called **Segments**. Writes are sequential append-only.
- **Segment Merging:** A background daemon merges smaller segments into larger ones, physically purging deleted records (which are marked with a bitset deletion vector until merged).
- **Near-Real-Time (NRT) Refresh:** New documents are visible for search only after a **Refresh** (default 1 second), which writes memory buffers to OS filesystem cache. Durability is maintained separately via the **Translog** (flushed on fsync).

### 2. Doc Values vs Fielddata (Sorting & Aggregations)
- **Doc Values:** On-disk, columnar data structures generated at index time. Used for sorting, aggregations, and script lookups. Very cache-friendly.
- **Fielddata:** In-memory un-inverted index built dynamically for analyzed `text` fields. Highly memory-intensive and discouraged for production analytics.

---

## Real World Stories & Incidents

### 1. LinkedIn's Multi-Billion Job & Member Search
LinkedIn powers its global talent and job search using Elasticsearch:
- Searching for candidates requires combining full-text relevance, exact boolean filters (skills, seniority), and personalized geographic distance ranking.
- LinkedIn routes queries across multi-cluster setups using custom scoring plugins, executing complex `function_score` queries that balance recruiter relevance and job freshness in under $20\text{ms}$.

### 2. The Dreaded Elasticsearch Cluster "Split-Brain" (Pre-7.x Outage)
In early Elasticsearch versions, clusters relied on the `Zen Discovery` consensus protocol:
- If a network partition occurred between datacenters, both sides of the cluster believed the other side had died.
- Both sides elected a new master node, and both began accepting indexing writes simultaneously.
- When the network healed, documents indexed on both sides had conflicting internal version numbers, corrupting cluster state and requiring days of manual index restoration from backups.
- *Modern Fix:* Since Elasticsearch 7.0, Zen Discovery was replaced with a formal Raft-based consensus algorithm that mathematically prevents split-brain elections.

---

## Complete Elasticsearch Query DSL & Operations Reference

### 1. Index Management & Dynamic Mappings

```json
// PUT /ecommerce_products
// Create Index with explicit settings and analyzers
PUT /ecommerce_products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "refresh_interval": "1s",
    "analysis": {
      "analyzer": {
        "custom_product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_id": { "type": "keyword" },
      "title": { 
        "type": "text", 
        "analyzer": "custom_product_analyzer",
        "fields": {
          "raw": { "type": "keyword" } // Multi-field: text for search, keyword for exact aggregation
        }
      },
      "description": { "type": "text" },
      "category": { "type": "keyword" },
      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "in_stock": { "type": "boolean" },
      "location": { "type": "geo_point" },
      "created_at": { "type": "date" },
      "tags": { "type": "keyword" },
      "reviews": {
        "type": "nested", // Nested type prevents object flattening
        "properties": {
          "reviewer": { "type": "keyword" },
          "rating": { "type": "integer" },
          "comment": { "type": "text" }
        }
      }
    }
  }
}
```

---

### 2. Document CRUD & Bulk Ingestion

```http
# 1. Index a single document
POST /ecommerce_products/_doc/prod_101
{
  "product_id": "prod_101",
  "title": "Sony WH-1000XM5 Wireless Noise Cancelling Headphones",
  "category": "Electronics",
  "price": 399.99,
  "in_stock": true,
  "location": { "lat": 37.7749, "lon": -122.4194 },
  "created_at": "2024-06-01T12:00:00Z",
  "tags": ["audio", "wireless", "premium"],
  "reviews": [
    { "reviewer": "Alice", "rating": 5, "comment": "Best noise cancelling ever!" },
    { "reviewer": "Bob", "rating": 4, "comment": "Great sound, slightly bulky case." }
  ]
}

# 2. Get document by ID
GET /ecommerce_products/_doc/prod_101

# 3. Partial Update with Script
POST /ecommerce_products/_update/prod_101
{
  "script": {
    "source": "ctx._source.price -= params.discount",
    "params": { "discount": 20.00 }
  }
}

# 4. High-Throughput Bulk API (NDJSON format)
POST /_bulk
{ "index": { "_index": "ecommerce_products", "_id": "prod_102" } }
{ "product_id": "prod_102", "title": "Apple AirPods Max", "category": "Electronics", "price": 549.00, "in_stock": true }
{ "delete": { "_index": "ecommerce_products", "_id": "prod_099" } }
```

---

### 3. Full-Text & Term-Level Search Queries

```json
// Match Query (Full-Text Analyzed Search)
GET /ecommerce_products/_search
{
  "query": {
    "match": {
      "title": {
        "query": "wireless noise cancelling headphones",
        "operator": "and",
        "fuzziness": "AUTO" // Auto-corrects typos (e.g., 'nois' -> 'noise')
      }
    }
  }
}

// Multi-Match Query (Search across multiple text fields with boosting)
GET /ecommerce_products/_search
{
  "query": {
    "multi_match": {
      "query": "bluetooth sound",
      "fields": ["title^3", "description"], // title is boosted 3x
      "type": "best_fields"
    }
  }
}

// Term & Range Query (Exact, Non-analyzed filters)
GET /ecommerce_products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100.00,
        "lte": 500.00
      }
    }
  }
}
```

---

### 4. Compound Queries: The `bool` Query Engine

The foundation of production search: combining relevance scoring with cacheable boolean filters.

```json
GET /ecommerce_products/_search
{
  "query": {
    "bool": {
      // MUST: Contributes to relevance score (_score)
      "must": [
        { "match": { "title": "wireless headphones" } }
      ],
      // SHOULD: Increases _score if present, but not mandatory
      "should": [
        { "match_phrase": { "title": "noise cancelling" } }
      ],
      // FILTER: Exact boolean check, DOES NOT score, RESULTS ARE CACHED IN MEMORY!
      "filter": [
        { "term": { "category": "Electronics" } },
        { "term": { "in_stock": true } },
        { "range": { "price": { "lte": 450.00 } } }
      ],
      // MUST_NOT: Inverts match, does not affect score
      "must_not": [
        { "term": { "tags": "refurbished" } }
      ]
    }
  },
  "sort": [
    { "_score": { "order": "desc" } },
    { "price": { "order": "asc" } }
  ],
  "from": 0,
  "size": 20
}
```

---

### 5. Nested Queries & Geospatial Queries

```json
// Nested Query: Querying array of nested objects correctly
GET /ecommerce_products/_search
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "bool": {
          "must": [
            { "match": { "reviews.comment": "noise cancelling" } },
            { "range": { "reviews.rating": { "gte": 4 } } }
          ]
        }
      }
    }
  }
}

// Geospatial Radius Search
GET /ecommerce_products/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "25km",
          "location": {
            "lat": 37.7749,
            "lon": -122.4194
          }
        }
      }
    }
  }
}
```

---

### 6. Aggregations (Metric, Bucket & Pipeline)

```json
// Multi-Faceted Aggregation: Category breakdown + Price Stats + Sub-aggregations
GET /ecommerce_products/_search
{
  "size": 0, // Omit raw hits; return only aggregate computations
  "aggs": {
    // 1. Bucket Aggregation: Group by category
    "categories": {
      "terms": {
        "field": "category",
        "size": 10
      },
      "aggs": {
        // Sub-metric: Average price per category
        "avg_price": { "avg": { "field": "price" } },
        // Sub-metric: Highest priced item
        "max_price": { "max": { "field": "price" } }
      }
    },
    // 2. Date Histogram: Sales count over time
    "sales_over_time": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      }
    },
    // 3. Cardinality: Unique tags count (uses HyperLogLog++)
    "unique_tags_count": {
      "cardinality": {
        "field": "tags"
      }
    }
  }
}
```

---

### 7. Index Lifecycle Management (ILM)

Production clusters transition indices through lifecycle tiers:

```json
PUT /_ilm/policy/logs_retention_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "30d"
          }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "allocate": { "number_of_replicas": 1 }
        }
      },
      "cold": {
        "min_age": "90d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```
