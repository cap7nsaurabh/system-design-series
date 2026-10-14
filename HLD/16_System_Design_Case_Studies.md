# 🔭 System Design Case Studies

> *"The best way to learn system design is by doing it — and the second-best way is studying how the best engineers did it."*

---

## 📋 Table of Contents
1. [How to Approach a System Design Interview](#approach)
2. [Case Study 1: URL Shortener (like bit.ly)](#url-shortener)
3. [Case Study 2: Twitter / X](#twitter)
4. [Case Study 3: YouTube](#youtube)
5. [Case Study 4: Uber / Ride-Sharing](#uber)
6. [Case Study 5: WhatsApp / Chat Messaging](#whatsapp)
7. [Trade-offs Reference Card](#tradeoffs-reference)
8. [Key Takeaways](#key-takeaways)

---

## 🗺️ How to Approach a System Design Interview

Before diving into case studies, here is the structured framework for tackling any system design question.

### The RESHADED Framework

**R** — Requirements clarification (5 minutes)
**E** — Estimation (3 minutes)
**S** — Storage (data model)
**H** — High-level design
**A** — Algorithms & data structures
**D** — Detailed design (deep-dive on 1-2 components)
**E** — Evaluation (trade-offs, bottlenecks)
**D** — Done — summarize

### Step 1: Requirements Clarification

Never assume. Always ask:
- **Functional requirements:** What does the system do? (core features, out-of-scope features)
- **Non-functional requirements:** How fast? How available? How consistent? What scale?

Example questions for URL shortener:
```
- How many URLs shortened per day? (write QPS)
- How many redirects per day? (read QPS)
- How long are shortened URLs valid?
- Should links be customizable? (e.g., bit.ly/my-product)
- Should we track click analytics? (bonus feature)
- What's the expected latency for redirect? (< 100ms?)
- Is availability or consistency more important?
```

### Step 2: Estimation (Back of Envelope)

Calculate expected scale. This drives your architecture choices.

```
Standard assumptions for interviews:
  1 character = 1 byte
  1KB = 10^3 bytes
  1MB = 10^6 bytes
  1GB = 10^9 bytes
  1TB = 10^12 bytes
  
  Daily active users (DAU) → QPS = DAU × avg_requests / 86400 seconds
  Peak QPS ≈ 3× average QPS
```

### Step 3: High-Level Design

Draw the big boxes first:
- Client
- Load Balancer / API Gateway
- App servers
- Database(s)
- Cache
- External services

Don't optimize prematurely. Get the happy path working first.

### Step 4: Deep Dive

The interviewer will ask you to go deeper on one or two specific components. This is where you show mastery.

---

## 🔗 Case Study 1: URL Shortener (like bit.ly)

### Requirements

**Functional:**
- User submits long URL → gets short URL (e.g., `sho.rt/abc123`)
- Visiting short URL → redirects to original long URL
- [Optional] Custom aliases (e.g., `sho.rt/my-product`)
- [Optional] Click analytics (how many clicks, from where)

**Non-functional:**
- High availability (redirects must work; no cached URL = broken link)
- Low latency redirects (< 100ms including network)
- URLs should be unique (no two longs → same short)
- System is read-heavy (10:1 read:write ratio or higher)
- Eventual consistency acceptable (no financial transactions)

### Estimation

```
Assumptions:
  100M new URLs shortened per day
  10 read (redirect) per URL created → 1B redirects per day

Write QPS:
  100M / 86400 ≈ 1,157 writes/second ≈ ~1K write QPS
  Peak: ~3K write QPS

Read QPS:
  1B / 86400 ≈ 11,574 reads/second ≈ ~12K read QPS
  Peak: ~36K read QPS

Storage (5-year retention):
  100M URLs/day × 365 days × 5 years = 182.5B URLs
  Each record: ~500 bytes (short_key + long_url + created_at + user_id)
  Total: 182.5B × 500B ≈ 91.25 TB
  
Read:Write ratio: ~10:1 (heavily read-dominant)
```

### Data Model

```sql
-- URL mappings table
CREATE TABLE url_mappings (
    short_key   VARCHAR(8) PRIMARY KEY,  -- e.g., "abc123XY"
    long_url    TEXT NOT NULL,           -- original URL (up to 2048 chars)
    user_id     BIGINT,                  -- creator (nullable for anonymous)
    created_at  TIMESTAMP NOT NULL,
    expires_at  TIMESTAMP,               -- optional expiry
    click_count BIGINT DEFAULT 0         -- approximate (updated async)
);
CREATE INDEX idx_url_mappings_user ON url_mappings(user_id, created_at DESC);

-- Click events (for analytics, separate table)
CREATE TABLE click_events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    short_key   VARCHAR(8) NOT NULL,
    clicked_at  TIMESTAMP NOT NULL,
    ip_address  INET,
    user_agent  TEXT,
    country     VARCHAR(2)
);
PARTITION BY RANGE (clicked_at);  -- Partition by month for efficient querying
```

### High-Level Architecture

```
Client → CDN (cached redirects!) → Load Balancer → App Server → Redis (hot keys)
                                                             → PostgreSQL (all URLs)

Write path: App Server → Generate short key → PostgreSQL → Return short URL
Read path:  App Server → Redis (MISS) → PostgreSQL → Redis (SET) → 301/302 redirect
```

### Short Key Generation

**Option 1: MD5/SHA-256 hash of long URL**
```python
import hashlib, base64

def generate_short_key(long_url: str) -> str:
    hash_bytes = hashlib.md5(long_url.encode()).digest()
    # Base62 encode first 6 bytes
    return base64url_encode(hash_bytes[:6])

# Base62 alphabet: [A-Za-z0-9] (62 characters)
# 6 chars of base62 = 62^6 = 56.8 billion combinations
# 8 chars of base62 = 62^8 = 218 trillion combinations
```

**Problem:** Hash collisions. Two different URLs could produce the same short key (very unlikely but possible). Need collision detection: check if key exists → if yes, try next chars of hash.

**Problem 2:** Same URL always gets same short key. Multiple users can't create different short keys for the same long URL.

**Option 2: Random ID Generation (Recommended)**
```python
import secrets, string

BASE62_ALPHABET = string.ascii_letters + string.digits  # 62 chars

def generate_short_key(length: int = 7) -> str:
    return ''.join(secrets.choice(BASE62_ALPHABET) for _ in range(length))

# 7 chars of base62 = 62^7 = 3.5 trillion combinations
# At 100M URLs/day: takes 35,000 days (96 years) to exhaust!
# Collision probability is negligible

# On insert, check for collision:
for attempt in range(3):
    key = generate_short_key()
    try:
        db.execute("INSERT INTO url_mappings (short_key, ...) VALUES (?,...)", [key, ...])
        return key
    except UniqueViolationError:
        continue  # Retry with different key
raise Exception("Unable to generate unique key")
```

**Option 3: Counter-Based with Base62 Encoding (No collision risk)**
```python
# Global atomic counter (can use Redis INCR or DB sequence)
counter_id = redis.incr("url_counter")  # Atomically returns: 1, 2, 3, ...

def encode_base62(num: int) -> str:
    ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    result = []
    while num:
        result.append(ALPHABET[num % 62])
        num //= 62
    return ''.join(reversed(result)).zfill(7)

short_key = encode_base62(counter_id)  # 1 → "0000001", 1000000 → "4c92"
```

**Problem:** Predictable (IDs are sequential). Competitor can enumerate all your URLs. **Solution:** XOR or permute the counter with a secret before encoding.

### Redirect: 301 vs 302

```
301 Moved Permanently:
  Browser caches the redirect permanently
  Subsequent visits: browser goes directly to long URL (no server contact!)
  ✅ Lower server load, faster for repeat visitors
  ❌ Click analytics become inaccurate (browser bypasses our server)
  ❌ If you change the mapping, browser still uses cached old destination

302 Found (Temporary Redirect):
  Browser does NOT cache (always goes through our server)
  ✅ Accurate click analytics (every redirect logged)
  ✅ Can change destination anytime
  ❌ Higher server load (every visit hits our servers)
```

**Recommendation:** 302 for analytics use cases. 301 if you want maximum performance and don't care about analytics.

### Caching Strategy

```
Hot URLs: Top 20% of URLs receive 80% of traffic (Zipf distribution)
Cache in Redis: short_key → long_url

Cache hit rate: ~80-90% for popular URLs
Redis cluster: 1TB RAM → stores all recent URLs

Expire from cache: set TTL to URL expiry date
CDN caching: Cache 302 responses at CDN edge with short TTL (1 min)
  → Next 10,000 requests to same short URL served from CDN edge!
  
Request path:
  CDN MISS → App Server → Redis HIT → Return 302 to CDN
  CDN stores 302 for 60 seconds → next 10K requests served from CDN
```

### Unique Constraint on Long URL (Optional)

Should two identical long URLs get the same or different short URLs?

- **Same:** (idempotent) Save storage, user gets same link if they shorten the same URL twice. But: privacy concern (user B can learn user A's short link by shortening same URL).
- **Different:** (new per request) More storage but private.

For public link shorteners (bit.ly style): allow both (check if user already shortened this URL → return existing).

For analytics-critical use cases: always create new (so each link's click count is independent).

### Analytics Deep-Dive (Bonus)

```
Click events are high-volume (1B/day). Two approaches:

Approach 1: Write to Kafka (async), Flink consumes for aggregation
  Click → App publishes to Kafka topic "url_clicks"
  Flink job: COUNT clicks per short_key per hour → write to ClickHouse
  Dashboard: query ClickHouse for click trends

Approach 2: Write directly to ClickHouse (columnar DB for analytics)
  Click → async write to ClickHouse (buffer locally, batch writes)
  ClickHouse handles 1M+ inserts/second natively
  Query: SELECT count(), toHour(clicked_at) FROM clicks WHERE short_key='abc' GROUP BY 2
```

### Scale Considerations

```
DB Bottleneck: 100M writes/day → ~1,200 writes/sec → PostgreSQL handles this fine
  Scale if needed: Shard by short_key prefix (16 shards → each handles 75 writes/sec)

Read bottleneck: 12K reads/sec → most served from Redis → PostgreSQL sees < 2K/sec

Redis sizing: 10M active URLs × 200 bytes each = 2GB (trivial for Redis cluster)

Multi-region:
  URL creation: single region (primary PostgreSQL)
  URL resolution: every region (each has read replica, CDN in front)
  → Global latency < 50ms for redirects
```

---

## 🐦 Case Study 2: Twitter / X

### Requirements

**Functional:**
- Users can post tweets (280 characters, optional media)
- Users follow other users
- User sees a timeline: tweets from people they follow
- Trends, search (bonus)
- Notifications (bonus)

**Non-functional:**
- 100M DAU
- Some users have 100M+ followers (celebrities)
- Read-heavy (100:1 read:write ratio for timelines)
- Eventually consistent timelines (okay if tweet appears after a few seconds)
- High availability for reads, consistency less critical

### Estimation

```
100M DAU, avg 5 tweets/user/month → 16M tweets/day
  16M / 86400 ≈ 185 tweets/second write
  Peak: ~1000 tweets/second

Timeline reads: 100M users × 20 timeline views/day = 2B reads/day
  2B / 86400 ≈ 23,148 reads/second
  Read:Write ≈ 125:1 (extremely read-heavy)

Storage:
  16M tweets/day × 200 bytes (text) = 3.2GB/day for text
  Media: some % of tweets have images (~2MB each)
  5 years: 16M × 365 × 5 × 200B ≈ 5.8TB (text only; media stored in object storage)

Follower problem:
  Average follows per user: 200
  Celebrity (e.g., Obama): 130M followers
  → A single Obama tweet must reach 130M timelines!
```

### The Core Problem: Timeline Fan-Out

**Approach 1: Pull (Fan-out on read)**
```
When user A opens Twitter:
  SELECT tweet_id, content FROM tweets
  WHERE user_id IN (SELECT followee_id FROM follows WHERE follower_id = user_A)
  ORDER BY created_at DESC LIMIT 20;

Problem: user A follows 500 people → scan 500 users' tweets → join → sort
  → Very slow at scale (with billions of tweets)
  → Must be computed fresh every time
  → p99 latency: seconds
```

**Approach 2: Push (Fan-out on write)**
```
When Obama tweets:
  Find all 130M followers
  Insert tweet_id into each follower's timeline cache
  
When user opens Twitter:
  Read their pre-computed timeline from cache
  → O(1) read! Fast.

Problem: Fan-out for Obama = 130M Redis writes
  At 185 tweets/second average, but 1 celebrity tweet = 130M writes
  → Spike load on Redis cluster
  → Also: 130M × 8 bytes (tweet_id) = 1GB of Redis writes per Obama tweet
```

**Approach 3: Hybrid (Twitter's Actual Solution)**

Twitter uses a combination:

```
Regular users (< ~10K followers): Push (fan-out on write)
  Tweet → worker fans out tweet_id to all followers' timelines in Redis
  
Celebrities (> ~10K followers): Pull (fan-out on read)
  Celebrity tweet is NOT pushed to all followers
  When follower opens timeline: Redis cache + SQL query for followed celebrities' recent tweets
  Merge and sort

Result: Most users' timelines are pre-computed (fast reads)
        Celebrity tweets fetched on read (avoids massive write spikes)
        Hybrid gives best of both worlds
```

### Architecture

```
Write Path (Posting a tweet):
  Client → API Gateway → Tweet Service
    1. Write tweet to PostgreSQL (source of truth)
    2. Upload media to S3 (if any)
    3. Publish TweetCreated event to Kafka

  Fan-out Workers (consume Kafka):
    For regular user: find followers → insert tweet_id into their Redis timeline list
    For celebrity: skip fan-out (timeline service queries on read)

Read Path (Loading timeline):
  Client → API Gateway → Timeline Service
    1. Read pre-computed timeline from Redis (ZSET ordered by timestamp): O(1)
    2. For each followed celebrity: fetch latest N tweets from Tweet Service
    3. Merge and sort
    4. Fetch tweet content for tweet_ids (from Redis cache or DB)
    5. Return to client
```

### Data Models

```sql
-- Core tables
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    display_name VARCHAR(100),
    bio TEXT,
    followers_count INT DEFAULT 0,
    following_count INT DEFAULT 0
);

CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,  -- Snowflake ID (time-sortable)
    user_id BIGINT REFERENCES users(user_id),
    content VARCHAR(280),
    media_urls TEXT[],
    reply_to_tweet_id BIGINT,
    retweet_count INT DEFAULT 0,
    like_count INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL
);
CREATE INDEX idx_tweets_user_created ON tweets(user_id, created_at DESC);

CREATE TABLE follows (
    follower_id BIGINT REFERENCES users(user_id),
    followee_id BIGINT REFERENCES users(user_id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);
-- For "get all followers of X": CREATE INDEX idx_follows_followee ON follows(followee_id);
```

**Redis Timeline Structure:**
```
Key: timeline:{user_id}
Type: ZSET (sorted set, sorted by tweet timestamp)
Members: tweet_ids
Score: created_at timestamp (Unix seconds)

ZADD timeline:user_123 1705312000 tweet_456
ZRANGE timeline:user_123 0 19 REV WITHSCORES  # Get latest 20 tweets
```

### Tweet ID Generation: Snowflake

Twitter invented the **Snowflake** ID format — a 64-bit globally unique, time-sortable ID.

```
64 bits total:
  [1 bit: sign (always 0)]
  [41 bits: milliseconds since epoch (custom epoch)]
  [10 bits: machine/datacenter ID]
  [12 bits: sequence number (per machine per millisecond)]

= 41-bit timestamp → time-sorted IDs!
= 10-bit machine ID → 1024 machines can generate IDs independently
= 12-bit sequence → 4096 IDs per millisecond per machine

Total: 4096 × 1024 machines × 1000/sec = 4.2 billion IDs/second

Most importantly: IDs are SORTABLE BY TIME (no ORDER BY created_at needed)
  Higher ID = newer tweet
```

### The Celebrity Problem: Lists and Search

**Twitter Lists:** Twitter allows users to create lists (curated set of accounts). Lists use the pull approach — when viewing a list timeline, fetch recent tweets from all accounts in the list in real-time.

**Search:** Elasticsearch indexing every tweet. Tweet content, hashtags, and user mentions indexed. Full-text search available in ~10-15 seconds of posting (near-real-time indexing via Kafka → Elasticsearch).

### Scale Points

- **Follower graph:** Neo4j or custom graph DB for complex graph queries. For simple "get followers of X" → PostgreSQL with proper indexes is fine up to 100M records.
- **Media:** S3 + CloudFront CDN. Images served directly from CDN, never from app servers.
- **Rate limiting:** 300 tweets per 3 hours per account (to prevent spam). Redis-based token bucket per user_id.
- **Trending topics:** Count hashtag occurrences in a sliding window (5 minutes, by region) using Kafka Streams or Redis sorted sets.

---

## 📺 Case Study 3: YouTube

### Requirements

**Functional:**
- Upload videos
- Stream videos
- Search for videos
- Comments, likes, subscriptions (bonus)
- Recommendations (bonus)

**Non-functional:**
- 500 hours of video uploaded per minute
- 1B hours of video watched per day
- Available worldwide, low buffering
- Videos must be durable (never lost)
- Storage-intensive, bandwidth-intensive

### Estimation

```
Uploads: 500 hours/minute = 30,000 minutes of video/minute
  At 2Mbps raw: 30,000 × 60s × 2Mbps = 3.6Gbps of raw upload data/minute

After transcoding to multiple resolutions:
  360p (500Kbps) + 720p (2Mbps) + 1080p (5Mbps) + 4K (20Mbps) = ~28Mbps per video
  Storage per minute of video: ~28Mbps × 60s / 8 = 210MB per minute of video

Annual storage:
  500 hours/min × 60 min/hr × 24 hr/day × 365 days = 262.8M hours/year
  210MB × 60 min = 12.6GB per hour of video
  262.8M hours × 12.6GB ≈ 3.3 Exabytes per year!

Video views: 1B hours/day
  Average video: 7 minutes
  1B × 60 / 7 = 8.57B video sessions/day
  100K sessions/second
  At 720p (2Mbps): 100K × 2Mbps = 200Gbps of bandwidth needed at peak
```

### Architecture

```
Upload Flow:
  User → Upload Service → S3 (raw video, private bucket)
       → Returns upload_id
  
  Upload Service → Kafka: VideoUploaded { video_id, s3_key, user_id }
  
  Transcoding Service (cluster of powerful EC2 instances):
    1. Download raw video from S3
    2. Transcode to multiple resolutions & codecs (FFmpeg or AWS MediaConvert)
       360p, 480p, 720p, 1080p, 4K (where applicable)
       AVC/H.264 (max compatibility) + AV1/VP9 (modern, 30-50% better compression)
       HLS segments (2-10 second .ts files) for adaptive bitrate
    3. Upload all variants to S3 (CDN-backed bucket, public)
    4. Update database: video_id → {s3_paths_per_quality, duration, status=READY}
    5. Publish VideoProcessed event to Kafka
  
  Video record published to CDN:
    CloudFront distribution in front of S3
    manifest.m3u8 → TTL: 60 seconds
    .ts segments → TTL: 86400 seconds (immutable content)
    
Playback Flow:
  User → CDN (nearest PoP) → GET /videos/abc123/manifest.m3u8
  Client player: parse manifest → select quality (e.g., 720p) → fetch segments
  Each segment: served directly from CDN edge (< 5ms for HIT)
```

### Adaptive Bitrate Streaming (ABR)

```
manifest.m3u8:
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=640x360
/hls/abc123/360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
/hls/abc123/720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
/hls/abc123/1080p/playlist.m3u8

Player behavior:
  Start: estimate bandwidth → pick appropriate quality
  Buffer: monitor download speed → switch quality if bandwidth changes
  Buffering detected: automatically drop to lower quality (smooth playback > HD)
```

### Search at YouTube Scale

Every video is indexed for full-text search.

```
When video is processed:
  - Extract title, description, tags, auto-generated transcript (speech-to-text)
  - Publish to Elasticsearch for search
  - Publish to recommendation ML pipeline (category, tags, channel history)
  
Search query: "cooking pasta"
  → Elasticsearch: text match on title/description/transcript
  → Ranked by: relevance score × view count weight × recency weight
  → Top K results → Recommendation re-ranking (personalized for this user)
```

### Recommendations (Simplified)

YouTube's recommendation engine is one of the most complex ML systems in the world. Simplified:

```
Candidate generation (fast, approximate):
  - Matrix factorization: user × video embeddings → similar users liked
  - Content-based: user's history → similar content embeddings
  - Returns ~100 candidates

Ranking (slower, precise):
  - Deep neural network (ranking model)
  - Features: video popularity, watch time, user history, CTR prediction
  - Returns top 10-20 ranked recommendations

Deployed: TensorFlow models served by TF Serving on GPU clusters
```

### Video Data Model

```sql
CREATE TABLE videos (
    video_id    VARCHAR(11) PRIMARY KEY,  -- YouTube-style 11-char ID
    user_id     BIGINT NOT NULL,
    title       VARCHAR(100),
    description TEXT,
    tags        TEXT[],
    status      ENUM('uploading', 'processing', 'published', 'deleted'),
    duration_seconds INT,
    view_count  BIGINT DEFAULT 0,        -- updated async (eventually consistent)
    like_count  BIGINT DEFAULT 0,
    created_at  TIMESTAMP,
    published_at TIMESTAMP,
    
    -- S3 paths per quality (stored as JSONB)
    video_qualities JSONB  -- {"360p": "s3://...", "720p": "s3://..."}
);

CREATE TABLE video_views (
    video_id   VARCHAR(11),
    user_id    BIGINT,
    watched_at TIMESTAMP,
    watch_duration_seconds INT,
    PRIMARY KEY (video_id, user_id, watched_at)
);
-- Partitioned by watched_at for efficient time-series queries
```

---

## 🚗 Case Study 4: Uber / Ride-Sharing

### Requirements

**Functional:**
- Rider requests a ride (location, destination)
- System matches rider with nearby driver
- Driver and rider can track each other's real-time location
- Compute and display fare estimate
- Trip history

**Non-functional:**
- Real-time: driver location updates every 4 seconds
- Low latency matching (< 5 seconds to find a driver)
- Globally distributed (150+ cities)
- Availability critical (peak hours, high traffic)

### Estimation

```
Uber stats (2023 approx):
  30M trips/day
  5M drivers active at peak

Driver location updates:
  5M drivers × update every 4 seconds = 1.25M location updates/second

Ride requests: 30M / 86400 ≈ 347 requests/second
Peak: ~1000 requests/second

Location storage (for finding nearby drivers):
  5M active drivers, each with (driver_id, lat, lng, last_updated)
  This must be queryable for "drivers within 5km of lat/lng" in < 100ms
```

### Geospatial Indexing: The Core Problem

How do you find all drivers within 5km of a given location efficiently?

**Naive approach:** `SELECT driver_id FROM driver_locations WHERE distance(lat, lng, req_lat, req_lng) < 5km`
- Must scan ALL 5M rows → compute distance for each → too slow!

**Geohash:**
Geohash encodes a lat/lng into a short alphanumeric string. Nearby locations have similar geohash prefixes.

```
San Francisco (37.7749, -122.4194) → Geohash: "9q8yy"
1 block north (37.776, -122.4194) → Geohash: "9q8yy"  (same prefix!)
Los Angeles (34.0522, -118.2437)  → Geohash: "9muu"   (different prefix)

Query: Find drivers within ~5km of request point
1. Compute geohash of request point: "9q8yy"
2. Find 8 neighboring geohash cells (to handle border cases)
3. Query: SELECT * FROM drivers WHERE geohash LIKE '9q8yy%' OR geohash LIKE '9q8yz%' ...
4. Compute exact distance on the ~100 results

With B-tree index on geohash prefix → fast string prefix scan!
```

**H3 (Hexagonal Hierarchical Geospatial Indexing) — Uber's own library:**
Uber open-sourced H3 in 2018. Uses hexagonal grid cells (more uniform area than squares).

```
lat/lng → H3 cell ID at resolution 9 (~174m cells)
Query: find all drivers in current cell + 1-ring neighbors = 7 cells
→ O(7) Redis lookups instead of O(N) distance calculations
```

**Redis GEO:**
Redis has native geospatial support using a sorted set with geohash scores.

```python
# Store driver location
redis.geoadd('drivers', longitude, latitude, driver_id)

# Find drivers within 5km
nearby = redis.georadius('drivers', req_lng, req_lat, 5, 'km', count=20)
# Returns list of (driver_id, distance, coordinates)
```

### Architecture

```
Driver Location Updates (1.25M/second):
  Driver app → WebSocket Gateway → Location Update Service
  Location Update Service → Redis GEO SET (overwrite driver's position)
  Also → Kafka topic "driver_locations" (for analytics, surge pricing ML)

Rider Requests a Ride:
  Rider app → API Gateway → Matching Service
  Matching Service:
    1. Find nearby available drivers within 5km (Redis GEO query)
    2. Filter by: driver is available, passes ride requirements
    3. Rank by: ETA (estimated time to arrive), rating, driver preferences
    4. Offer trip to top N drivers (nearest first)
    
Driver Trip Offer & Accept:
  Push notification to driver app (WebSocket or FCN)
  Driver has 15 seconds to accept
  If declines/no response: try next driver
  Race condition: multiple riders sending offers → driver can only accept one
    Solution: Redis distributed lock per driver for the 15-second window

Trip in Progress:
  Both rider and driver apps send location every 4 seconds via WebSocket
  WebSocket Gateway distributes:
    Driver location → Rider's app (so rider can see driver on map)
    Rider location → Driver's app (navigation)
  
ETA Calculation:
  Input: (driver_lat, driver_lng, rider_lat, rider_lng)
  Real-time traffic data from ML model + historical patterns
  Returns: ETA in seconds (used for matching and display)
```

### Surge Pricing

```
Supply: number of available drivers in area
Demand: ride requests in last 5 minutes in area

When demand > supply by a threshold → multiply base fare by surge factor (1.2x, 1.5x, 2x, 3x)

Uber divides cities into hexagonal cells (H3)
For each cell, every 5 seconds:
  drivers_available = count drivers in cell with status=AVAILABLE
  requests_5min = count ride requests from cell in last 5 min
  surge_multiplier = ml_model.predict(drivers_available, requests_5min, hour, weather)

Surge data stored in Redis → very fast read for every ride request
```

### Fault Tolerance for Matching

```
Matching Service crashes during driver selection?
  Use optimistic locking: driver state stored in DB with version number
  Only one service can change driver from AVAILABLE → ON_TRIP at a time
  
  UPDATE drivers SET status='ON_TRIP', version=version+1
  WHERE driver_id=123 AND status='AVAILABLE' AND version=current_version
  
  If returns 0 rows updated → another service got there first → retry with different driver
```

---

## 💬 Case Study 5: WhatsApp / Chat Messaging

### Requirements

**Functional:**
- One-to-one messaging
- Group messaging (up to 256 members)
- Media messages (images, video, documents)
- Message delivery status (sent, delivered, read)
- Online presence (last seen)
- End-to-end encryption (E2EE)

**Non-functional:**
- 2B+ users
- 100B+ messages/day
- Messages must be delivered (at-least-once)
- Low latency (< 100ms for message delivery)
- E2EE (server never sees plaintext message content)

### Estimation

```
100B messages/day / 86400 = 1.16M messages/second
  Peak: ~3M messages/second

Storage:
  Most messages are text (~200 bytes each)
  100B × 200B = 20TB/day (text only)
  Media: ~30% have media, avg 100KB = 30B × 100KB = 3PB/day for media
  
  WhatsApp stores messages on user devices (E2EE) — server stores only briefly for delivery
  After delivery: messages deleted from server (or stored encrypted on user's backup choice)
  
Connections:
  2B users, say 500M online simultaneously (peak)
  500M persistent WebSocket connections
  Each connection: ~50KB RAM → 25TB RAM just for connection state
  → Must distribute across thousands of servers!
```

### The Connection Problem

500M concurrent WebSocket connections can't go to one server. Must shard connections.

```
Approach: Consistent Hashing for Connection Servers
  User connects → Gateway Server (consistent hash of user_id → server)
  User A always goes to Gateway Server 3
  User B always goes to Gateway Server 7

  When User A sends message to User B:
    A's gateway server needs to know: "which server holds User B's connection?"
    
  Solution: Presence Service (Redis)
    On connect: SET user:{id}:server "gateway-3" EX 300  (TTL=5 minutes, refreshed)
    On message: lookup user_b's gateway server → route to that server → push to user_b

Message Flow:
  A sends message to B:
  1. A's WebSocket → Gateway Server 3 (A's home server)
  2. Gateway Server 3: persist message to Cassandra (for delivery guarantee)
  3. Gateway Server 3: lookup User B's server via Redis ("gateway-7")
  4. Gateway Server 3 → Gateway Server 7: "deliver message to User B"
  5. Gateway Server 7 → User B's WebSocket: deliver message
  6. User B's client sends delivery ACK → Gateway 7 → Gateway 3 → User A (✓ delivered)
  
  If User B is offline:
    Message stored in Cassandra with status "pending"
    When B comes online: Gateway Server fetches pending messages from Cassandra → delivers
```

### Message Storage (Cassandra)

```sql
-- Primary message storage (Cassandra data model)
CREATE TABLE messages (
    chat_id    UUID,           -- unique per conversation (hash of sorted user_ids)
    message_id TIMEUUID,       -- time-UUID: sortable by creation time
    sender_id  BIGINT,
    content    BLOB,           -- encrypted ciphertext (E2EE)
    media_url  TEXT,
    status     TINYINT,        -- 1=sent, 2=delivered, 3=read
    PRIMARY KEY (chat_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Pending messages for offline users
CREATE TABLE pending_messages (
    recipient_id BIGINT,
    message_id   TIMEUUID,
    chat_id      UUID,
    PRIMARY KEY (recipient_id, message_id)
);
```

Cassandra choice rationale:
- Write-heavy (millions of messages/second) → LSM tree optimized for writes
- Time-series data (recent messages most accessed) → partition by chat_id
- Global distribution → Cassandra's multi-datacenter support

### End-to-End Encryption (Signal Protocol)

WhatsApp uses the **Signal Protocol** for E2EE:

```
Key concepts:
  Each user generates an Identity Key Pair (long-term)
  Each user publishes Pre-Keys to WhatsApp server (signed one-time keys)
  
Establishing a session (first message from A to B):
  A downloads B's pre-key from server
  A performs Diffie-Hellman key exchange (using both key pairs)
  Result: shared secret that only A and B can compute
  
  Server NEVER sees the shared secret — can't decrypt messages even if subpoenaed

Message encryption:
  Message → encrypted with shared session key → sent to server as ciphertext
  Server delivers ciphertext → B decrypts with shared key
  
  Sender can verify: "Only B can decrypt this" (public key cryptography)
  
WhatsApp server only sees: encrypted blob, sender, recipient, timestamps
  Cannot decrypt even with full server access
```

### Group Messaging Architecture

```
Group of 256 members:
  Option 1: Send message to server → server fans out to 255 members' servers
    Server relays (more efficient for large groups)
    
  Option 2: Client sends 255 individual encrypted messages
    True E2EE: each message encrypted with recipient's key
    256 messages transmitted per group message
    WhatsApp chose this for smaller groups; server-side fan-out for larger

WhatsApp approach (sender keys):
  One-time setup: A generates SenderKey, encrypts with each group member's key, distributes
  Each subsequent message: encrypted with SenderKey (single message, much cheaper)
  If member leaves: new SenderKey generated (forward secrecy)
```

### Delivery Receipt Flow

```
Message status:
  ✓  (gray single check): Message sent to server
  ✓✓ (gray double check): Message delivered to recipient's device
  ✓✓ (blue double check): Message read by recipient

Implementation:
  User B's client sends delivery receipt: { message_id, status: "delivered" }
  This receipt goes through WhatsApp servers → to User A's client
  User A's client updates UI: message status to ✓✓
  
  User B opens the chat → B's client sends read receipt
  → A's client updates to blue ✓✓
  
  Privacy setting: B can disable read receipts → B still sees A's receipts, A can't see B's
  Implementation: if B has read_receipts=off → B's client doesn't send read receipt events
```

---

## 📋 Trade-offs Reference Card

| Design Decision | Option A | Option B | Choose A When | Choose B When |
|---|---|---|---|---|
| Fan-out | Write (push) | Read (pull) | Mostly read, few followers | Many followers (celebrities), write-heavy |
| DB for timelines | SQL | NoSQL (Cassandra) | Strong consistency needed | High write throughput, wide rows |
| URL ID generation | Random base62 | Counter + base62 | Privacy (non-predictable) | Simplicity, no collisions |
| Redirect type | 301 | 302 | Performance (browser caches) | Analytics accuracy |
| Message storage | Server-side | Client-side (E2EE) | Server needs to read content | Privacy compliance |
| Geospatial index | Redis GEO | PostGIS (PostgreSQL) | Sub-second, high-update | Complex queries, analytics |
| Video streaming | HLS | DASH | Apple ecosystem, wide compat | Google ecosystem |
| DB for messages | Cassandra | PostgreSQL | High write throughput | Complex queries, ACID |
| Consistency model | Strong | Eventual | Financial, inventory | Social feeds, view counts |

---

## 🎯 Key Takeaways

### Common Patterns Across All Systems

| Pattern | Used In | Why |
|---|---|---|
| Snowflake/UUID IDs | Twitter, YouTube, Uber | Globally unique, distributed ID generation |
| CDN for media | YouTube, Twitter, WhatsApp | Serve static assets from edge; origin only for dynamic |
| Redis for hot data | URL shortener, Twitter timeline, Uber drivers | Sub-millisecond reads for frequently accessed data |
| Kafka for fan-out | Twitter, YouTube | Decouple write from fan-out; async; replay |
| Cassandra for writes | WhatsApp, Twitter | LSM tree → high write throughput; time-series data |
| WebSockets | Uber, WhatsApp, Twitter | Real-time bidirectional; persistent connections |
| Consistent hashing | WhatsApp gateways, Uber drivers | Minimize resharding; distribute load evenly |
| Async processing | YouTube transcoding, Twitter fan-out | Don't make user wait; offload to background workers |
| Geospatial indexing | Uber | Redis GEO / H3 for sub-100ms proximity queries |

### Mindset for System Design

1. **Start with requirements.** Don't design in a vacuum. Read QPS, write QPS, latency requirements, and data model drive all architecture decisions.

2. **Scale is a spectrum.** Design for today's scale, with extension points for 10× growth. Don't over-engineer from day 1, but know your scaling levers.

3. **Trade-offs are real.** Every design decision is a trade-off. Know WHY you're making each choice — what you gain and what you sacrifice.

4. **State is the hard part.** Stateless services scale trivially. Stateful data (databases, caches, message queues) is where all the interesting engineering happens.

5. **Failure is the normal case.** Design for failure from the start: circuit breakers, retries, graceful degradation, health checks.

6. **Measurement beats guessing.** Back-of-envelope estimation → design choices. Profile before optimizing. Monitor in production. Data beats intuition.

---

*This completes the 16-document System Design series. Refer to [00_README.md](00_README.md) for the complete index and learning path.*
