# 🌍 Content Delivery Networks (CDN)

> *"The speed of light is a hard limit. CDNs are how we engineer around it."*

---

## 📋 Table of Contents

1. [What is a CDN?](#what-is-a-cdn)
2. [CDN Architecture](#cdn-architecture)
3. [How CDN Caching Works](#how-cdn-caching-works)
4. [Cache-Control Headers Deep Dive](#cache-control-headers)
5. [CDN for Different Content Types](#content-types)
6. [Anycast Routing](#anycast-routing)
7. [CDN Providers Comparison](#cdn-providers)
8. [CDN for Security](#cdn-for-security)
9. [Edge Computing](#edge-computing)
10. [Cache Invalidation at CDN Level](#cache-invalidation)
11. [Video Streaming via CDN](#video-streaming)
12. [Multi-CDN Strategies](#multi-cdn)
13. [Real-Life Scenario](#real-life-scenario)
14. [🔥 Big Tech Story: Cloudflare BGP Hijacking Incident](#big-tech-story)
15. [Trade-offs & Drawbacks](#trade-offs)
16. [Best Practices](#best-practices)
17. [Key Takeaways](#key-takeaways)

---

## 🤔 What is a CDN?

A **Content Delivery Network (CDN)** is a geographically distributed network of servers that delivers content to users from the location nearest to them.

**The fundamental problem CDNs solve:** The speed of light.

```
Without CDN:
  User in Tokyo → Request → Origin Server in New York (11,000 km away)
  Speed of light: ~300,000 km/s → one-way: ~37ms → round trip: ~74ms
  Plus TCP handshake, TLS, server processing → total: 200-500ms per request

With CDN:
  User in Tokyo → Request → CDN Edge in Tokyo (5 km away)
  Round trip: ~1ms
  Content served from edge cache: < 5ms total
```

For a website making 50 HTTP requests per page load, this difference is 10 seconds vs 0.25 seconds.

**What CDNs deliver:**
- Static assets: HTML, CSS, JavaScript, images, videos, fonts
- API responses (cacheable GET requests)
- Video streaming (the biggest CDN use case by bandwidth)
- Software downloads
- Live streaming events

---

## 🏗️ CDN Architecture

### Points of Presence (PoPs)

CDNs operate **Points of Presence (PoPs)** — data centers strategically located worldwide in high-traffic internet exchange points (IXPs).

```
Global CDN Network:

New York (PoP) ──── London (PoP) ──── Frankfurt (PoP)
     │                   │                   │
Los Angeles (PoP)    Amsterdam (PoP)    Warsaw (PoP)
     │
São Paulo (PoP) ──── Johannesburg (PoP)
                                            
Tokyo (PoP) ──── Singapore (PoP) ──── Sydney (PoP)
     │
Mumbai (PoP) ──── Dubai (PoP)
```

Major CDNs have 100-300+ PoPs. Cloudflare has 300+. Akamai has 4,000+.

### Inside a PoP

Each PoP contains:
- **Edge servers:** Cache and serve content
- **Shield/Parent caches:** Intermediate cache between edges and origin
- **Load balancers:** Distribute requests among edge servers
- **Networking equipment:** BGP routers for anycast, firewall, DDoS scrubbing

```
User Request
    │
    ▼
Edge Server (L1 cache)
    │ MISS
    ▼
Shield/Parent Cache (L2 cache)
    │ MISS
    ▼
Origin Server (your infrastructure)
```

Multi-tier caching reduces origin load. Most requests are served from L1 edge. A percentage miss to L2 shield. Only a small fraction reaches origin.

### Origin Servers

Your application servers. The CDN is a transparent proxy in front of them.

```
                      CDN Layer
User ──────────► [Edge Server] ──► [Shield] ──► Origin Server
                     ↑
               (Serves cache HIT
                directly)
```

The origin doesn't know most of the user traffic — it only sees cache misses.

---

## 💾 How CDN Caching Works

### Origin Pull (Pull-Based CDN)

The CDN fetches content from origin on-demand when first requested.

```
Request #1 (Cache MISS):
  User → Edge Server: GET /image.jpg
  Edge → Origin: GET /image.jpg
  Origin → Edge: 200 OK + image data + Cache-Control: max-age=86400
  Edge: Caches image for 24 hours
  Edge → User: 200 OK + image data

Request #2 (Cache HIT):
  User → Edge Server: GET /image.jpg
  Edge: Image in cache, TTL not expired
  Edge → User: 200 OK + image data (X-Cache: HIT)
  (Origin never contacted!)
```

**Benefits:** Simple, only popular content is cached (no wasted caching of rarely accessed files)
**Drawbacks:** First user for each edge location gets slower response (cold start)

### Push-Based CDN

You proactively push content to CDN edge servers before users request it.

```
Deployment:
  Origin → Push /app.js, /style.css, /logo.png → all edge servers
  (Happens during release, before traffic arrives)

User Request:
  User → Edge: GET /app.js
  Edge: Always a cache HIT (pre-populated)
  Edge → User: 200 OK + app.js (instant)
```

**Benefits:** No cold start, guaranteed availability
**Drawbacks:** Must manage what's cached, expensive if content rarely accessed

**When to use push:** Known files you'll serve globally (JavaScript bundles, fonts, critical CSS). Software downloads. Video assets.

### CDN Cache States

- **HIT:** Content found in cache and not expired → served from edge
- **MISS:** Not in cache → fetch from origin, cache, serve
- **EXPIRED:** In cache but TTL elapsed → revalidate with origin
- **BYPASS:** Explicitly told not to cache → always fetches from origin
- **STALE:** TTL elapsed but serving stale while revalidating in background

Response headers indicate state:
```
X-Cache: HIT          # Cloudflare
CF-Cache-Status: HIT  # Cloudflare
X-Cache: Miss from cloudfront  # CloudFront
Age: 3600             # Seconds this response has been cached
```

---

## 📋 Cache-Control Headers Deep Dive

These HTTP headers are the primary mechanism to control CDN (and browser) caching behavior.

### The Most Important Directives

```http
Cache-Control: public, max-age=31536000, immutable
```

**`public`**
Response can be cached by any cache (CDN, proxy, browser). Without this, CDNs may not cache the response.

**`private`**
Only the browser can cache this response. CDNs and proxies must NOT cache it.
```http
Cache-Control: private, max-age=3600
# Use for: User-specific data (profile, account info)
```

**`max-age=N`**
Cache this response for N seconds. Both browser and CDN honor this.

**`s-maxage=N`**
Like `max-age` but only for shared caches (CDNs, proxies). Overrides `max-age` for CDNs.
```http
Cache-Control: public, max-age=60, s-maxage=3600
# Browser caches for 1 minute; CDN caches for 1 hour
```

**`no-cache`**
Misleading name! Does NOT mean "don't cache." Means "cache, but always revalidate before use."
```http
Cache-Control: no-cache
# Response is cached, but browser must check with server (If-None-Match/If-Modified-Since)
# Server can return 304 Not Modified → browser uses cached copy (saved download)
# Useful for: frequently updated resources where you want freshness but also fast revalidation
```

**`no-store`**
Truly do NOT cache. Never store this response anywhere. Use for sensitive data.
```http
Cache-Control: no-store
# Use for: banking transactions, login forms, sensitive PII
```

**`immutable`**
The response will never change. Don't bother revalidating even if max-age expires.
```http
Cache-Control: public, max-age=31536000, immutable
# Use for: content-addressed assets (app.a1b2c3d.js) where filename changes when content changes
# Browser won't even send a request to check for updates!
```

**`stale-while-revalidate=N`**
Serve stale content for N seconds extra while fetching fresh content in the background.
```http
Cache-Control: public, max-age=60, stale-while-revalidate=30
# After 60s: serve stale + fetch fresh in background (no user sees a slow response)
# After 90s: stale too old, must wait for fresh
```

**`stale-if-error=N`**
If origin returns an error (5xx), serve stale content for N seconds.
```http
Cache-Control: public, max-age=3600, stale-if-error=86400
# If origin is down: serve up to 24-hour-old cached content rather than showing error
# Critical for availability
```

### ETag & Conditional Requests

ETags enable efficient revalidation without re-downloading unchanged content.

```
Server → Client:
  HTTP 200 OK
  ETag: "v3a8b7c2"
  Content: [full body]

Client → Server (on next request):
  If-None-Match: "v3a8b7c2"

If content unchanged:
  HTTP 304 Not Modified
  (No body! Client uses cached version)
  Saved: full body download

If content changed:
  HTTP 200 OK
  ETag: "v4d9e1f3"
  Content: [new full body]
```

### Practical Caching Strategy

```
Hashed/versioned assets (app.a1b2c3d.js, style.xyz.css):
  Cache-Control: public, max-age=31536000, immutable
  → Cache for 1 year; never stale (filename changes when content changes)

HTML pages:
  Cache-Control: public, max-age=0, must-revalidate
  OR: Cache-Control: no-cache
  → Always revalidate (HTML changes frequently, points to new hashed assets)

API responses (slowly changing):
  Cache-Control: public, s-maxage=60, stale-while-revalidate=30
  → CDN caches 1 minute; 30s grace with background revalidation

User-specific API responses:
  Cache-Control: private, max-age=60
  → Browser only; CDN must not cache

Sensitive data (payments, auth):
  Cache-Control: no-store
  → Never cache anywhere
```

---

## 📁 CDN for Different Content Types

### Static Assets

Images, CSS, JavaScript, fonts — ideal for CDN caching. 

**Best practices:**
- Content-hash filenames: `logo.a1b2c3d4.png` — when content changes, filename changes → no cache invalidation needed
- Serve with `immutable` directive
- Enable Brotli/gzip compression at CDN level
- Use WebP format for images with fallback

### Dynamic Content Acceleration

Even non-cacheable dynamic content benefits from CDN:
- CDN uses faster **persistent connections** to origin (HTTP/2 multiplexing)
- CDN uses **anycast routing** to bring traffic closer to origin
- **TCP optimization** (reduced handshakes via connection reuse)
- Some CDNs (Cloudflare, Fastly) offer "dynamic content acceleration" for authenticated requests

### API Response Caching

Public API responses (weather data, product catalogs, currency rates) can be cached at CDN with short TTLs.

```
GET /api/products → Cache-Control: public, s-maxage=300
# CDN caches for 5 minutes
# 10,000 concurrent users → CDN serves all from cache
# Origin receives < 1% of traffic
```

---

## 📡 Anycast Routing

Anycast is a network routing technique where the **same IP address is advertised from multiple geographic locations**. The internet's routing protocol (BGP) automatically routes each user to the nearest location announcing that IP.

```
IP: 104.16.132.229 announced from:
  - New York (via AS13335)
  - London (via AS13335)
  - Singapore (via AS13335)
  - Sydney (via AS13335)

User in Tokyo → Internet routing → Singapore PoP (nearest)
User in Paris → Internet routing → London PoP (nearest)
User in Brazil → Internet routing → São Paulo PoP (nearest)
```

**Benefits of anycast over DNS-based routing:**
- **Faster:** No DNS lookup needed; routing happens at IP level
- **Faster failover:** BGP reconverges in seconds; DNS TTLs take minutes
- **No DNS TTL problem:** Clients use the same IP regardless of which PoP answers
- **Built-in DDoS mitigation:** Attack traffic is automatically distributed globally

**Who uses anycast:** Cloudflare, Fastly, Google's 8.8.8.8 DNS, root DNS nameservers.

---

## 🏢 CDN Providers Comparison

| Provider | Edge Locations | Strengths | Best For |
|---|---|---|---|
| **Cloudflare** | 300+ | Security (DDoS, WAF), performance, Workers edge computing | General web, DDoS protection, security-first |
| **Akamai** | 4,000+ | Enterprise-grade, massive scale, streaming | Large media companies, enterprises |
| **AWS CloudFront** | 450+ | Deep AWS integration, Lambda@Edge | AWS-native apps |
| **Fastly** | 90+ | Real-time purging (<150ms), powerful VCL config, Compute@Edge | High-frequency invalidation, media |
| **Google Cloud CDN** | 140+ | HTTP/2 push, GCP integration | GCP-native apps |
| **Azure CDN** | 130+ | Azure integration | Azure-native apps |
| **Bunny CDN** | 120+ | Very low cost, simple | Budget-conscious, startups |

### Decision Guide

**Choose Cloudflare:** If you need DDoS protection + CDN + DNS + security in one platform. Excellent free tier. Edge Workers for serverless at edge.

**Choose Akamai:** If you're a large media company delivering video at massive scale, or enterprise with strict SLAs. Most expensive but most battle-tested for extreme scale.

**Choose CloudFront:** If you're heavily on AWS (S3, ALB, API Gateway origin). Lambda@Edge for customization. Deep IAM integration.

**Choose Fastly:** If you need real-time cache purging (< 1 second) and complex caching logic. Popular with media companies (NY Times, Vox, GitHub).

---

## 🔒 CDN for Security

### DDoS Protection

CDNs are the first line of DDoS defense because:
- CDN has massive bandwidth capacity (Cloudflare: 197 Tbps capacity globally)
- Anycast distributes attack traffic globally → no single location overwhelmed
- CDN can identify and drop malicious traffic before it reaches origin

```
DDoS Attack: 500 Gbps of traffic
  → Hits Cloudflare's anycast network (distributed across 300+ PoPs)
  → Each PoP absorbs ~1.7 Gbps
  → Cloudflare scrubs malicious packets (UDP floods, SYN floods, HTTP floods)
  → Legitimate traffic passes through
  → Origin receives only normal traffic
```

**Types of DDoS attacks CDNs mitigate:**
- Volume-based (UDP flood, ICMP flood): raw bandwidth exhaustion
- Protocol attacks (SYN flood, Ping of Death): exploiting network protocol weaknesses
- Application layer (HTTP flood, Slowloris): HTTP-level attacks

### Web Application Firewall (WAF)

CDN WAF filters malicious HTTP requests before they reach your servers.

Rules block:
- SQL injection: `' OR 1=1 --`
- XSS: `<script>alert(1)</script>`
- Directory traversal: `../../etc/passwd`
- OWASP Top 10 attacks
- Custom rules based on your application

Cloudflare WAF, AWS WAF, Fastly WAF are cloud-managed WAF solutions integrated with CDN.

### Bot Management

CDNs identify and block automated traffic (scrapers, credential stuffers, inventory bots).

Methods:
- JavaScript challenges (prove you're a browser)
- CAPTCHA challenges
- Machine learning-based bot detection (unusual request patterns, timing)
- Device fingerprinting

---

## 🖥️ Edge Computing

Traditional model: All computation happens in your data center (origin).
Edge model: Computation happens at CDN edge, milliseconds from the user.

### Cloudflare Workers

JavaScript runtime running at Cloudflare's edge (300+ locations).

```javascript
// A/B testing at the edge (no origin request needed)
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
    const userId = getCookieValue(request.headers.get('Cookie'), 'user_id');
    const group = hash(userId) % 2 === 0 ? 'A' : 'B';
    
    // Rewrite URL to A or B variant
    const url = new URL(request.url);
    url.pathname = `/variants/${group}${url.pathname}`;
    
    return fetch(url.toString(), request);
}
```

**Use cases:**
- A/B testing without origin round-trip
- Request/response transformation
- Authentication at the edge (JWT validation without hitting auth server)
- Geolocation-based content customization
- Rate limiting per user
- Image optimization and resizing

### AWS Lambda@Edge

Lambda functions triggered by CloudFront events:
- **Viewer Request:** Before CloudFront checks cache
- **Origin Request:** After cache miss, before origin request
- **Origin Response:** After origin response, before caching
- **Viewer Response:** Before response is sent to user

```python
# Lambda@Edge: Add security headers to all responses
def lambda_handler(event, context):
    response = event['Records'][0]['cf']['response']
    headers = response['headers']
    
    headers['strict-transport-security'] = [{
        'key': 'Strict-Transport-Security',
        'value': 'max-age=31536000; includeSubDomains; preload'
    }]
    headers['content-security-policy'] = [{
        'key': 'Content-Security-Policy',
        'value': "default-src 'self'; script-src 'self' 'nonce-{NONCE}'"
    }]
    headers['x-content-type-options'] = [{'key': 'X-Content-Type-Options', 'value': 'nosniff'}]
    
    return response
```

### Fastly Compute@Edge

WebAssembly-based edge computing. Run code compiled from Rust, Go, JavaScript at the edge.

---

## 🗑️ Cache Invalidation at CDN Level

The hardest part of using a CDN: removing cached content before TTL expires.

### URL-Based Purging

```bash
# Cloudflare: Purge specific URL
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {api_token}" \
  -d '{"files":["https://example.com/api/products"]}'

# AWS CloudFront: Create invalidation
aws cloudfront create-invalidation \
  --distribution-id E1234567 \
  --paths "/api/products" "/static/app.js"
```

**Latency:** Cloudflare < 1 second. CloudFront: up to 15 minutes.

### Tag-Based (Surrogate Key) Purging

Tag cached responses with keys, then purge all responses with a given tag.

```
Response headers:
  Cache-Tag: product:789, category:electronics, sale:black-friday

# Later, when product 789 updates:
Purge all cached responses tagged: product:789
→ All pages showing product 789 are purged instantly
```

Supported by: Fastly (Surrogate-Key header), Cloudflare (Cache-Tag), Varnish.

This is the most efficient invalidation strategy for content-heavy sites. The NY Times uses Fastly's surrogate keys to invalidate related article pages when a story is updated.

### Versioned URLs (Best Strategy)

Avoid invalidation entirely by using content-hash URLs.

```
Before:  /static/app.js          (never-expiring cache → must invalidate on update)
After:   /static/app.a1b2c3d.js  (hash changes when content changes)

New deploy: app.e5f6g7h.js (new hash)
HTML points to new URL → CDN caches new file alongside old
Old file expires naturally → no invalidation needed
```

This is how modern frontend build tools (Webpack, Vite) work. Zero invalidation cost.

---

## 📹 Video Streaming via CDN

Video is the dominant CDN use case by bandwidth. Netflix consumes ~15% of global internet bandwidth. YouTube: another 10%.

### Adaptive Bitrate Streaming (ABR)

Videos are encoded at multiple quality levels. The player dynamically switches based on available bandwidth.

```
360p  (500 Kbps):   /video/abc/360p/segment-001.ts
720p  (2 Mbps):     /video/abc/720p/segment-001.ts
1080p (5 Mbps):     /video/abc/1080p/segment-001.ts
4K    (20 Mbps):    /video/abc/4k/segment-001.ts
```

**HLS (HTTP Live Streaming) — Apple's protocol:**
```
# Manifest file (.m3u8) — playlist of segments
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=640x360
/video/abc/360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
/video/abc/720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
/video/abc/1080p/playlist.m3u8
```

The player requests the manifest, chooses appropriate quality, downloads 2-10 second segments sequentially. Each segment is a standard HTTP GET request → **CDN serves them perfectly.**

### Why CDN is Essential for Video

Without CDN: 1 million users watching a video → 1 million requests to origin at 5 Mbps each = 5 Pbps. Impossible for any single origin.

With CDN: 1 million users → segment is cached at edge after first request → same segment served to all users from local cache. Origin sees only the initial cache-miss request.

```
First user: GET /segment-001-1080p.ts → MISS → fetch from origin → cache at edge
User 2-1,000,000: GET /segment-001-1080p.ts → HIT → serve from edge cache
```

Origin bandwidth: 1 request. CDN bandwidth: 1 million × 5 Mbps = 5 Tbps → distributed across global CDN infrastructure.

---

## 🔀 Multi-CDN Strategies

Relying on a single CDN creates a single point of failure. Netflix, Spotify, and major media companies use **multi-CDN**.

### Why Multi-CDN?

- **Resilience:** If Cloudflare has an outage, traffic fails over to Fastly/CloudFront
- **Performance:** Route users to whichever CDN has lowest latency for their region
- **Cost optimization:** Use cheapest CDN for each geography
- **Negotiating power:** Having multiple CDN contracts gives leverage in pricing

### Implementation

**Option 1: DNS-based CDN selection**
```
DNS layer:
  User in US → CNAME to Cloudflare
  User in APAC → CNAME to CloudFront
  User in EU → CNAME to Fastly

Health monitoring → if Cloudflare fails → switch US users to CloudFront
```

**Option 2: Real User Monitoring (RUM)**
Measure CDN performance for each user and route to fastest CDN. Companies like MCDN, Cedexis (now Citrix), and Fastly's Multi-CDN product offer this.

---

## 🌐 Real-Life Scenario: Video Streaming Startup

**The situation:** A video streaming startup wants to deliver 4K content globally to 10 million concurrent users.

**Challenge without CDN:**
- 10M users × 20 Mbps (4K) = 200 Tbps needed
- Origin servers would need 200 Tbps of bandwidth capacity
- Even with 1,000 servers at 200 Gbps each: impossible cost

**CDN Architecture:**

```
Content Pipeline:
  Creator uploads video → S3 (origin storage)
  → Transcoding service (generates HLS segments: 360p, 720p, 1080p, 4K)
  → Segments stored in S3 with Cache-Control: public, max-age=86400, immutable

CDN Layer:
  User → CDN Edge (nearest PoP) → [Cache HIT = serve segment]
                                   [Cache MISS = fetch from S3, cache, serve]

S3 (Origin) only serves:
  - First request for each segment from each PoP
  - Manifest files (shorter TTL: s-maxage=60, stale-while-revalidate=30)
```

**Cache hit rate calculation:**
- 10M users watching the same show at the same time
- Each segment (~5 seconds) served to all users from each PoP
- Tokyo PoP: first user fetches segment → next 500K users in Tokyo get cache HIT
- Cache hit rate: ~99.99%
- Origin sees: 200 Tbps × 0.01% miss rate = 20 Gbps → manageable!

**Other CDN configurations:**
```
Thumbnails:            Cache-Control: public, max-age=604800, immutable
Manifest files (.m3u8): Cache-Control: public, s-maxage=60, stale-while-revalidate=30
Video segments:        Cache-Control: public, max-age=86400, immutable
User watch progress:   Cache-Control: private, no-store  (user-specific, never cache)
```

**Total infrastructure cost reduction:** 40x cheaper than serving directly from origin.

---

## 🔥 Big Tech Story: Cloudflare BGP Hijacking Incident

### The Background

On June 24, 2019, a small internet service provider in Pennsylvania called **Verizon** (actually a customer of Verizon's ISP services, using Verizon as an upstream) caused one of the largest internet outages in recent memory.

The cause? A **BGP route leak**.

### How BGP Works (Briefly)

The internet is composed of thousands of **Autonomous Systems (ASes)** — networks operated by ISPs, cloud providers, CDNs, enterprises. They communicate reachability information using **BGP (Border Gateway Protocol)**.

When Cloudflare announces "I have the IP range 104.16.0.0/12," every other AS on the internet routes traffic destined for those IPs to Cloudflare's network. This is how the internet works.

### What Happened

1. A small company called **Allegheny Technologies** was using **DQE Communications** (a small ISP) as their internet provider.
2. DQE Communications used **Verizon** as their upstream provider.
3. Allegheny's network equipment had a misconfiguration: it was **advertising ALL internet routes it learned from Verizon back to Verizon** — 368,000+ routes.
4. **Verizon accepted these routes** without proper filtering (a major BGP hygiene failure).
5. **Verizon then propagated these routes globally** — telling the entire internet that the best path to Cloudflare, Amazon AWS, Facebook, Google, and others was through... Allegheny Technologies (a small company with very limited bandwidth).

### The Impact

Traffic from a significant portion of the internet was routed to Allegheny's tiny network.

- **Cloudflare** — completely unreachable from large portions of the internet for ~2 hours
- **Amazon AWS** — severely degraded
- **Facebook** — unreachable for many users
- **Fastly** — degraded
- Millions of websites that relied on these services → offline

The traffic trying to reach Cloudflare's IP addresses hit Allegheny's tiny links and was dropped.

### Why Cloudflare Was Particularly Affected

Cloudflare uses **anycast** for its CDN. All their CDN traffic uses IPs like `104.16.x.x`. When BGP told the internet that the best path to `104.16.x.x` was through Allegheny, ALL Cloudflare CDN traffic (globally) was affected simultaneously.

This is both a strength and weakness of anycast: the same IP everywhere means a BGP leak affects all users of those IPs at once.

### What Should Have Prevented This

**RPKI (Resource Public Key Infrastructure):** A system where IP address holders cryptographically sign their route announcements. Routers can reject invalid routes.

Cloudflare, Verizon, and the internet community have pushed for RPKI adoption since this incident. As of 2024, ~40% of internet routes are covered by RPKI (up from <20% in 2019).

**IRR (Internet Routing Registry) filtering:** ISPs should filter routes learned from customers based on registered prefixes. Verizon should have had filters preventing Allegheny from advertising 368,000 global routes.

### Cloudflare's Response

- Cloudflare became a vocal advocate for RPKI adoption
- They launched **Cloudflare RPKI Portal** (free tool for networks to validate routes)
- They published detailed technical post-mortems educating the internet community
- They worked with ARIN (American Registry for Internet Numbers) to accelerate RPKI deployment

### The Lesson for System Designers

Even a well-engineered CDN with 300 PoPs and anycast can be taken down by something completely outside your control: a misconfigured router at a company you've never heard of in Pennsylvania.

**Mitigations:**
1. **Multi-CDN:** If one CDN's IP routes are hijacked, traffic failovers to another CDN
2. **RPKI validation:** Reject cryptographically invalid routes
3. **Monitoring:** BGP monitoring services (BGPmon, CAIDA's BGP Visibility Scanner) can detect route leaks in near-real-time
4. **Architecture diversity:** Critical paths should have DNS-based fallback to different IP ranges

---

## ⚖️ Trade-offs & Drawbacks

### 1. Cache Consistency

The core tension: longer cache TTL = better performance = more stale data risk.

If you set `max-age=86400` and update your product price, users may see old price for 24 hours. Balance between performance and freshness based on data criticality.

**Mitigation:** Content-hash URLs for immutable assets. Short TTL + stale-while-revalidate for frequently updated content.

### 2. Dynamic Content Limitations

CDNs excel at static content. Personalized, dynamic content (user-specific pages) cannot be cached per user. A page showing `Hello, Alice!` cached would show "Hello, Alice!" to Bob.

**Mitigation:** Use Edge Side Includes (ESI) to separate cacheable components from user-specific components. Or use edge computing to personalize cached responses.

### 3. Cost

CDN pricing has two components:
- **Request charges:** Per HTTP request to CDN
- **Bandwidth charges:** Per GB transferred

For high-traffic sites, CDN costs can be significant. Cloudflare Pro/Business/Enterprise, AWS CloudFront, Fastly — all charge based on traffic.

**Mitigation:** Maximize cache hit rate (higher cache hit = lower origin bandwidth = lower total cost). Optimize content (smaller assets = less bandwidth).

### 4. Vendor Lock-in

Deep integration with one CDN (Lambda@Edge, Cloudflare Workers) makes switching painful.

**Mitigation:** Keep edge compute logic minimal and portable. Use standards-based approaches where possible.

### 5. Privacy Concerns

By routing all user traffic through CDN, the CDN provider sees all HTTP requests, user IPs, and response bodies (for HTTP; HTTPS bodies encrypted except between user and edge).

In GDPR-sensitive contexts, having user data routed through US CDN providers may have compliance implications.

### 6. BGP Vulnerabilities (as shown above)

Anycast-based CDNs are vulnerable to BGP route leaks/hijacking, which can be caused by third parties outside your control.

---

## ✅ Best Practices

1. **Use content-hash URLs for all static assets.** `app.a1b2c3d.js` → permanent caching, no invalidation needed.

2. **Set meaningful Cache-Control headers on every response.** Never rely on CDN default behavior. Be explicit.

3. **Use `stale-while-revalidate` for frequently-updated content.** Users never see latency from revalidation while content stays fresh.

4. **Monitor cache hit rates.** Target > 90% hit rate. Below 80% means your caching strategy needs work.

5. **Enable compression at CDN level.** Brotli is 15-25% smaller than gzip. CDN should compress before caching.

6. **Use CDN for DDoS protection, not just performance.** Put your CDN in front of everything, not just static assets.

7. **Test CDN behavior.** Use `curl -I https://yourdomain.com/asset` and check response headers (`X-Cache`, `CF-Cache-Status`, `Age`). Verify your caching intentions.

8. **Set short TTLs on HTML pages, long TTLs on hashed assets.** HTML points to the latest version (new hashed filenames after deploy). Assets are immutable once deployed.

9. **Purge strategically, not by URL.** Use tag-based purging (surrogate keys) for content relationships. Avoid purging by URL for large-scale content updates (too many API calls).

10. **Implement multi-CDN for critical services.** At minimum, have a secondary CDN ready for failover. Test the failover regularly.

11. **Use CDN analytics.** CDN dashboards show real-time traffic, cache hit rates, and geographic distribution. Use this data to optimize your caching strategy.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **CDN purpose** | Reduce latency by serving from the closest edge; reduce origin load through caching |
| **Origin pull** | CDN fetches on first miss; ideal for most content |
| **Cache-Control** | `public, max-age=31536000, immutable` for hashed assets; `no-cache` for HTML |
| **Anycast routing** | Same IP from 300+ locations; BGP routes to nearest → inherent DDoS resistance |
| **Edge computing** | Run code (JS/Wasm) at CDN edge for ultra-low latency personalization |
| **Cache invalidation** | Content-hash URLs are the best "invalidation strategy" — no invalidation needed |
| **Surrogate keys** | Tag-based bulk purging for content that has complex relationships |
| **Video streaming** | CDN is not optional for video at scale; ABR + HLS segments are perfectly CDN-cacheable |
| **Multi-CDN** | Single CDN is a SPOF at the routing level; multi-CDN with DNS failover is best practice |
| **Cloudflare BGP lesson** | Even perfectly engineered CDNs can fail due to global BGP infrastructure issues |

---

*Next: [08_Microservices.md](08_Microservices.md) — Building and operating distributed microservices with DDD, circuit breakers, service mesh, and Kubernetes.*
