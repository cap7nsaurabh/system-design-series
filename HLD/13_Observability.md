# 📊 Observability: Monitoring, Logging & Tracing

> *"You can't fix what you can't see. And in distributed systems, you often can't see what's broken."*

---

## 📋 Table of Contents
1. [The Three Pillars](#three-pillars)
2. [Metrics: Prometheus & Grafana](#metrics)
3. [SLIs, SLOs, SLAs & Error Budgets](#slos)
4. [Logging: ELK, Loki & Structured Logs](#logging)
5. [Distributed Tracing: OpenTelemetry & Jaeger](#tracing)
6. [Alerting Best Practices](#alerting)
7. [Chaos Engineering](#chaos-engineering)
8. [APM Tools](#apm-tools)
9. [Real-Life Scenario](#real-life-scenario)
10. [🔥 Big Tech Story: Netflix Chaos Monkey](#big-tech-story)
11. [Trade-offs & Best Practices](#trade-offs)
12. [Key Takeaways](#key-takeaways)

---

## 🏛️ The Three Pillars of Observability

**Observability** is the ability to understand the internal state of a system from its external outputs. The three pillars together answer different questions:

| Pillar | Question Answered | Data Type | Example Tool |
|---|---|---|---|
| **Metrics** | "Is the system healthy? How is it performing?" | Numeric time-series | Prometheus, Datadog |
| **Logs** | "What exactly happened? What were the details?" | Structured/unstructured text | ELK, Loki |
| **Traces** | "Where did this request go? What was slow?" | Request path with timing | Jaeger, Zipkin, AWS X-Ray |

**You need all three.** Metrics tell you something is wrong; logs tell you what the error message was; traces show you exactly which service and which code path caused it.

```
Scenario: API p99 latency spikes from 200ms to 2000ms

Metrics → "p99 latency is 2000ms on /checkout endpoint"
Logs    → "DB timeout errors: 'connection timeout after 1500ms'"  
Traces  → "Trace abc123: Order Service (50ms) → Payment Service (1750ms) ← HERE"
         → "Span: DB query SELECT * FROM transactions WHERE user_id=? took 1600ms"
         → "Problem: missing index on transactions.user_id!"
```

Without all three, you're guessing.

---

## 📈 Metrics: Prometheus & Grafana

### Metric Types

**Counter:** Monotonically increasing value. Resets on restart.
```
http_requests_total{method="POST", endpoint="/checkout", status="200"} 54821
http_requests_total{method="POST", endpoint="/checkout", status="500"} 12

# Query (PromQL): Rate of errors per second over 5 minutes
rate(http_requests_total{status="500"}[5m])
```

**Gauge:** A value that can go up or down.
```
active_connections 342
memory_used_bytes 2147483648
queue_depth 1205
```

**Histogram:** Samples observations into configurable buckets. Used for latency.
```
http_request_duration_seconds_bucket{le="0.1"}  9800  # 9800 requests < 100ms
http_request_duration_seconds_bucket{le="0.5"}  9950  # 9950 requests < 500ms
http_request_duration_seconds_bucket{le="1.0"}  9999
http_request_duration_seconds_bucket{le="+Inf"} 10000
http_request_duration_seconds_count            10000
http_request_duration_seconds_sum             523.4  # total seconds

# PromQL: 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Summary:** Pre-calculated quantiles (client-side). Less flexible than histograms (can't aggregate across instances).

### Prometheus Architecture

```
┌───────────────────┐    ┌─────────────────┐
│  Target Services  │◄───│   Prometheus    │
│  (expose /metrics)│    │  (scrape every  │
│                   │    │   15 seconds)   │
│  /metrics:        │    └────────┬────────┘
│  http_requests... │             │ PromQL queries
│  memory_used...   │    ┌────────▼────────┐
│  db_connections...|    │    Grafana      │
└───────────────────┘    │  (dashboards)  │
                         └────────┬────────┘
                                  │ alerts
                         ┌────────▼────────┐
                         │ Alertmanager   │
                         │ (PagerDuty,    │
                         │  Slack, email) │
                         └─────────────────┘
```

**Prometheus scraping model:** Prometheus PULLS metrics from targets by HTTP GET to `/metrics`. This is opposite of push-based systems (StatsD, InfluxDB Telegraf).

**Advantages of pull:** Prometheus knows exactly when a target is down (no metrics = down). Can discover targets dynamically (Kubernetes service discovery).

### Instrumenting Your App

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

ACTIVE_CONNECTIONS = Gauge('active_connections', 'Active WebSocket connections')

# Use in request handler
def handle_request(method, endpoint):
    start = time.time()
    ACTIVE_CONNECTIONS.inc()
    try:
        response = process_request(method, endpoint)
        REQUEST_COUNT.labels(method=method, endpoint=endpoint, status_code=200).inc()
        return response
    except Exception as e:
        REQUEST_COUNT.labels(method=method, endpoint=endpoint, status_code=500).inc()
        raise
    finally:
        duration = time.time() - start
        REQUEST_LATENCY.labels(method=method, endpoint=endpoint).observe(duration)
        ACTIVE_CONNECTIONS.dec()

# Expose /metrics endpoint
start_http_server(8000)
```

### Key Cardinality Warning

**Cardinality** = number of unique label combinations. High cardinality kills Prometheus.

```python
# ❌ DANGEROUS: user_id as a label (millions of unique values!)
REQUEST_COUNT = Counter('requests', 'Requests', ['user_id', 'endpoint'])
# user_id can be millions → millions of time series → OOM

# ✅ SAFE: Use aggregatable labels only
REQUEST_COUNT = Counter('requests', 'Requests', ['endpoint', 'status_code'])
# endpoint: ~50 values, status_code: ~10 → 500 time series (fine)

# For per-user analytics: use logs or a purpose-built system (ClickHouse)
```

---

## 🎯 SLIs, SLOs, SLAs & Error Budgets

### SLI (Service Level Indicator)

A quantitative measure of service behavior.

```
Common SLIs:
  Availability: (successful_requests / total_requests) × 100%
  Latency: % of requests served within X ms (e.g., p99 < 500ms)
  Error rate: (error_requests / total_requests) × 100%
  Throughput: requests per second

Example SLI measurement:
  Availability = (requests returning 2xx) / (all requests) × 100
              = 99,950 / 100,000 × 100 = 99.95%
```

### SLO (Service Level Objective)

A target value or range for an SLI. Your internal goal.

```
SLO: 99.9% of requests return 2xx over a rolling 30-day window
SLO: p99 latency < 500ms
SLO: p50 latency < 100ms
```

### SLA (Service Level Agreement)

An external, contractual commitment with consequences for violation (financial penalties, refunds).

```
SLA (contractual): 99.9% uptime guaranteed monthly
  If violated: customer receives service credits

Note: SLO is always stricter than SLA internally
  SLA: 99.9% → SLO: 99.95% (buffer so you never breach SLA)
```

### Error Budget

The allowed amount of downtime or errors implied by the SLO.

```
SLO: 99.9% availability over 30 days

Available error budget:
  30 days × 24 hours × 60 min = 43,200 minutes/month
  43,200 × (1 - 0.999) = 43.2 minutes of downtime allowed

Error budget burn rate:
  If your downtime rate is 2× normal → burn rate = 2
  At burn rate 2: budget exhausted in 15 days (instead of 30)

Alert: if burn rate > 14.4× over 1 hour → page on-call (will exhaust weekly budget in 1 hour)
Alert: if burn rate > 6× over 6 hours → ticket (will exhaust monthly budget in 5 days)
```

**The power of error budgets:** They make the reliability vs velocity tradeoff explicit and measurable. If you have budget left → ship features faster. If budget is nearly exhausted → freeze releases, focus on reliability.

---

## 📝 Logging: ELK, Loki & Structured Logs

### Structured Logging

Unstructured logs are hard to query. Structured logs (JSON) enable powerful filtering.

```python
import structlog

log = structlog.get_logger()

# ❌ Unstructured (hard to parse):
print(f"User 123 placed order 456 for $99.99 at 2024-01-15 10:30:00")

# ✅ Structured (queryable):
log.info(
    "order_placed",
    user_id=123,
    order_id=456,
    amount=99.99,
    currency="USD",
    items_count=3,
    payment_method="credit_card",
    request_id="req_abc123",   # ← trace correlation ID
    duration_ms=45
)
# Output:
# {"event": "order_placed", "user_id": 123, "order_id": 456,
#  "amount": 99.99, "timestamp": "2024-01-15T10:30:00Z", ...}
```

**Log Levels:**
| Level | Use For |
|---|---|
| DEBUG | Detailed diagnostics (dev/staging only) |
| INFO | Normal operations, key business events |
| WARN | Unexpected but handled; might indicate an issue |
| ERROR | Request failed, error handled, action needed |
| FATAL | System cannot continue, imminent crash |

**In production:** INFO and above. DEBUG is too verbose — can easily generate GB/hour.

### Centralized Log Aggregation

All service logs shipped to a central store for search, alerting, and retention.

**ELK Stack:**
```
Services → Filebeat/Fluentd (log shipper) → Logstash (parse/transform) → Elasticsearch (index/store) → Kibana (visualize/search)
```

**Grafana Loki (cheaper alternative):**
```
Services → Promtail (log shipper) → Loki (index only labels, not full text) → Grafana (query)

Loki's advantage: Only indexes labels (like Prometheus), NOT the full log content
→ Much cheaper storage (store raw logs in S3, index only labels)
→ Trade-off: full-text search is slower
```

**Log aggregation patterns in Kubernetes:**
- **DaemonSet (Recommended):** One log agent (Fluentd/Promtail) per node, reads all pod logs from `/var/log/pods/`
- **Sidecar:** One log agent per pod (more resource overhead but pod-level control)
- **Application-level:** App streams to stdout → Kubernetes captures → DaemonSet ships

### Log Sampling

At 100K+ requests/second, logging every request at DEBUG level produces TBs of data daily. Sampling reduces volume.

```python
import random

def should_log_request(request):
    # Always log errors
    if request.status_code >= 500:
        return True
    # Always log slow requests
    if request.duration_ms > 1000:
        return True
    # Sample 1% of normal requests
    return random.random() < 0.01

# With trace sampling: sample entire traces (keep all spans for sampled traces)
# Head-based sampling: decide at trace start (propagate decision)
# Tail-based sampling: decide at trace end (better — can choose to keep slow/error traces)
```

---

## 🔍 Distributed Tracing: OpenTelemetry & Jaeger

### Core Concepts

**Trace:** The complete journey of a single request through your system. Has a globally unique `trace_id`.

**Span:** A single operation within a trace (one service call, one DB query). Has a `span_id` and `parent_span_id`.

**Context propagation:** The trace context (trace_id, span_id) is passed through every service via HTTP headers:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^  ^                                ^               ^
             v  trace_id (16 bytes hex)          span_id         flags
```

### OpenTelemetry (OTel)

The industry-standard instrumentation framework. Vendor-neutral — instrument once, export to any backend (Jaeger, Zipkin, Datadog, Honeycomb).

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Setup (done once at startup)
provider = TracerProvider()
exporter = OTLPSpanExporter(endpoint="http://otel-collector:4317")
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# Auto-instrument common libraries (zero code change to existing HTTP/DB calls!)
RequestsInstrumentor().instrument()  # All requests calls auto-traced
SQLAlchemyInstrumentor().instrument()  # All DB queries auto-traced

# Manual instrumentation for business logic
tracer = trace.get_tracer("order-service")

def process_order(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        
        with tracer.start_as_current_span("validate_inventory"):
            items = check_inventory(order_id)
            span.set_attribute("inventory.items_count", len(items))
        
        with tracer.start_as_current_span("charge_payment") as payment_span:
            result = charge_payment(order_id)
            payment_span.set_attribute("payment.amount", result.amount)
            payment_span.set_attribute("payment.method", result.method)
```

### Jaeger — Visualizing Traces

Jaeger (open source by Uber) provides a UI for querying and visualizing traces.

```
Jaeger UI shows:
  Trace abc123 (total: 350ms)
  ├── API Gateway              [0ms - 5ms]      5ms
  ├── Order Service            [5ms - 120ms]  115ms
  │   ├── DB: SELECT order    [5ms - 25ms]    20ms
  │   ├── Inventory RPC call  [25ms - 80ms]   55ms ← ⚠️ slow
  │   │   └── DB: SELECT stock [26ms - 75ms]  49ms ← root cause
  │   └── Kafka publish       [80ms - 85ms]    5ms
  └── Return to client        [120ms - 125ms]  5ms

Immediately shows: Inventory DB query is the bottleneck
Solution: Add index on stock.product_id
```

### Tail-Based Sampling

Don't sample randomly — keep traces that are interesting (slow, errored).

```yaml
# OpenTelemetry Collector tail sampling config
processors:
  tail_sampling:
    decision_wait: 10s  # Wait 10s to collect all spans before deciding
    policies:
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]  # Always keep error traces
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000  # Always keep traces > 1 second
      - name: sample-rest
        type: probabilistic
        probabilistic:
          sampling_percentage: 1  # 1% of everything else
```

---

## 🚨 Alerting Best Practices

### Reducing Alert Fatigue

The #1 observability problem in production: too many alerts → engineers ignore them → real incidents missed.

**Properties of good alerts:**
1. **Actionable:** An alert fires → engineer knows what to do
2. **Relevant:** Alert only when user is impacted or will be imminently
3. **Accurate:** Low false positive rate (should cry wolf rarely)
4. **Timely:** Fires soon enough to prevent cascading impact

**Alert on symptoms, not causes:**
```
❌ Alert: "CPU > 90%" 
   (CPU might be high because of useful work — users aren't impacted yet)

✅ Alert: "Error rate > 1% for 5 minutes"
   (Users ARE experiencing errors right now)

✅ Alert: "p99 latency > 2s for 5 minutes"
   (Users ARE experiencing slow responses)

❌ Alert: "Connection pool utilization > 80%"
   (Might be fine; alert when connections are actually failing)
```

**Severity levels:**
```
P1 (Page immediately): System down or severe degradation. Users impacted NOW.
  Example: error rate > 5%, complete service outage

P2 (Page during business hours): Significant but not critical.
  Example: error rate 1-5%, latency degraded but not terrible

P3 (Ticket): Investigate soon but no immediate user impact.
  Example: slow error budget burn rate, disk filling up

P4 (Log only): Worth knowing but no action needed immediately.
  Example: deprecated API endpoint being called
```

**Alert on error budget burn rate** (Google SRE approach):
```
Alert 1 (urgent): burn rate > 14.4× for 1 hour
  → At this rate, monthly error budget exhausted in 2 hours
  → Page on-call immediately

Alert 2 (warning): burn rate > 6× for 6 hours  
  → At this rate, monthly budget exhausted in 5 days
  → Create ticket, engineer investigates

Alert 3 (info): burn rate > 1× for 3 days
  → Just tracking, no emergency
```

### Runbooks

Every alert should link to a runbook — a step-by-step guide for the on-call engineer.

```markdown
## Alert: High Error Rate on /checkout

### Impact
Users cannot complete purchases. Revenue impact: ~$50K/minute.

### Immediate Checks
1. Check `kubectl get pods -n production | grep checkout` — any pods CrashLooping?
2. Check Jaeger for recent traces — which service/DB is failing?
3. Check `kubectl logs -n production deploy/checkout-service --tail=100`

### Common Causes
1. **DB connection exhaustion** → Check PgBouncer metrics → Restart if stuck
2. **Payment service timeout** → Check payment service health endpoint
3. **Memory OOM** → Check pod memory limits, scale up if needed

### Escalation
If not resolved in 15 minutes → escalate to #incident-response Slack + page payment-team
```

---

## 🐵 Chaos Engineering

**Chaos engineering** is the practice of deliberately introducing failures into a production system to build confidence in its resilience.

> "We break things on purpose to learn how they break before they break when we don't want them to."

### The Hypothesis-Driven Approach

```
1. Define "steady state" behavior (metrics during normal operation)
2. Hypothesize: "If we inject [failure], the system will maintain [steady state]"
3. Introduce failure in production (or staging)
4. Observe: Did steady state hold?
5. Fix any discovered weaknesses
6. Repeat
```

### Types of Chaos Experiments

**Infrastructure failures:**
- Kill random pods (Chaos Monkey)
- Terminate entire availability zones
- Kill database primary node
- Fill disk to 100%

**Network failures:**
- Inject 500ms latency on inter-service communication
- Drop 10% of network packets
- DNS resolution failures
- Network partition between services

**Application failures:**
- Memory pressure (fill RAM)
- CPU stress
- Thread pool exhaustion
- Clock skew

### Tools

**Chaos Monkey (Netflix):** Randomly terminates EC2 instances (or pods) in production during business hours.

**Chaos Toolkit:** Open source framework for defining and running chaos experiments.

**AWS Fault Injection Simulator (FIS):** Managed chaos for AWS services.

**Gremlin:** Commercial chaos platform with pre-built attack library.

```yaml
# Chaos Toolkit experiment: test circuit breaker
{
  "title": "Circuit breaker activates on payment service failure",
  "description": "When payment service returns 503, checkout should degrade gracefully",
  "steady-state-hypothesis": {
    "title": "Checkout API is responding",
    "probes": [{
      "type": "http",
      "url": "https://api.example.com/checkout/health",
      "expected_status": 200
    }]
  },
  "method": [{
    "type": "action",
    "name": "inject-payment-service-failure",
    "provider": {
      "type": "process",
      "path": "kubectl",
      "arguments": "patch deployment payment-service -p '{\"spec\":{\"replicas\":0}}'"
    }
  }],
  "rollbacks": [{
    "type": "action",
    "name": "restore-payment-service",
    "provider": {
      "type": "process",
      "path": "kubectl",
      "arguments": "patch deployment payment-service -p '{\"spec\":{\"replicas\":3}}'"
    }
  }]
}
```

---

## 🛠️ APM Tools

**APM (Application Performance Monitoring)** tools combine all three pillars with additional features.

| Tool | Strengths | Pricing |
|---|---|---|
| **Datadog** | Excellent UX, comprehensive (metrics+logs+traces+profiling), easy setup | Expensive |
| **New Relic** | Good all-in-one, free tier available | Moderate |
| **Dynatrace** | AI-powered root cause analysis, auto-discovery | Expensive |
| **Grafana Stack** | Open source (Prometheus+Loki+Tempo+Grafana), very cost-effective | Free (self-hosted) |
| **Honeycomb** | Excellent for high-cardinality event data and exploration | Moderate |
| **AWS CloudWatch** | Tightly integrated with AWS, managed | Pay per use |

**Grafana Stack (Open Source):**
```
Metrics:  Prometheus → Grafana
Logs:     Loki → Grafana  
Traces:   Tempo → Grafana
Dashboards: Grafana

Single pane of glass: Grafana correlates all three
"Click on a spike in metrics → jump to logs at that time → see traces"
```

---

## 📱 Real-Life Scenario: Finding a Slow DB Query

**The situation:** Users report checkout is "slow sometimes." p99 latency is 2000ms; normally 200ms.

**Step 1: Metrics alert**
```
Alert fires: p99 latency > 1000ms for 10 minutes on POST /checkout
Grafana shows: latency spike started 45 minutes ago
Error rate: still < 0.1% (requests are slow, not failing)
```

**Step 2: Check traces in Jaeger**
```
Query: service=checkout-service, min_duration=1000ms, last 1 hour
Results: 847 slow traces

Open trace abc123 (2100ms total):
  checkout-service     [0-50ms]    50ms
  ├── auth validation  [0-10ms]    10ms
  ├── inventory check  [10-15ms]   5ms
  ├── create_order DB  [15-2090ms] 2075ms ← !!!
  └── kafka publish    [2090-2100] 10ms

Slow DB span attributes:
  db.type: postgresql
  db.statement: "INSERT INTO orders ... RETURNING *"
  db.rows_affected: 1
  wait_event: "Lock"  ← Lock wait!
```

**Step 3: Check logs**
```
Loki query: {service="checkout-service"} |= "lock" since=45m

Logs show:
  2024-01-15 10:15:23 WARN db_query_slow order_id=456 duration_ms=1820 
    lock_wait_ms=1750 query="INSERT INTO orders"
  2024-01-15 10:15:45 WARN db_query_slow order_id=789 duration_ms=2100 
    lock_wait_ms=2050 query="INSERT INTO orders"
```

**Step 4: Investigate**
```
PostgreSQL pg_locks query shows:
  A long-running transaction from a data export job
  Holding locks on the orders table
  Export job started 45 minutes ago!
```

**Step 5: Fix**
```
Killed the export job → lock released → latency immediately dropped to 150ms
Root cause: export job using SELECT FOR SHARE on orders table
Fix: export from read replica, not primary; use SERIALIZABLE isolation with retries
```

**Total time from alert to resolution: 23 minutes.** Without distributed tracing, this would have been hours of guesswork.

---

## 🔥 Big Tech Story: Netflix Chaos Monkey

### The Background (2010-2011)

Netflix had just moved its infrastructure from data centers to AWS (one of the earliest major companies to do so). They were deeply worried: what if an AWS instance failed? What if an availability zone went down?

Traditional approach: test in staging, then hope production is fine.

Netflix engineer **Cory Bennett** had a different idea: **break production on purpose, regularly, during business hours, to force engineers to build resilient systems.**

### The Birth of Chaos Monkey (2011)

The first Chaos Monkey was a simple script that randomly terminated EC2 instances within Netflix's production ASGs (Auto Scaling Groups). It ran during business hours only (engineers needed to be awake to respond and learn from failures).

The philosophy:
> "If we inject failures randomly, engineers will build systems that handle them automatically. If we only simulate failures in staging, engineers will 'fix' them only for staging."

**Early resistance:** Many engineers were horrified. "You want to randomly kill production servers?!"

**Netflix's response:** "If you can't handle one instance dying, you can't call yourself resilient. And one instance WILL die in production — we just won't control when."

The Chaos Monkey forced every team to:
- Implement proper health checks
- Implement auto-restart and circuit breakers
- Test failover paths actually work
- Remove any single points of failure

### Expansion: The Simian Army (2012)

Netflix expanded chaos engineering into a suite of tools they called the "Simian Army":

| Tool | What It Breaks |
|---|---|
| **Chaos Monkey** | Kills random EC2 instances |
| **Chaos Gorilla** | Takes down an entire AWS Availability Zone |
| **Chaos Kong** | Simulates an entire AWS Region failure |
| **Latency Monkey** | Injects artificial latency into service calls |
| **Doctor Monkey** | Identifies unhealthy instances and removes them |
| **Janitor Monkey** | Cleans up unused cloud resources (cost savings!) |
| **Security Monkey** | Finds security policy violations |
| **Conformity Monkey** | Ensures instances follow best practices |

**Chaos Gorilla** was particularly impactful: when Netflix killed an entire AZ, they discovered several services that had "multi-AZ" in their names but still failed because of dependency on single-AZ resources.

### The Results

After running Chaos Monkey for a year, Netflix engineers became desensitized to instance failures — **they stopped being scary because they'd been handled so many times.** The automatic recovery systems just worked.

When AWS itself had major outages (us-east-1 incidents in 2011, 2012, etc.), Netflix largely stayed up while other companies went down. Netflix had proven their chaos tolerance on real production systems.

### Open Sourced (2012)

Netflix open-sourced Chaos Monkey in 2012. The concept of chaos engineering spread to the entire industry.

**Principles of Chaos Engineering** (published by Netflix, 2017):
1. Start by defining "steady state" — the measurable, normal behavior
2. Hypothesize that steady state will hold in both control and experimental groups
3. Introduce variables that reflect real-world events (instance failure, network partition)
4. Try to disprove the hypothesis — look for weaknesses
5. Run experiments in production (staging gives false confidence)
6. Automate experiments to run continuously

Today, companies like Amazon, Google, Microsoft, and LinkedIn have formal chaos engineering programs. The concept has evolved from "kill random servers" to sophisticated fault injection frameworks.

**The lesson:** The only way to know your system handles failures is to test it handling failures. Paper designs and staging environments are not enough. The production environment always has surprises.

---

## ⚖️ Trade-offs & Best Practices

### Observability Trade-offs

| Concern | Trade-off |
|---|---|
| **Cost** | Logs/traces are expensive to store. Sampling reduces cost but may miss rare events. |
| **Cardinality** | High-cardinality metrics (user_id labels) crash Prometheus. Use logs for per-user data. |
| **Performance overhead** | Tracing adds 1-5% CPU overhead per service. Sampling mitigates this. |
| **Alert fatigue** | Too many alerts → ignored alerts → real incidents missed. |
| **Sampling vs completeness** | Sample 1% of traces → miss rare bugs. Sample 100% → too expensive. |

### Best Practices

1. **Start with the RED method:** Request rate, Error rate, Duration (latency). These three metrics answer "is the service healthy?" for every service.

2. **Add the USE method for infrastructure:** Utilization, Saturation, Errors for every CPU/memory/disk/network resource.

3. **Instrument before you go to production.** Retrofitting observability into production services is extremely hard. Make it part of every service's initial setup.

4. **Correlate all three pillars.** Every log entry should include the `trace_id`. Every trace span should include the `request_id`. Grafana/Datadog use these to jump between pillars.

5. **Alert on error budget burn rate** (not just raw thresholds) for smarter alerting.

6. **Write runbooks for every alert.** If an alert doesn't have a runbook, it shouldn't exist.

7. **Do chaos experiments regularly** — at least quarterly for critical paths. Start with small blast radius (single container) and expand.

8. **Sample traces, always keep error traces.** 1% sampling with tail-based sampling (keep slow/error traces) gives 90% of the value at 1% of the cost.

9. **Set log retention policies.** Production logs: 30 days hot, 90 days cold (S3). Security audit logs: 1 year (compliance). Don't keep everything forever.

10. **Practice incident response.** Run quarterly "game days" where you simulate incidents and practice using your observability tools to diagnose them.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **Three Pillars** | Metrics = health check; Logs = details; Traces = request path. Need all three. |
| **Cardinality** | Never use high-cardinality values (user_id) as Prometheus labels |
| **SLO + Error Budget** | Makes the reliability vs velocity tradeoff explicit and measurable |
| **Structured Logs** | JSON logs are queryable; unstructured logs are grep-able at best |
| **OpenTelemetry** | Instrument once, export to any backend. The industry standard. |
| **Tail-based sampling** | Keep interesting traces (slow, errored); sample the rest |
| **Alert fatigue** | Alert on symptoms not causes; error budget burn rate over raw thresholds |
| **Chaos engineering** | Break production on purpose to build resilient systems |
| **Netflix's lesson** | If a failure isn't tested, it will surprise you. Test failures constantly. |

---

*Next: [14_Data_Pipelines.md](14_Data_Pipelines.md) — Batch vs stream processing, Spark, Flink, data lakehouses, and Uber's real-time analytics platform.*
