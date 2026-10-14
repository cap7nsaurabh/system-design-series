# 🏥 High Availability & Disaster Recovery

> *"It's not if you'll have an outage, but when. And when you do, your preparedness is measured in seconds, not plans."*

---

## 📋 Table of Contents
1. [Key Metrics: RTO, RPO, RTA, RLO](#metrics)
2. [Availability Nines](#availability-nines)
3. [Single Points of Failure (SPOF)](#spof)
4. [Active-Passive vs Active-Active](#active-passive-vs-active-active)
5. [Multi-Region Architectures](#multi-region)
6. [Database HA Strategies](#database-ha)
7. [Backup Strategies](#backup-strategies)
8. [Failover Automation](#failover-automation)
9. [Graceful Degradation](#graceful-degradation)
10. [Real-Life Scenario](#real-life-scenario)
11. [🔥 Big Tech Story: AWS us-east-1 Outages](#big-tech-story)
12. [Disaster Recovery Runbook](#dr-runbook)
13. [Trade-offs & Best Practices](#trade-offs)
14. [Key Takeaways](#key-takeaways)

---

## 📏 Key Metrics: RTO, RPO, RTA, RLO

### RTO (Recovery Time Objective)

**How long can your system be down?**

The maximum acceptable time between failure and full service restoration.

```
Outage starts: 10:00 AM
System fully restored: 10:15 AM
Recovery time: 15 minutes

If RTO = 30 minutes → ACCEPTABLE
If RTO = 10 minutes → VIOLATED (took 15 min, exceeded 10 min target)
```

### RPO (Recovery Point Objective)

**How much data can you afford to lose?**

The maximum acceptable amount of data loss measured in time. How far back can you restore to?

```
Last backup: 2:00 AM
Failure: 10:00 AM
Data since last backup: 8 hours of transactions

If RPO = 24 hours → ACCEPTABLE (only lost 8 hours)
If RPO = 1 hour → VIOLATED (lost 8 hours, exceeded 1 hour target)
```

### Business Tier vs Technology Configuration

| Business Tier | RTO | RPO | Technology |
|---|---|---|---|
| Mission-critical (payments, trading) | < 1 min | 0 (zero data loss) | Synchronous multi-region replication, active-active |
| Business-critical (checkout, ordering) | < 15 min | < 5 min | Hot standby, automated failover |
| Important (analytics, reports) | < 4 hours | < 1 hour | Warm standby, hourly snapshots |
| Standard (internal tools, dev) | < 24 hours | < 24 hours | Backup restore, daily snapshots |
| Development | Best effort | Best effort | None required |

### RTA (Recovery Time Actual)

The actual time it took to recover. Tracked against RTO to measure performance.

### RLO (Recovery Level Objective)

What percentage of full functionality must be restored. Sometimes 80% is acceptable (non-critical features can wait).

---

## ⏱️ Availability Nines

"Nines" refers to the number of 9s in the availability percentage.

| Availability | Annual Downtime | Monthly Downtime | Use Case |
|---|---|---|---|
| 99% (two 9s) | 87.6 hours | 7.3 hours | Internal tools, dev environments |
| 99.9% (three 9s) | 8.76 hours | 43.8 minutes | Standard production services |
| 99.95% | 4.38 hours | 21.9 minutes | Premium production services |
| 99.99% (four 9s) | 52.6 minutes | 4.38 minutes | High-value business services |
| 99.999% (five 9s) | 5.26 minutes | 26.3 seconds | Telecom, financial systems |
| 99.9999% (six 9s) | 31.6 seconds | 2.6 seconds | Aircraft, medical devices |

**Going from 99.9% to 99.99% is not 0.09% better — it's 10x better.** The cost difference is also non-linear.

**Practical note:** AWS's SLAs for most services are 99.9-99.99%. Building 99.999% on top of 99.99% infrastructure requires active-active multi-region with no shared dependencies.

**Availability of composed services:**
```
If Service A = 99.99% and Service B = 99.99%, and both must be up:
  Combined availability = 0.9999 × 0.9999 = 99.98%

If Service A = 99.99% or Service B = 99.99% (A fails over to B):
  Combined availability = 1 - (0.0001 × 0.0001) = 99.999999%
  
→ Redundancy (OR logic) dramatically improves availability
→ Dependencies (AND logic) reduce availability
```

---

## 🎯 Single Points of Failure (SPOF)

A SPOF is any component whose failure brings down the entire system.

### Common SPOFs and Their Solutions

| SPOF | Solution |
|---|---|
| Single load balancer | Active-passive LB pair, or cloud-managed LB (AWS ALB) |
| Single app server | Horizontal scaling + auto-scaling group |
| Single database | Primary + standby replica with automated failover |
| Single availability zone | Deploy across multiple AZs |
| Single region | Multi-region active-active or active-passive |
| Single DNS provider | Multi-DNS (Route 53 + Cloudflare), low TTLs |
| Single CDN | Multi-CDN with DNS failover |
| Single cloud provider | Multi-cloud (rare, expensive, complex) |
| Single internet uplink | Multiple ISP connections with BGP failover |
| NAT Gateway | One NAT per AZ, not one per region |

**SPOF audit process:**
1. Draw the system architecture diagram
2. For every component, ask: "What happens if this fails?"
3. If the answer is "the system goes down" → it's a SPOF → add redundancy

---

## 🔄 Active-Passive vs Active-Active

### Active-Passive

One node is active (handling traffic). Another is passive (standby, receives replicated data but serves no traffic). On failure, traffic switches to passive.

```
Normal:
  Load Balancer → Active Server (serving traffic)
                  Passive Server (idle, receiving replication)

Failure:
  Active Server fails
  Health check detects failure (10-30 seconds)
  Load Balancer → Passive Server (now serving traffic)
  
Recovery time: 30 seconds - 5 minutes (depending on failover mechanism)
Passive server may have brief "warm-up" period
```

**Pros:**
- Simple
- No data conflicts (only one writer)
- Passive is always up-to-date (if synchronous replication)

**Cons:**
- Wasted resource (passive server idle)
- Recovery time = detection time + failover time
- Passive may lag if async replication

### Active-Active

Both (or all) nodes serve traffic simultaneously. If one fails, others absorb its traffic.

```
Normal:
  Load Balancer → Active Server 1 (50% of traffic)
               → Active Server 2 (50% of traffic)

Failure of Server 1:
  Load Balancer detects unhealthy
  Load Balancer → Active Server 2 (100% of traffic)
  
Recovery time: Near-zero (load balancer reroutes in seconds)
```

**Pros:**
- No wasted resources (all servers active)
- Near-zero failover time
- Can handle higher traffic (horizontal capacity)

**Cons:**
- More complex (distributed state coordination)
- Conflict resolution needed for stateful data (write-write conflicts)

### When to Use Which

| Scenario | Recommendation |
|---|---|
| Database primary/standby | Active-Passive (one writer avoids conflicts) |
| Stateless application servers | Active-Active (simple, all identical) |
| Multi-region databases | Active-Active only possible with global locking (Spanner) or conflict resolution (DynamoDB Global Tables) |
| Cache (Redis) | Active-Passive for simplicity; Redis Cluster for Active-Active |

---

## 🌍 Multi-Region Architectures

### Why Multi-Region?

- **Availability:** Region failure is rare (~0.01%) but catastrophic without multi-region
- **Latency:** Serve users from the nearest region (100ms vs 300ms)
- **Data residency:** GDPR requires EU data in EU; some countries require local data

### Active-Passive Multi-Region

```
Primary Region (us-east-1): ACTIVE — serves all traffic
Secondary Region (us-west-2): PASSIVE — receives replication

Normal:  All traffic → us-east-1
Failure: Route 53 health check detects us-east-1 down
         DNS failover → us-west-2 (TTL 60s → up to 60s for propagation)
         us-west-2 promotes to primary

RTO: 5-15 minutes (DNS propagation + DB promotion)
RPO: seconds to minutes (depends on replication lag)
Cost: ~2x (secondary sits mostly idle)
```

**AWS Aurora Global Database:**
```
Primary Region: Read/Write
Secondary Region: Read-only replicas (lag < 1 second globally)

On failover:
  Aurora automatically promotes secondary to primary
  < 1 minute RTO
  Near-zero RPO (async replication, typically < 1 second lag)
```

### Active-Active Multi-Region

```
us-east-1: ACTIVE — serves 40% of traffic (North/South America)
eu-west-1: ACTIVE — serves 40% of traffic (Europe, Africa)
ap-southeast-1: ACTIVE — serves 20% of traffic (Asia-Pacific)

GeoDNS routes each user to nearest region

Failure of eu-west-1:
  Route 53 health check detects failure
  EU users routed to us-east-1 or ap-southeast-1
  Other regions unaffected
  
RTO: < 1 minute (DNS failover)
RPO: 0 for reads (served from other regions); seconds for writes (replication lag)
```

**The hard part: distributed writes.** With active-active multi-region, writes can happen in any region. How do they sync?

**Option 1: Global database with synchronous replication (Google Spanner)**
```
Write in us-east-1 → synchronously replicated to eu-west-1 before ack
Latency penalty: ~100ms added to every write (cross-region RTT)
```

**Option 2: Async replication + conflict resolution (DynamoDB Global Tables)**
```
Write in us-east-1 → replicated async to eu-west-1 (~1 second)
Concurrent writes to same item in both regions → last writer wins
Acceptable for: user preferences, session data, shopping cart
Not acceptable for: financial transactions, inventory (use strong consistency)
```

**Option 3: Read-your-writes with shard-by-region**
```
User 123 (US) → always writes to us-east-1 shard
User 456 (EU) → always writes to eu-west-1 shard
Reads: Reads from local region first, fallback to others
No write conflicts! (Each user's data in one home region)
Works for: social apps, user profiles
```

---

## 🗃️ Database HA Strategies

### PostgreSQL High Availability

**Streaming Replication (built-in):**
```
Primary → WAL stream → Standby (streaming, < 1 second lag)

Synchronous standby:
  COMMIT only after at least 1 standby has flushed WAL
  Zero data loss (RPO = 0)
  Write latency +10-50ms (cross-DC roundtrip)

Asynchronous standby:
  COMMIT after primary writes WAL (standby may lag)
  Some data loss possible (RPO = replication lag)
  No write latency impact
```

**Patroni (PostgreSQL HA using etcd/ZooKeeper):**
```
Patroni monitors PostgreSQL cluster via etcd (Raft-based)
If primary is down for 30 seconds → Patroni promotes standby
Automatic failover with < 60s RTO
Used by: Zalando, GitLab, others
```

**AWS RDS Multi-AZ:**
```
Synchronous standby in different AZ
Automatic failover: 30-120 seconds RTO
Zero data loss (synchronous replication)
1.8x more expensive (you pay for standby)
```

### Cassandra HA

Cassandra is designed for HA from the ground up. No primary/standby — all nodes are peers.

```
Replication factor 3, across 3 AZs:
  Data written to 3 nodes (one per AZ)
  Consistency level QUORUM: wait for 2/3 nodes to ack
  
  AZ failure: 1 node down, 2 remaining → still have quorum
  Up to 1 AZ can fail without data loss or unavailability
```

### Redis HA

**Redis Sentinel (simple HA):**
```
Primary ← Replica 1
        ← Replica 2
        
Sentinel monitors (3 Sentinel instances for quorum)
Primary down → Sentinels elect leader → promote Replica 1 → repoint clients
Failover: 15-30 seconds
```

**Redis Cluster (sharding + HA):**
```
6 nodes: 3 primary + 3 replica
  Primary 1 (slots 0-5460) + Replica
  Primary 2 (slots 5461-10922) + Replica
  Primary 3 (slots 10923-16383) + Replica

Failure of Primary 1 → its Replica auto-promoted
Remaining cluster: functional (some slots on new primary)
```

---

## 💾 Backup Strategies

### 3-2-1 Backup Rule

- **3** copies of data
- **2** different storage media types
- **1** copy offsite (different geographic location, different cloud)

```
Production data: RDS PostgreSQL (primary + standby)
Daily snapshot: RDS automated backup to S3 (same region)
Weekly export: S3 cross-region replication (different region)

This satisfies 3-2-1:
  3 copies: RDS primary + RDS standby + S3 backup
  2 media: RDS (block) + S3 (object)
  1 offsite: S3 cross-region
```

### Database Backup Methods

| Method | Technology | RTO | RPO | Use Case |
|---|---|---|---|---|
| Full backup | pg_dump, mysqldump | Hours (restore time) | Since last backup | Small databases |
| Incremental backup | WAL archiving + base backup | Minutes-Hours | Near zero | Medium databases |
| Snapshot | EBS snapshot, RDS snapshot | Minutes | Since snapshot | AWS-native |
| Continuous replication | Streaming replication, DMS | Seconds | Seconds | Production HA |
| Point-in-time restore | WAL + base backup | Minutes | Any point in time | RDS PITR |

**AWS RDS Point-in-Time Recovery:**
```
RDS retains WAL logs for up to 35 days
You can restore to any second within that window

Example: "Restore my database to 10:22:47 AM on January 14th"
RDS restores base snapshot + replays WAL up to that exact second
```

### Testing Backups

**Critical rule: An untested backup is not a backup.**

```
Backup testing schedule:
  Daily: Automated restore to staging environment; verify app runs
  Monthly: Full DR drill - restore to isolated environment, test all critical paths
  Quarterly: Full production failover drill (during low-traffic period)
  
Metrics to track:
  Restore time (vs RTO target)
  Data integrity (vs RPO target)
  App functionality post-restore (% of features working)
```

The primary cause of failed DR is: "The backups are fine but we've never tried to restore from them."

---

## 🤖 Failover Automation

Manual failover takes too long (humans make mistakes under pressure, take 15+ minutes).

### Health Checks

**Shallow health check:** Is the process running?
```
GET /health → 200 OK
```

**Deep health check:** Can the service actually serve requests? Are all dependencies healthy?
```python
@app.route('/health/deep')
def deep_health():
    checks = {}
    
    # Check DB
    try:
        db.execute("SELECT 1")
        checks['database'] = 'ok'
    except Exception as e:
        checks['database'] = f'error: {e}'
        
    # Check Redis
    try:
        redis.ping()
        checks['cache'] = 'ok'
    except Exception as e:
        checks['cache'] = f'error: {e}'
    
    # Check downstream service
    try:
        response = requests.get('http://payment-service/health', timeout=2)
        checks['payment_service'] = 'ok' if response.status_code == 200 else 'degraded'
    except Exception:
        checks['payment_service'] = 'unreachable'
    
    status = 200 if all(v == 'ok' for v in checks.values()) else 503
    return jsonify({'status': checks}), status
```

### AWS Route 53 Health Check + DNS Failover

```
Route 53 monitors: https://primary.example.com/health every 10 seconds
If 3 consecutive failures → mark primary as unhealthy
DNS failover: CNAME primary.example.com → secondary.example.com

Failover time: 30 seconds (3 × 10s checks) + DNS TTL (30-60s)
Total: ~60-90 seconds
```

### Database Automated Failover

**AWS RDS Multi-AZ:**
```
RDS detects primary failure (network connectivity loss)
Automatic failover: promotes standby
DNS CNAME for endpoint updated automatically
Application reconnects to same endpoint → now hits standby (new primary)
Total RTO: 30-120 seconds
```

**Patroni for PostgreSQL:**
```
Primary disappears from etcd
Patroni candidates (standbys) compete to acquire leader key in etcd
Winner promotes itself to primary
Other standbys follow the new primary
HAProxy /primary health endpoint updates → load balancer sends writes to new primary
Total RTO: 15-60 seconds
```

---

## 🌊 Graceful Degradation

When a component fails, the system should degrade gracefully rather than fail completely.

### Circuit Breaker with Fallback

```python
class RecommendationService:
    def __init__(self):
        self.cb = CircuitBreaker(failure_threshold=5, timeout=30)
    
    def get_recommendations(self, user_id: str) -> List[Product]:
        try:
            # Circuit breaker protects call to ML recommendation service
            return self.cb.call(lambda: ml_service.recommend(user_id))
        except CircuitBreakerOpenError:
            # Circuit is open → ML service is down
            # Fallback: return popular products (pre-computed, cached)
            return self.get_popular_products()
    
    def get_popular_products(self) -> List[Product]:
        # Cached in Redis, always available
        return redis.get('popular_products') or []
```

### Feature Flags for Emergency Degradation

```python
# LaunchDarkly or homegrown feature flags
if feature_flag.is_enabled('recommendation_engine', user_id):
    recommendations = ml_service.recommend(user_id)
else:
    recommendations = popular_products  # Fallback

# During an outage: flip flag to disable recommendations for all users
# → Users see popular items (degraded but functional) rather than errors
# → ML service outage doesn't break the whole site
```

### Timeouts Everywhere

```python
# Every external call must have a timeout
result = requests.get(
    'http://payment-service/charge',
    timeout=(3.05, 10)  # (connect_timeout, read_timeout)
)
# If payment service is slow: fail fast rather than blocking thread forever
```

**Timeout budget:** `Total request budget` must be larger than the sum of all downstream timeouts. If you have a 2-second total timeout and call 3 services, each service timeout must be < 600ms (leaving buffer for overhead).

---

## 📱 Real-Life Scenario: E-commerce Multi-Region Setup

**The requirement:** Process orders globally with < 200ms latency worldwide. Zero data loss for payments. 99.99% availability.

**Architecture:**

```
                        GeoDNS (Route 53)
                              │
              ┌───────────────┼────────────────┐
              │               │                │
         us-east-1        eu-west-1       ap-southeast-1
         (Primary)         (Active)          (Active)
              │               │                │
        ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
        │ ALB+WAF   │   │ ALB+WAF   │   │ ALB+WAF   │
        └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
        │ App Services │ App Services │ App Services │
        │ (ECS)       │ (ECS)       │ (ECS)       │
        └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
              │               │                │
        Aurora Global Database (us-east-1 = read/write)
              │               │                │
        (writer)        (reader replica)  (reader replica)
                        < 1 second lag     < 1 second lag
```

**Write path:**
```
Order placed in EU → Payment must be strongly consistent
  Option A: Route all payment writes to us-east-1 (higher latency for EU)
  Option B: Saga pattern with compensation (optimistic, rollback if conflict)
  
Chosen: Route payments to us-east-1 (aurora global write endpoint)
         +100ms latency for EU payments (acceptable for financial accuracy)
         All other writes (session, cart, preferences) to local region replica
```

**Failure scenarios:**

```
Scenario 1: ap-southeast-1 goes down
  Route 53 health check fails
  Asian users routed to us-east-1 (higher latency but functional)
  Availability: 100% for other regions, degraded latency for APAC
  RTO: < 1 minute

Scenario 2: us-east-1 goes down (primary DB region)
  Aurora Global DB promotes eu-west-1 replica to read/write
  Route 53 reroutes traffic to eu-west-1 and ap-southeast-1
  Typical Aurora promotion time: < 1 minute
  Data loss: last ~1 second of transactions (async replication lag)
  
Scenario 3: Complete region + partial region failure
  If 2 of 3 regions fail: surviving region handles all traffic
  (Design capacity: each region can handle 2× normal load)
```

**Cost vs availability:**
- Single region: $10K/month → 99.9% availability
- Active-passive multi-region: $18K/month → 99.99%
- Active-active multi-region: $30K/month → 99.999%

For a payment processor doing $50M/month: 1 hour of downtime ≈ $70K lost revenue. The $20K/month difference for multi-region pays for itself after one prevented outage.

---

## 🔥 Big Tech Story: AWS us-east-1 Outages

### Why us-east-1 Is Special (And Why That's a Problem)

AWS's oldest region, `us-east-1` (Northern Virginia), was the first AWS region and hosts a disproportionate share of AWS infrastructure, including AWS's own control planes for many global services.

Many companies run exclusively in us-east-1 because:
- It was the default
- It has the most services
- They never thought about regional failure

The result: when us-east-1 has problems, a significant fraction of the internet is affected.

### The December 2021 Outage

On December 7, 2021, a major AWS outage started in us-east-1 and affected services globally. The root cause: an automated activity to expand capacity in us-east-1 triggered an unexpected behavior in the AWS network that caused widespread networking issues.

**What failed:**
- Amazon.com itself — degraded (their own warehouse picker apps ran on AWS)
- Disney+ — streaming disruptions
- Netflix — some regions affected
- DoorDash, Instacart — degraded/down
- Robinhood, Coinbase — trading disruptions
- Ring doorbells — couldn't connect (entire home security system dependent on single AWS region)
- McDonald's ordering kiosks — down
- Deliveroo, Tesco, ASOS — degraded

**Duration:** ~8 hours for full resolution.

**Estimated economic impact:** Hundreds of millions of dollars across affected companies.

### What Made It Worse

1. **AWS Console itself was down.** Engineers couldn't access the AWS Console (also running in us-east-1) to diagnose and fix their own infrastructure. The fireman's hose was on fire.

2. **Service dependencies were hidden.** Companies thought they were running in multiple regions, but their monitoring tools, alerting systems, or deployment pipelines were all running in us-east-1 and failed simultaneously.

3. **Lambda functions in all regions calling back to us-east-1.** Some customers' Lambda functions in us-west-2 were calling services in us-east-1 for data, creating invisible inter-region dependencies.

4. **AWS Cognito (auth) had issues.** Many apps couldn't even authenticate users because their auth provider (Cognito) ran in us-east-1.

### Previous us-east-1 Major Outages

- **July 2012:** EBS failures in us-east-1 took down Netflix, Instagram, Pinterest
- **October 2012:** ELB/EC2 issues took down Reddit, Heroku, and others
- **April 2017:** S3 us-east-1 had a 4-hour outage due to a typo (an engineer ran a command with an unintended large scope — decommissioned too many S3 components)
- **November 2020:** Kinesis issues in us-east-1 cascaded to cause widespread issues (CloudWatch, Cognito, EventBridge all depend on Kinesis)
- **December 2021:** (described above)

### The Kinesis → Everything Cascade (2020)

The November 2020 outage is a masterclass in cascade failures.

1. AWS added new capacity to Kinesis in us-east-1 (more servers)
2. The increased server count exceeded the limit for a feature in Kinesis's control plane (a socket limit issue that had never been triggered before)
3. Kinesis started having errors

So far: just a Kinesis issue. Why did everything else fail?

4. **AWS CloudWatch** uses Kinesis internally. CloudWatch started failing.
5. **Amazon Cognito** uses CloudWatch. Cognito authentication started failing.
6. Apps that use Cognito for user auth → users can't log in.
7. AWS Lambda uses CloudWatch for logging. Lambda function invocations started failing because they couldn't write logs.
8. Applications using Lambda → broken.
9. AWS EventBridge uses Kinesis. Automated workflows broke.

A capacity expansion bug in one service caused cascading failures across half of AWS's service catalog. The fundamental cause: too many services depending on Kinesis (and therefore, on its health).

### Lessons for System Architects

1. **Never run in a single region.** us-east-1 fails roughly once a year with significant impact. Active-passive multi-region at minimum.

2. **Audit your hidden inter-region dependencies.** Your monitoring, alerting, CI/CD, and auth systems — are they in the same region as production? They must be independent.

3. **Avoid circular dependencies.** The AWS situation where CloudWatch fails because Kinesis fails, and Kinesis uses CloudWatch for its own monitoring, is a circular dependency death spiral. Your health monitoring system must not depend on the system it monitors.

4. **Test your DR.** Many companies thought they had multi-region but discovered they didn't when the actual outage hit. Run chaos experiments in your secondary region quarterly.

5. **Design for the console being down.** AWS Console runs in us-east-1. In a regional outage, you may not be able to access it. Automation (Infrastructure as Code, runbooks with CLI commands, cross-region IAM users) must work without the console.

6. **RTO and RPO are not theoretical.** They become very real during an outage. Know your numbers, test against them, and make sure your DR runbook achieves them.

---

## 📋 Disaster Recovery Runbook Template

Every team should have a DR runbook. Here's a template:

```markdown
# Disaster Recovery Runbook: Database Failure

## Severity: P1
## Estimated RTO: 15 minutes
## Estimated RPO: < 30 seconds

## Initial Assessment (0-2 minutes)
- [ ] Check CloudWatch: is RDS primary still responding?
  Command: `aws rds describe-db-instances --db-instance-identifier prod-postgres`
- [ ] Check application error rates in Grafana dashboard: "DB Connection Errors"
- [ ] Confirm this is a DB failure, not application bug (check recent deploys)

## Automated Failover Check (2-5 minutes)
- [ ] For RDS Multi-AZ: check if automatic failover already initiated
  Event: "Multi-AZ failover has completed" in RDS Events
- [ ] Wait up to 2 minutes for automatic failover to complete
- [ ] Test connectivity: `psql -h prod-postgres.cluster-xyz.us-east-1.rds.amazonaws.com -U app`

## Manual Failover (if needed, 5-15 minutes)
- [ ] Trigger manual failover via AWS CLI:
  `aws rds failover-db-cluster --db-cluster-identifier prod-postgres-cluster`
- [ ] Monitor failover status in RDS Events console
- [ ] Verify application is reconnecting (check connection pool metrics)

## Application Recovery (if needed)
- [ ] Rolling restart of app pods to clear stale connections:
  `kubectl rollout restart deployment/app-server -n production`
- [ ] Verify health endpoint: `curl https://api.prod/health/deep`

## Post-Incident
- [ ] Confirm all services are healthy (error rate < 0.1%, latency normal)
- [ ] Document actual RTO and RPO achieved
- [ ] Schedule post-mortem within 48 hours
- [ ] Create tickets for any missing automation

## Contacts
- On-call DB team: PagerDuty "Database" rotation
- AWS Enterprise Support: 1-800-xxx-xxxx (Case ID: pre-create case)
- Incident Commander: #incident-response Slack channel
```

---

## ⚖️ Trade-offs & Best Practices

### Trade-offs

| Feature | Pro | Con |
|---|---|---|
| Multi-region active-active | < 1 min RTO, latency wins | Distributed writes complexity, conflict resolution, 3× cost |
| Synchronous replication | Zero data loss (RPO=0) | Write latency +50-150ms (cross-region RTT) |
| Async replication | No write latency impact | Potential data loss (RPO = replication lag) |
| Automatic failover | Fast recovery, no human needed | Risk of "split-brain" if health checks are misconfigured |
| Active-passive | Simpler, lower cost | Recovery time = detection + promotion (minutes) |

### Best Practices

1. **Start with multi-AZ, plan for multi-region.** Multi-AZ protects against hardware failure, AZ failure. Multi-region protects against regional catastrophes. Most businesses need multi-AZ; critical businesses need multi-region.

2. **Test your DR quarterly.** Untested DR is fiction. Run real failovers, measure actual RTO/RPO, fix gaps.

3. **Automate failover.** Human failover takes 20+ minutes. Automated failover takes 1-2 minutes. Every minute of outage costs money.

4. **Separate monitoring from the system being monitored.** Your monitoring and alerting infrastructure must survive the failure you're trying to detect.

5. **Know your RTO and RPO.** These must be business-driven numbers, not vague aspirations. Work backward from acceptable business impact to required technology.

6. **Implement circuit breakers for cascade protection.** When DB fails, prevent all app threads from blocking on DB connections (cascade failure). Circuit breaker + fallback keeps the app alive.

7. **Pre-create incident response channels.** During an outage, you don't want to spend 5 minutes creating a Slack channel. Have a permanent #incident-response channel and documented escalation path.

8. **Run game days.** Simulate outages in a controlled manner with the team. Practice communication, decision-making, and runbook execution before an actual incident.

9. **Chaos engineering in staging, then production.** Start by breaking things in staging to build DR muscle memory, then graduate to production (lower blast radius experiments first).

10. **Document lessons from every incident.** Post-mortems should identify: what went wrong, why, what was done to recover, what systemic improvements prevent recurrence.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **RTO** | How long can system be down; drives failover speed requirements |
| **RPO** | How much data can be lost; drives replication synchrony requirements |
| **Availability nines** | 99.9% to 99.99% is a 10x improvement; each nine is exponentially harder |
| **SPOF elimination** | Every single-point-of-failure is a business risk; redundancy at every layer |
| **Active-active** | Best availability and performance; complex for stateful systems |
| **Aurora Global DB** | < 1 minute regional failover, < 1 second replication lag |
| **3-2-1 backup** | 3 copies, 2 media, 1 offsite; test restores regularly |
| **Graceful degradation** | System degrades partially on failure rather than completely |
| **AWS us-east-1 lesson** | No single region is safe; hidden inter-region dependencies create invisible SPOFs |
| **DR runbooks** | Documented, tested, automated; DRs work under stress only if practiced |

---

*Next: [16_System_Design_Case_Studies.md](16_System_Design_Case_Studies.md) — End-to-end designs: URL shortener, Twitter, YouTube, Uber, and WhatsApp.*
