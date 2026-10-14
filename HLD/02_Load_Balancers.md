# ⚖️ Load Balancers — Distributing the Weight of the World

> *"A system is only as strong as its weakest link — unless you design so that there are no single links."*

---

## 📋 Table of Contents

1. [What is a Load Balancer?](#what-is-a-load-balancer)
2. [L4 vs L7 Load Balancers](#l4-vs-l7-load-balancers)
3. [Load Balancing Algorithms](#load-balancing-algorithms)
4. [Hardware vs Software Load Balancers](#hardware-vs-software)
5. [Active-Passive vs Active-Active HA](#active-passive-vs-active-active)
6. [Sticky Sessions / Session Persistence](#sticky-sessions)
7. [Health Checks](#health-checks)
8. [SSL Termination](#ssl-termination)
9. [Global Server Load Balancing (GSLB)](#gslb)
10. [Load Balancer as Single Point of Failure](#spof)
11. [Real-Life Scenario](#real-life-scenario)
12. [🔥 Big Tech Story: GitHub's DDoS and Load Balancer Evolution](#big-tech-story)
13. [Trade-offs & Drawbacks](#trade-offs)
14. [Best Practices](#best-practices)
15. [Key Takeaways](#key-takeaways)

---

## 🤔 What is a Load Balancer?

A **load balancer** is a device (hardware or software) that distributes incoming network traffic across multiple backend servers (a "server pool" or "upstream group").

The goals are:
- **Prevent overload** on any single server
- **Maximize throughput** by using all available servers
- **Minimize response time** by routing to least-busy servers
- **Ensure high availability** by routing around failed servers
- **Scale horizontally** — add more servers without changing client configuration

Without a load balancer, every client must know the address of every server. With a load balancer, clients talk to a single IP/hostname, and the load balancer handles distribution transparently.

```
Without LB:                    With LB:
                                           ┌──────────┐
Client → Server1               Client──►  │   Load   │──► Server1
Client → Server2               Client──►  │ Balancer │──► Server2
Client → Server3               Client──►  │          │──► Server3
(Clients must manage this)     Clients     └──────────┘
                                just talk to one address
```

---

## 🔀 L4 vs L7 Load Balancers

The most important distinction in load balancing. The "L" refers to the OSI model layer at which the load balancer operates.

### L4 — Transport Layer Load Balancer

Operates at Layer 4 (TCP/UDP). Makes routing decisions based on:
- Source IP address
- Destination IP address
- Source/Destination port number

**It does NOT look inside the packets.** It doesn't know what's in the HTTP request. It just moves TCP/UDP streams.

**How it works:**
```
Client sends TCP packet to LB:
  SrcIP: 203.0.113.5, DstIP: 10.0.0.1 (LB), Port: 80

LB rewrites destination (NAT) and forwards:
  SrcIP: 203.0.113.5, DstIP: 10.0.1.2 (Server), Port: 80

(OR uses Direct Server Return where response goes directly to client)
```

**Characteristics:**
- Extremely fast (no packet inspection)
- Protocol-agnostic: works with any TCP/UDP protocol
- Cannot make decisions based on HTTP content (URL, cookies, headers)
- Cannot do SSL termination (doesn't understand SSL)
- Examples: AWS NLB, HAProxy (L4 mode), Linux ipvs (LVS)

### L7 — Application Layer Load Balancer

Operates at Layer 7 (HTTP/HTTPS). Terminates the connection from the client, reads the request, then makes a new connection to the backend server.

Makes routing decisions based on:
- HTTP URL path (`/api/*` vs `/static/*`)
- HTTP headers (`Host:`, `Cookie:`, `Authorization:`)
- HTTP method (GET, POST)
- Query parameters
- Request body content

**How it works:**
```
Client → LB:  GET /api/users/123 HTTP/1.1
              Host: api.example.com
              Cookie: session=abc123

LB inspects the request:
  - Path starts with /api/ → route to API servers
  - Cookie session=abc123 → sticky session to Server2

LB → Server2: GET /api/users/123 HTTP/1.1
              X-Forwarded-For: 203.0.113.5
```

**Characteristics:**
- Slower than L4 (must parse HTTP)
- Content-based routing (URL path, headers, cookies)
- SSL termination (decrypts HTTPS, routes HTTP internally)
- HTTP/2 support, WebSocket support
- Can modify requests/responses (add headers, rewrite URLs)
- Examples: AWS ALB, Nginx, HAProxy (HTTP mode), Envoy, Traefik

### Comparison Table

| Feature | L4 Load Balancer | L7 Load Balancer |
|---|---|---|
| OSI Layer | Transport (4) | Application (7) |
| Protocols | TCP, UDP | HTTP, HTTPS, gRPC |
| Routing basis | IP + Port | URL, headers, cookies |
| SSL Termination | ❌ No | ✅ Yes |
| Content-based routing | ❌ No | ✅ Yes |
| Performance | ✅ Faster (less processing) | Slightly slower |
| Health checks | TCP port checks | HTTP status code checks |
| Logging | IP/port level | Request-level (URL, status) |
| WebSocket support | ✅ Yes (passthrough) | ✅ Yes (with config) |
| AWS Product | NLB | ALB |
| Use case | Non-HTTP, ultra-high performance | HTTP APIs, microservices |

### When to Use Which?

**Use L4 when:**
- You need to load balance non-HTTP protocols (SMTP, MySQL, custom TCP)
- You need absolute maximum throughput (millions of packets/sec)
- You're doing UDP load balancing (DNS, game servers)
- You need to preserve client IP without X-Forwarded-For (Direct Server Return)

**Use L7 when:**
- You're running HTTP/HTTPS services (most web apps)
- You need path-based routing (`/api/*` → API servers, `/web/*` → web servers)
- You need SSL termination
- You need sticky sessions based on cookies
- You want request-level logging and metrics

**Common setup:** L4 LB in front of L7 LBs for extreme scale:
```
Internet → L4 LB (NLB) → Multiple L7 LBs (ALBs) → Application Servers
```

---

## 🎯 Load Balancing Algorithms

The algorithm determines which backend server receives each incoming request.

### 1. Round Robin

Routes requests sequentially through the server list.

```
Requests:  1  2  3  4  5  6
Servers:   A  B  C  A  B  C
```

**Pros:** Simple, even distribution  
**Cons:** Doesn't account for server capacity or current load. If requests have variable processing time, servers get uneven load.

**Best for:** Servers of equal capacity, stateless requests with similar processing time.

---

### 2. Weighted Round Robin

Like Round Robin but each server gets a weight. Servers with higher weights receive proportionally more requests.

```
Server A: weight 3 → gets 3 requests per cycle
Server B: weight 1 → gets 1 request per cycle
Server C: weight 2 → gets 2 requests per cycle

Cycle: A A A B C C → next cycle: A A A B C C...
```

**Pros:** Accounts for different server capacities (e.g., some servers have more CPU)  
**Cons:** Static weights don't adapt to runtime conditions

**Best for:** Heterogeneous server fleets (mix of instance types).

---

### 3. Least Connections

Routes each new request to the server with the **fewest active connections**.

```
Server A: 100 connections
Server B: 50 connections   ← New request goes here
Server C: 80 connections
```

**Pros:** Adapts to varying request processing times; naturally handles long-lived connections  
**Cons:** Connection count doesn't always correlate with actual CPU/memory load

**Best for:** Long-lived connections (WebSockets, database connections), workloads with variable processing time.

---

### 4. Least Response Time

Routes requests to the server with the **lowest average response time AND fewest active connections** (combines both metrics).

```
Server A: avg response 50ms, 100 connections → score = 5000
Server B: avg response 20ms, 200 connections → score = 4000  ← wins
Server C: avg response 80ms, 10  connections → score = 800   ← wins
```

**Pros:** Most accurate reflection of actual server health and performance  
**Cons:** More complex to compute; requires LB to track response times

**Best for:** High-performance APIs where latency matters.

---

### 5. IP Hash (Consistent Hashing on IP)

Routes requests based on a hash of the client's IP address, ensuring the same client always goes to the same server.

```
hash(client_IP) % num_servers = server_index
hash("203.0.113.5") % 3 = 2 → Server C (always)
```

**Pros:** Provides basic session persistence without cookies; works for L4  
**Cons:** Changing the number of servers changes all routing; if a server goes down, affected clients get rerouted; uneven distribution if clients are behind a NAT (many clients → same IP)

**Best for:** Simple stateful apps where all requests from one IP should hit the same server.

---

### 6. Resource Based (Adaptive)

LB agents on each server report current CPU, memory, and queue depth. LB routes to the server with most available resources.

**Pros:** Most accurate, prevents overloading any server  
**Cons:** Agents add overhead; more complex setup

**Best for:** Heterogeneous workloads, ML inference servers, CPU-heavy tasks.

---

### 7. Random

Routes each request to a randomly selected server.

**Pros:** Stateless, simple, works well at scale due to law of large numbers  
**Cons:** No intelligence; hot spots possible with small server counts

**Best for:** Large server pools (50+ servers) where randomness averages out.

---

### Algorithm Comparison

| Algorithm | Accounts for Server Load | Sticky Routing | Complexity | Best For |
|---|---|---|---|---|
| Round Robin | ❌ | ❌ | Low | Homogeneous servers, equal work |
| Weighted RR | Partially (static) | ❌ | Low | Heterogeneous capacity |
| Least Connections | ✅ (connection count) | ❌ | Medium | Long-lived connections |
| Least Response Time | ✅ (response time) | ❌ | High | Latency-sensitive APIs |
| IP Hash | ❌ | ✅ (by IP) | Low | Basic session persistence |
| Resource Based | ✅ (actual resources) | ❌ | High | Variable workload |
| Random | ❌ | ❌ | Very Low | Large fleets |

---

## 🖥️ Hardware vs Software Load Balancers

### Hardware Load Balancers (e.g., F5 BIG-IP, Citrix ADC)

Physical appliances purpose-built for load balancing. Use specialized ASICs/FPGAs for packet processing.

**Pros:**
- Extremely high throughput (100 Gbps+)
- Purpose-built hardware for packet processing
- Mature feature set (WAF, SSL offload, DDoS protection)
- Predictable performance

**Cons:**
- Expensive ($50K–$500K+ per appliance)
- Vendor lock-in
- Slow to provision/scale (physical hardware)
- Must buy ahead of peak (overprovisioning)

**Still used by:** Large enterprises, banks, telecoms with predictable traffic and compliance requirements.

### Software Load Balancers

Run on commodity hardware or in the cloud. Can be on bare metal, VMs, or containers.

**HAProxy:**
- Open source, battle-tested
- Excellent performance (handles millions of connections)
- Supports L4 and L7
- Advanced health checks, detailed stats
- Used by: Stack Overflow, GitHub, Reddit

**Nginx:**
- Originally a web server, excellent as reverse proxy/LB
- HTTP/2, WebSocket support
- Nginx Plus adds active health checks, dynamic reconfiguration
- Used by: Airbnb, Dropbox, Pinterest

**Envoy:**
- Modern, cloud-native L7 proxy from Lyft
- Designed for microservices (service mesh)
- Native gRPC support, excellent observability (stats, tracing)
- The proxy of choice in service meshes (Istio uses Envoy)
- Used by: Google, Lyft, Airbnb

### Cloud Load Balancers

| Provider | L4 | L7 | Global |
|---|---|---|---|
| AWS | NLB (Network LB) | ALB (Application LB) | Global Accelerator |
| GCP | TCP/UDP LB | HTTP(S) LB | Global HTTP LB |
| Azure | Azure LB | Application Gateway | Front Door |
| Cloudflare | — | Cloudflare LB | ✅ (Global anycast) |

**AWS ALB features:**
- Path-based routing (`/api/*` → API target group)
- Host-based routing (`api.example.com` vs `www.example.com`)
- WebSocket and HTTP/2 support
- Native integration with AWS WAF, ACM (SSL certs), ECS, EKS
- Weighted target groups (canary deployments: 5% → new version)

**AWS NLB features:**
- Handles millions of requests/second
- Static IP addresses (useful for whitelisting)
- Preserves source IP address
- TLS termination (added feature)
- Ultra-low latency

---

## 🔄 Active-Passive vs Active-Active HA

Load balancers themselves can be single points of failure. Here's how to make them highly available.

### Active-Passive (Primary-Standby)

```
                    ┌─────────────────────┐
Internet ──────────►│  LB-Primary (ACTIVE)│──► Backend Servers
                    └─────────────────────┘
                             │ Health check
                    ┌─────────────────────┐
                    │  LB-Secondary(PASSIVE)│  (warm standby)
                    └─────────────────────┘
```

- Both LBs share a **Virtual IP (VIP)** using protocols like VRRP (Virtual Router Redundancy Protocol) or HSRP
- Only the primary responds to the VIP
- If primary fails, secondary detects it (heartbeat timeout) and claims the VIP within seconds
- Failover: 2–30 seconds typically
- Secondary is idle (wasted capacity) during normal operation

**Best for:** Simple HA requirement, cost-sensitive deployments.

### Active-Active (Both LBs Handle Traffic)

```
                    ┌─────────────────────┐
Internet ──────────►│    LB-Primary       │──► Backend Set A
     (DNS round     └─────────────────────┘
      robin or      ┌─────────────────────┐
      anycast)─────►│    LB-Secondary     │──► Backend Set B
                    └─────────────────────┘
```

- Both LBs are in service simultaneously
- Traffic split between them (via DNS, anycast, or upstream routing)
- If one fails, the other takes all traffic
- Doubles the capacity; no wasted hardware

**Challenge:** Session synchronization between LBs (if using sticky sessions, LBs need to share session state)

**Best for:** High-traffic systems needing maximum throughput and HA.

---

## 🍪 Sticky Sessions / Session Persistence

### The Problem

Traditional web apps store user session state **on the server** (in memory or local files). If a load balancer sends a user's second request to a different server, that server doesn't have the session → user gets logged out.

### Solution: Sticky Sessions

The load balancer pins (sticks) a specific client to a specific server for the duration of their session.

**Method 1: Cookie-Based Stickiness (Preferred)**

The LB injects a cookie (e.g., `AWSALB`) into the first response. Subsequent requests include this cookie, and the LB routes to the same server.

```
Request 1:  Client → LB → Server B (no cookie)
Response 1: LB → Client (sets cookie: AWSALB=server-b-hash)

Request 2:  Client → LB (sends cookie: AWSALB=server-b-hash) → Server B (same!)
Request 3:  Client → LB (sends cookie) → Server B (same!)
```

AWS ALB sticky sessions use this method with configurable duration (1 second to 7 days).

**Method 2: IP-Hash Stickiness**

Routes based on client IP hash. Problem: doesn't work behind NAT (many users share one IP).

### Problems with Sticky Sessions

- **Uneven load distribution:** If a few clients have long-lived sessions, their pinned server gets overloaded
- **Failure hurts:** If a server goes down, all its pinned users lose sessions
- **Scaling is harder:** Adding a new server doesn't immediately receive sticky sessions from existing users
- **Anti-pattern at scale:** Major tech companies (Netflix, Twitter) store session state externally (Redis) so any server can handle any request — **stateless architecture**

**Production recommendation:** Store session state in Redis. Make all app servers stateless. Avoid sticky sessions.

---

## ❤️ Health Checks

Without health checks, a load balancer would route to dead servers. Health checks detect unhealthy backends and remove them from the rotation.

### Active Health Checks

The load balancer **proactively** sends requests to backend servers on a schedule.

**TCP Health Check:**
```
LB → Server: TCP SYN to port 8080
Server → LB: TCP SYN-ACK (server is alive)
```
Confirms the port is open but doesn't verify the application is healthy.

**HTTP Health Check:**
```
LB → Server: GET /health HTTP/1.1
Server → LB: 200 OK {"status": "healthy"}
```
Confirms the application is responding. Can check DB connectivity, cache connectivity, etc.

```python
# Example health check endpoint
@app.route("/health")
def health():
    try:
        db.execute("SELECT 1")  # Check DB
        redis.ping()             # Check cache
        return {"status": "healthy", "db": "ok", "cache": "ok"}, 200
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}, 503
```

**Configuration example (Nginx):**
```nginx
upstream backend {
    server server1:8080;
    server server2:8080;
    server server3:8080;
    
    # Health check (Nginx Plus feature)
    health_check interval=10s fails=3 passes=2 uri=/health;
}
```

**AWS ALB health check:**
```
Health check path:    /health
Healthy threshold:    2 (2 consecutive successes → mark healthy)
Unhealthy threshold:  3 (3 consecutive failures → mark unhealthy)
Interval:             30 seconds
Timeout:              5 seconds
```

### Passive Health Checks

The load balancer **observes** response errors on actual traffic.

```
If server returns 5xx status codes on 50% of requests in 30 seconds → mark unhealthy
If server connection times out 3 times → mark unhealthy
```

Nginx's `max_fails` and `fail_timeout` directives implement this:
```nginx
upstream backend {
    server server1:8080 max_fails=3 fail_timeout=30s;
    server server2:8080 max_fails=3 fail_timeout=30s;
}
```

**Best practice:** Use both active and passive health checks. Active catches silent failures; passive catches issues under real load.

---

## 🔒 SSL Termination

HTTPS traffic is encrypted end-to-end. Someone must decrypt it to inspect and route it.

### At the Load Balancer (Most Common)

```
Client ──HTTPS──► Load Balancer (decrypts) ──HTTP──► Backend Servers
```

**Benefits:**
- Centralized SSL certificate management
- Backend servers don't need to handle crypto (cheaper compute)
- LB can inspect HTTP content for routing (required for L7)
- Simpler backend deployment (no cert management per server)

**Drawbacks:**
- Traffic between LB and backend is unencrypted (mitigated if on private network/VPC)

### Re-Encryption (More Secure)

```
Client ──HTTPS──► LB (decrypts) ──HTTPS──► Backend Servers
```

LB decrypts, routes, then re-encrypts for the backend. More CPU overhead but traffic encrypted end-to-end.

### SSL Passthrough (L4 LB)

```
Client ──HTTPS──► L4 LB (doesn't decrypt) ──HTTPS──► Backend (decrypts)
```

L4 LB passes encrypted bytes without inspection. Backend handles decryption.

**Use when:** Regulatory requirements mandate end-to-end encryption and LB must not see plaintext.

---

## 🌍 Global Server Load Balancing (GSLB)

When your infrastructure spans multiple geographic regions, you need to route users to the nearest/healthiest region. This is GSLB.

### DNS-Based GSLB

The most common approach. The DNS server returns different IP addresses based on the requester's location.

```
User in Tokyo queries: api.example.com
DNS (GeoDNS) responds: 13.112.x.x (ap-northeast-1 ALB)

User in London queries: api.example.com
DNS (GeoDNS) responds: 18.185.x.x (eu-west-1 ALB)
```

**AWS Route 53 policies:**
- **Geolocation routing:** Route based on user's country/continent
- **Geoproximity routing:** Route based on physical distance to resource
- **Latency-based routing:** Route to the region with lowest measured latency
- **Failover routing:** Primary region + failover region
- **Weighted routing:** Split traffic by percentage (e.g., 90% region A, 10% region B for canary)

**Limitation:** DNS TTL means failover is not instant. If TTL=60s, users may hit a dead region for up to 60 seconds after failure.

### Anycast Routing (Faster)

Multiple servers worldwide share the **same IP address**. BGP routing automatically directs each client to the nearest server.

```
IP: 1.1.1.1 (Cloudflare's DNS)
This IP is announced from 200+ data centers worldwide.
Your packets go to the closest one automatically.
No DNS change required. Failover is BGP convergence speed (~seconds).
```

Used by: Cloudflare, Fastly, Google's 8.8.8.8 DNS

---

## 🚫 Load Balancer as Single Point of Failure

The irony: load balancers, designed to eliminate SPOFs, are themselves potential SPOFs.

### Solutions

**1. Multiple Load Balancers (Active-Active/Active-Passive)**
Already covered — use VRRP/Keepalived for on-prem, or use cloud-managed LBs (AWS ALB is inherently HA across AZs).

**2. DNS Round Robin**
Multiple LB IPs registered in DNS. Clients try each if one fails (with retries). Simple but not instant failover.

**3. Anycast**
Multiple LB nodes share one IP. BGP handles failover automatically. Used by CDNs and large platforms.

**4. Cloud Managed Load Balancers**
AWS ALB, GCP HTTPS LB, Azure Front Door are fully managed and implicitly HA. AWS ALB distributes load balancer nodes across multiple Availability Zones automatically.

---

## 🛒 Real-Life Scenario: Black Friday at an E-Commerce Site

**The situation:** An e-commerce platform handles 10,000 requests/second on normal days. On Black Friday, traffic spikes to 150,000 requests/second within minutes of deals going live.

**Without proper load balancing:**
- Single server → crashes at 5K RPS
- Even with 30 servers → no routing mechanism → clients hardcode server IPs → some servers overwhelmed, others idle

**The architecture with load balancers:**

```
Users (150K RPS)
     │
     ▼
AWS Route 53 (Latency-based routing)
     │
     ├──► us-east-1: ALB → 50 EC2 instances (Auto Scaling)
     ├──► eu-west-1:  ALB → 20 EC2 instances
     └──► ap-south-1: ALB → 10 EC2 instances

Each ALB:
  - Algorithm: Least Outstanding Requests
  - Health check: GET /health every 10s
  - Sticky: Disabled (sessions in Redis)
  - SSL termination: Yes
  - Auto Scaling: Min 10, Max 100, target 60% CPU
```

**What happens at 00:00 Black Friday:**
1. Traffic spikes 15x
2. Auto Scaling Group detects CPU > 60% → adds 40 EC2 instances in 3 minutes
3. ALB detects new healthy instances → starts routing to them
4. Least Outstanding Requests distributes evenly (long checkout requests don't overload one server)
5. One EC2 instance crashes (memory leak) → ALB health check fails → removed from rotation in 30 seconds
6. Users: seamless experience, 99.98% uptime

**Key design decisions:**
- Stateless servers (sessions in Redis) → ALB can route any user to any server
- Auto Scaling connected to ALB → new servers auto-registered
- Least Outstanding Requests → prevents slow checkout servers from accumulating requests

---

## 🔥 Big Tech Story: GitHub's Load Balancer Evolution

### The Problem (2012-2014)

GitHub was growing explosively — from millions to tens of millions of developers. Their infrastructure was simple: HAProxy running on a few servers, routing to a fleet of Rails application servers.

One day in 2012, they suffered a major outage. A misconfigured HAProxy reload caused a brief period where new connections were rejected. The reload (which swaps in new config) took milliseconds — but during those milliseconds, **thousands of connections were dropped**. For a developer platform, dropped git push/pull operations were catastrophic.

### GitHub's Solution: Seamless HAProxy Reloads

The team engineered **zero-downtime HAProxy reloads** using a clever socket-passing technique:
1. New HAProxy process starts, binds to the same port via `SO_REUSEPORT`
2. Old HAProxy drains existing connections
3. When all old connections are done, old process exits
4. No connections dropped

They published this as open source and it influenced HAProxy's own `--master-worker` mode.

### The DDoS Challenge

As GitHub became the center of the open source world, it became a target for DDoS attacks. The most famous: in **March 2015**, GitHub suffered the **largest DDoS attack ever recorded at that time** — 400 Gbps of traffic, originating from Chinese internet users whose browsers were being weaponized by a script injected by a Chinese CDN.

The attack targeted two specific GitHub pages that hosted tools for circumventing Chinese internet censorship.

GitHub's load balancers at that scale couldn't simply absorb it. Their strategy:
1. **Anycast routing** (via their CDN partners) to spread attack traffic globally
2. **Rate limiting at the edge** (dropping malformed requests)
3. **Traffic scrubbing** partnerships to filter attack traffic upstream
4. The attack eventually subsided after 5 days as GitHub worked with partners and the attack's effectiveness decreased

### Moving to Their Own Global LB

In 2018, GitHub (now acquired by Microsoft) began transitioning to **Azure's global infrastructure** with Azure Front Door — essentially a globally distributed L7 load balancer with CDN capabilities.

The key insight: **at scale, the load balancer and the CDN converge**. Both route traffic to the right server based on geography and health. Azure Front Door does both in one service.

**Lessons:**
1. Zero-downtime config reloads are non-negotiable in production
2. At GitHub's scale, a "load balancer" is a global traffic routing system, not a single box
3. DDoS protection must be built into the routing layer from the start
4. Running your own load balancer infrastructure is a significant engineering investment; cloud managed services are worth it for most companies

---

## ⚖️ Trade-offs & Drawbacks

### 1. Added Latency
Every request passes through the load balancer, adding network hops. For L7 LBs, SSL termination + HTTP parsing adds 1–5ms. For latency-sensitive systems, this matters.

**Mitigation:** Use L4 LBs for ultra-latency-sensitive protocols. Run LBs on fast instances close to backends.

### 2. The Load Balancer Itself is a Bottleneck
Even a load balancer has a maximum throughput. At extreme scale (millions of RPS), even powerful LBs can become bottlenecks.

**Mitigation:** Active-Active LBs, anycast routing, use L4 + L7 tiered architecture.

### 3. Cost
Cloud load balancers (AWS ALB, NLB) charge for:
- Per hour the LB exists (~$16–$22/month per LB)
- Per LCU (Load Balancer Capacity Unit) consumed by traffic

For low-traffic systems, this can be expensive relative to server costs.

**Mitigation:** Share ALBs across multiple services using host-based routing. Use path-based routing to serve multiple apps from one ALB.

### 4. SSL Certificate Management
Every LB must have valid SSL certificates. With multiple LBs across regions, certificate rotation becomes complex.

**Mitigation:** Use AWS ACM (auto-renews) or Let's Encrypt with automation. Wildcard certificates reduce the number needed.

### 5. Debugging Complexity
With a load balancer in the path, debugging production issues is harder. Which server did the request go to? What was the LB's behavior?

**Mitigation:** LBs should add headers (`X-Forwarded-For`, `X-Request-ID`). Enable access logs. Use distributed tracing.

### 6. WebSocket Complexity
WebSockets are long-lived connections. A load balancer must keep routing new packets in the WebSocket session to the same backend server. If the LB restarts or the server goes down, the WebSocket connection breaks.

**Mitigation:** Use sticky sessions or route WebSocket traffic differently. App should handle reconnection gracefully.

---

## ✅ Best Practices

1. **Make your backends stateless.** Store sessions in Redis. This lets any backend serve any request — no sticky sessions needed.

2. **Always configure health checks.** Use HTTP health checks that verify your app is actually working (not just TCP port open). Check DB and cache connectivity in the health endpoint.

3. **Use AWS ALB (or equivalent) for HTTP services.** Don't run your own HAProxy/Nginx unless you have a specific reason. Cloud LBs are managed, autoscaling, and multi-AZ by default.

4. **Enable access logging.** Load balancer access logs are invaluable for debugging. Route them to S3 and query with Athena.

5. **Set appropriate timeouts.** Connection timeout, idle timeout, request timeout — all should be configured. Default idle timeout on AWS ALB is 60s; increase for WebSockets.

6. **Use target group weights for canary deployments.** Route 5% of traffic to new version, watch metrics, then ramp up. Zero-risk deployments.

7. **Configure connection draining (deregistration delay).** When removing a server, the LB should let in-flight requests complete (30–60 seconds). Don't kill servers abruptly.

8. **Use separate target groups for different services.** Don't put your API servers and background job workers in the same target group. Health and capacity are different.

9. **Monitor LB metrics.** Track: RequestCount, TargetResponseTime, HTTPCode_Target_5XX, HealthyHostCount, UnHealthyHostCount. Set alarms.

10. **Test failover.** Regularly terminate instances to verify health checks and failover work correctly. Use chaos engineering practices.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **L4 vs L7** | L4 = fast, protocol-agnostic; L7 = intelligent routing, SSL, header inspection |
| **Round Robin** | Good default for homogeneous, stateless backends |
| **Least Connections** | Best for variable processing time or long-lived connections |
| **Sticky Sessions** | Avoid them; make backends stateless and store state in Redis |
| **Health Checks** | Both active (probing) and passive (observing) are necessary |
| **SSL Termination** | At the LB is standard; re-encrypt to backend for extra security |
| **GSLB** | Route users to nearest/healthiest region via DNS or anycast |
| **HA for LBs** | Use Active-Active with anycast or cloud-managed LBs (inherently HA) |
| **GitHub's Lesson** | At scale, the LB is a global traffic routing system, not a box |
| **State is the enemy** | Stateless backends + shared session store = unlimited horizontal scaling |

---

*Next: [03_Caching.md](03_Caching.md) — Supercharging read performance with Redis, cache strategies, and avoiding the thundering herd.*
