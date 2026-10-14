# Real-Time Notifications & Push Systems: WebSockets, SSE & Mobile Push at Scale

## The Real-Life Scenario
Consider the real-time communications scale of **WhatsApp**, **Slack**, or **Instagram**:
- **WhatsApp** delivers over **100 billion messages per day** to over 2 billion active users.
- When an Instagram creator with 50 million followers goes live, millions of followers receive a push notification within seconds.
- Meanwhile, in collaborative tools like Slack or Figma, when a coworker types a message or moves a cursor, everyone in the channel sees the update reflected in **< 50 milliseconds**.

Delivering low-latency, real-time updates to millions of simultaneously connected mobile and web clients requires specialized network protocols and connection gateway architectures.

---

## Real-Time Protocols: Polling vs Long Polling vs SSE vs WebSockets

```
+-------------------------------------------------------------------------+
|                  Real-Time Transport Protocol Mechanics                 |
+-------------------------------------------------------------------------+

 1. Short Polling:
    Client ──HTTP GET──► Server ──200 (No new data)──► Client (Repeats every 2s)
    ▲ Massive wasted HTTP header overhead, high latency, server connection exhaustion.

 2. Long Polling:
    Client ──HTTP GET──► Server (Holds connection open until event arrives) ──► 200 OK
    ▲ Lower latency than polling, but reconnect overhead on every event.

 3. Server-Sent Events (SSE):
    Client ──HTTP GET (text/event-stream)──► Server ──Stream 1──► Stream 2──► Stream 3
    ▲ Unidirectional (Server-to-Client) over HTTP/2, auto-reconnects, firewall friendly!

 4. WebSockets:
    Client ──HTTP Upgrade Handshake──► Server ◄──Full-Duplex Bidirectional TCP──►
    ▲ Sub-millisecond latency, minimal 2-byte frame overhead, persistent stateful TCP.
```

### Protocol Comparison Matrix

| Protocol | Direction | Overhead | Transport | Best Used For |
|---|---|---|---|---|
| **Short Polling** | Client $\to$ Server | Extreme (Full HTTP headers) | HTTP/1.1 | Simple dashboards, low-stakes polling |
| **Long Polling** | Client $\to$ Server | High | HTTP/1.1 | Legacy browser fallback |
| **SSE** | Server $\to$ Client | Low (HTTP/2 multiplexed) | HTTP/2 | Stock tickers, AI streaming, live feeds |
| **WebSockets** | Full-Duplex (Bidirectional) | Minimal (2-byte framing) | TCP | Gaming, Chat apps, Collaboration tools |

---

## Architectural Deep Dive: 10-Million-Connection WebSocket Gateway

WebSockets are **stateful TCP connections**. A client remains connected to a specific physical server process for hours. When User $A$ sends a message to User $B$, how does Server 1 find User $B$ connected to Server 42?

```
+-------------------------------------------------------------------------+
|                Distributed WebSocket Gateway Architecture               |
+-------------------------------------------------------------------------+
 [Client User A]                                    [Client User B]
        │                                                  │
        │ Persistent TCP Socket                            │ Persistent TCP Socket
        ▼                                                  ▼
 [Gateway Node 01]                                  [Gateway Node 42]
   (Maintains Epoll / Netty sockets)                  (Maintains Epoll / Netty sockets)
        │                                                  ▲
        │ 1. Look up User B route                          │ 4. Push frame to User B
        ▼                                                  │
 [Redis Session Registry]                                  │
 Key: "user:B:gateway" ──► "gateway-pod-42"                │
        │                                                  │
        │ 2. Publish to gateway-specific channel           │
        ▼                                                  │
 [Redis Pub/Sub / Kafka Message Bus] ──────────────────────┘
 Channel: "gateway.pod.42"
```

### 1. Connection Multiplexing (Solving the C10M Problem)
- Traditional thread-per-connection architectures (Tomcat) fail at $\approx 5{,}000$ connections due to OS thread memory allocation.
- Modern WebSocket gateways use **I/O Multiplexing (Linux `epoll` / Netty / Go goroutines)**: a single CPU core handles over **100,000 idle TCP connections** using non-blocking event loops with minimal RAM.

---

## Mobile Push Notifications: How APNs & FCM Work

Mobile operating systems (iOS and Android) aggressively kill background battery-draining apps. WebSockets cannot stay open in the background on mobile devices.

```
 [Application Backend]
          │
          │ 1. Dispatch Push Payload via HTTP/2 POST
          ▼
   ┌───────────────────────────────────┬───────────────────────────────────┐
   │ Apple Push Notification (APNs)    │ Firebase Cloud Messaging (FCM)   │
   │ - Device Token                    │ - FCM Registration Token          │
   └───────────────────────────────────┴───────────────────────────────────┘
          │                                   │
          │ 2. Persistent OS-Level Socket     │ 2. Persistent OS-Level Socket
          ▼                                   ▼
 [iOS Device (Apple OS daemon)]      [Android Device (Google Play Services)]
          │                                   │
          └─────────────────┬─────────────────┘
                            │ 3. Wakes up app / Displays system tray banner
                            ▼
                    [User Lock Screen]
```

---

## The Celebrity Notification Fanout Problem

When Cristiano Ronaldo publishes an Instagram Reel, sending 50 million push notifications simultaneously creates an acute system bottleneck:

```
                  [Celebrity Publish Event]
                             │
                             ▼
                  [Apache Kafka Topic]
                  Topic: "notification.fanout"
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
  [Worker Pool 1]     [Worker Pool 2]     [Worker Pool N]
  (Batch 1 - 50k)     (Batch 50k - 100k)  (Batch 100k - 150k)
         │                   │                   │
         └───────────────────┼───────────────────┘
                             ▼
               [Smart Notification Filter]
               - Has user turned off notifications?
               - Is user currently active in app? (Suppress push, show in-app banner!)
               - Frequency capping (Max 3 pushes per user per hour)
                             │
                             ▼ (Parallel HTTP/2 Connections)
                     [Apple APNs / Google FCM]
```

### Smart Fanout Optimizations:
1. **Tiered Delivery:** Send push notifications immediately to users who have opened the app in the last 24 hours. Batch and delay delivery to inactive users over a 30-minute window to avoid overwhelming third-party APNs/FCM connections.
2. **Notification Aggregation / Collapsing:** Instead of generating 100 notifications for 100 likes:
   - APNs `apns-collapse-id` / FCM `collapse_key`: Collapses multiple pending pushes into a single notification: *"Alice and 99 others liked your photo"*.

---

## Production Implementation: Server-Sent Events (SSE) in Python / FastAPI

SSE is the optimal, lightweight choice for one-way streaming (e.g., ChatGPT AI responses, financial market tickers, dashboard notifications):

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def event_generator(user_id: str):
    queue = asyncio.Queue()
    # Subscribe to Redis Pub/Sub for this user in background
    # redis_sub.subscribe(f"notify:{user_id}", queue)
    
    try:
        while True:
            # Wait for event from queue
            event_data = await queue.get()
            
            # SSE Protocol format: "data: <content>\n\n"
            yield f"event: notification\ndata: {event_data}\n\n"
    except asyncio.CancelledError:
        print(f"Client {user_id} disconnected.")

@app.get("/stream/notifications/{user_id}")
async def stream_notifications(user_id: str):
    return StreamingResponse(
        event_generator(user_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no" # Disables NGINX proxy buffering
        }
    )
```

---

## Real World Stories & Case Studies

### 1. WhatsApp's 2-Million-Connection Erlang Nodes
WhatsApp's early engineering team famously achieved **over 2 million concurrent TCP connections per server** with just 32 engineers:
- They utilized **Erlang / FreeBSD**, optimizing kernel network buffers (`kern.ipc.maxsockets`) and TCP send/receive buffer sizes down to $4\text{ KB}$ per socket to minimize RAM consumption.
- Erlang's lightweight actor model allocated only $\approx 300\text{ bytes}$ of memory per active connection process.

---

## System Design Interview Cheat Sheet: Notifications & Push

1. **Protocol Selection:** Use **WebSockets** for two-way chat/collaboration; use **SSE** for server-to-client streaming (notifications, AI output); use **APNs/FCM** for background mobile push.
2. **Stateful Gateway Routing:** Always detail the **Redis Session Registry** mapping `user_id` to `gateway_ip`.
3. **Celebrity Fanout Mitigation:** Mention **Tiered Delivery**, **Frequency Capping**, and **APNs Collapse Keys**.
4. **Idempotency:** Assign unique notification IDs to avoid alarming users with duplicate billing or security alerts.
