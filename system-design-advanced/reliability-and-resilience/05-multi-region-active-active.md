---
slug: multi-region-active-active
title: Multi Region Active Active
readTime: 20 min
orderIndex: 5
premium: false
---



# Multi-Region Active-Active Architecture (When One Region Isn't Enough)

> Audience: platform architects and infrastructure engineers designing globally distributed systems.

This article assumes:
- Your users are geographically distributed across continents.
- Regional failures happen: AWS us-east-1 goes down, Azure has outages, natural disasters strike data centers.
- Network latency across continents is physics-limited (speed of light).
- Data consistency across regions is hard - CAP theorem is real.

---

## Challenge: Your "multi-region" architecture fails the regional outage test

### Scenario
You proudly announce: "We're multi-region! We have servers in US and EU."

Then AWS us-east-1 has a major outage.

Reality check:
- Your US database was the "primary"
- EU database was a "read replica"
- EU can serve reads but not writes
- All US customers can't create orders, update profiles, or process payments
- Failover requires 45 minutes of manual work

This is multi-region, but it's not active-active.

### Interactive question (pause and think)
What's the difference between:
1. Multi-region deployment (servers in multiple regions)
2. Multi-region active-passive (failover capability)
3. Multi-region active-active (simultaneous operation)

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: They represent increasing levels of availability and complexity.

1. **Multi-region deployment**: Geographic distribution, but no failover plan
2. **Active-passive**: One region serves traffic, other is hot standby
3. **Active-active**: All regions serve traffic simultaneously, survive regional failures transparently

### Real-world analogy (fire stations)
Imagine a city with two fire stations:

**Active-passive**: Fire Station A handles all calls. Fire Station B is staffed but idle, only activates if Station A burns down.

**Active-active**: Both stations handle calls simultaneously. If Station A is destroyed, Station B already handles 50% of the city. The impact is degradation, not catastrophe.

### Key insight box
> Active-active means every region can independently serve reads AND writes, with data eventually synchronized across regions.

### Challenge question
If active-active is so much better, why doesn't everyone do it? What's the hidden cost?

---

##  Mental model - Active-active is a distributed systems problem, not a deployment problem

### Scenario
Your VP says: "Just deploy to multiple regions. That's active-active."

You know it's not that simple.

### Interactive question (pause and think)
You deploy identical application servers to US and EU regions. Which statement is true?

A. "We're now active-active."
B. "We need to solve data synchronization first."
C. "We need to solve routing first."
D. "B and C, plus conflict resolution, monitoring, testing..."

### Progressive reveal
Answer: D.

Active-active isn't about deployment - it's about solving distributed data problems.

### Mental model
Think of active-active as:

- **Distributed consensus problem**: How do regions agree on state?
- **CAP theorem trade-off**: Pick consistency, availability, partition tolerance (choose 2)
- **Eventual consistency game**: Accept that regions will temporarily disagree

The challenges:
1. **Data layer**: How to keep databases synchronized
2. **Routing layer**: How to direct users to the right region
3. **Conflict resolution**: What happens when both regions modify the same data
4. **Failure detection**: How to know when a region is down
5. **Operational complexity**: How to deploy, test, and debug across regions

### Real-world parallel (bank branches)
Two bank branches in different cities (US and EU):

**The problem**: Customer withdraws $100 in US branch, simultaneously tries to withdraw $100 in EU branch. Account has $100 balance.

**Bad solution**: Both branches allow withdrawal (now -$100, data inconsistency).

**Active-passive solution**: Only US branch can approve withdrawals (EU can't help if US is down).

**Active-active solution**: Both can approve, but they synchronize and detect conflicts. Maybe allow temporary overdraft, reconcile later.

### Key insight box
> Active-active is 20% deployment and 80% distributed data correctness. If you only solve deployment, you'll have data corruption at scale.

### Challenge question
Can you have active-active for reads but active-passive for writes? Would that be useful?

---

##  Understanding the CAP theorem trade-offs in multi-region

### Scenario
You're designing active-active architecture. Your database vendor says "Choose: strong consistency or high availability."

You want both. You can't have both (during network partitions).

### CAP theorem reminder

**C (Consistency)**: Every read sees the most recent write
**A (Availability)**: Every request gets a response (success or failure)
**P (Partition tolerance)**: System works despite network failures

**The theorem**: During a network partition (P), you must choose between C and A.

### Visual: CAP trade-offs in multi-region

```text
┌─────────────────────────────────────────────────────────┐
│              CAP THEOREM IN MULTI-REGION                │
└─────────────────────────────────────────────────────────┘

Scenario: Network partition between US and EU regions

          US Region                    EU Region
             │                              │
             │  ╳╳╳╳╳ partition ╳╳╳╳╳       │
             │                              │
             ▼                              ▼

CHOICE 1: Prioritize Consistency (CP)
  US: Reject writes until partition heals
  EU: Reject writes until partition heals
  Result: Unavailable, but consistent
  Example: Traditional RDBMS with synchronous replication

CHOICE 2: Prioritize Availability (AP)
  US: Accept writes, queue for EU later
  EU: Accept writes, queue for US later
  Result: Available, but temporarily inconsistent
  Example: Cassandra, DynamoDB

CHOICE 3: Hybrid (common in practice)
  Critical data: Choose CP (payments, inventory)
  Non-critical data: Choose AP (analytics, caching)
  Result: Different consistency per feature
```

### Multi-region consistency patterns

**1. Strong consistency (CP)**
```yaml
approach: synchronous_replication
write_flow:
  1. Write to US region
  2. Wait for EU region to acknowledge
  3. Only then confirm to client

pros:
  - Data is always consistent
  - No conflict resolution needed

cons:
  - High latency (cross-region network)
  - Unavailable during partition
  - Lower throughput

use_cases:
  - Financial transactions
  - Inventory management
  - User authentication
```

**2. Eventual consistency (AP)**
```yaml
approach: asynchronous_replication
write_flow:
  1. Write to local region (US or EU)
  2. Immediately confirm to client
  3. Asynchronously replicate to other region

pros:
  - Low latency (local writes)
  - High availability
  - Higher throughput

cons:
  - Temporary inconsistency
  - Requires conflict resolution
  - Complex to reason about

use_cases:
  - Social media feeds
  - Recommendation engines
  - Analytics data
```

**3. Read-your-writes consistency**
```yaml
approach: session_affinity
write_flow:
  1. Write to local region
  2. Replicate asynchronously
  3. Ensure same user always hits same region (sticky sessions)

result:
  - User sees their own writes immediately
  - Other users might see stale data temporarily

use_cases:
  - User profile updates
  - Shopping cart
  - Preferences
```

### Interactive question
You're building a global e-commerce platform. Which consistency model for each feature?

A. Product catalog: Strong / Eventual / Read-your-writes
B. Inventory count: Strong / Eventual / Read-your-writes
C. Shopping cart: Strong / Eventual / Read-your-writes
D. Payment processing: Strong / Eventual / Read-your-writes

### Progressive reveal
Answer:
- Product catalog: Eventual (stale product descriptions are tolerable)
- Inventory count: Strong (can't oversell)
- Shopping cart: Read-your-writes (user's own cart must be consistent)
- Payment processing: Strong (no duplicate charges)

### Key insight box
> There's no "one size fits all" consistency model. Active-active requires choosing the right model per feature based on business requirements.

### Challenge question
If you choose eventual consistency, how do you handle conflicts when both regions modify the same record simultaneously?

---

##  Core components of active-active architecture

### Scenario
You're building active-active. What are the actual building blocks?

### Architecture layers

```text
┌─────────────────────────────────────────────────────────┐
│         ACTIVE-ACTIVE ARCHITECTURE LAYERS               │
└─────────────────────────────────────────────────────────┘

Layer 1: GLOBAL ROUTING
  ├─ DNS-based (Route53, Cloudflare)
  ├─ Anycast (BGP routing)
  └─ Application-level (smart clients)
  Purpose: Direct users to nearest healthy region

Layer 2: REGIONAL LOAD BALANCING
  ├─ Regional LB (ALB, NLB)
  ├─ Service mesh (Istio, Linkerd)
  └─ API Gateway
  Purpose: Distribute load within region

Layer 3: APPLICATION TIER (stateless)
  ├─ Identical code deployed to all regions
  ├─ Autoscaling per region
  └─ Circuit breakers for cross-region calls
  Purpose: Process requests

Layer 4: DATA TIER (stateful - the hard part)
  ├─ Multi-region databases
  ├─ Replication (sync or async)
  ├─ Conflict resolution
  └─ Sharding strategy
  Purpose: Store and sync data

Layer 5: OBSERVABILITY
  ├─ Global metrics aggregation
  ├─ Cross-region tracing
  └─ Regional health checks
  Purpose: Detect failures, debug issues
```

### Data tier patterns (the heart of active-active)

**Pattern 1: Multi-master database replication**
```yaml
database: PostgreSQL with logical replication
topology:
  us_region:
    primary: postgres-us-primary
    replicates_to: [postgres-eu-primary]

  eu_region:
    primary: postgres-eu-primary
    replicates_to: [postgres-us-primary]

conflict_resolution: last-write-wins (timestamp-based)

challenges:
  - Replication lag (seconds to minutes)
  - Conflict detection and resolution
  - Schema changes across regions

example_tech:
  - PostgreSQL with BDR (bi-directional replication)
  - MySQL with Galera Cluster
  - CockroachDB (built-in multi-region)
```

**Pattern 2: Distributed databases (Cassandra, DynamoDB)**
```yaml
database: DynamoDB Global Tables
topology:
  regions: [us-east-1, eu-west-1, ap-south-1]
  replication: automatic (managed by AWS)
  consistency: eventual (tunable)

conflict_resolution:
  strategy: last-write-wins
  conflict_detection: vector clocks

benefits:
  - Managed replication
  - No manual failover
  - High availability

challenges:
  - Eventually consistent by default
  - Limited query capabilities
  - Cost (pay per region)
```

**Pattern 3: Sharding by geography**
```yaml
approach: geo_sharding
strategy:
  us_customers: write to us_region (primary)
  eu_customers: write to eu_region (primary)
  cross_region_reads: allowed (may be stale)

example:
  customer_id: user-us-12345
  routing: route to US region

  customer_id: user-eu-67890
  routing: route to EU region

benefits:
  - Low latency (local writes)
  - Reduced conflicts (customers usually in one region)
  - Data residency compliance (GDPR)

challenges:
  - What if customer travels? (temporary latency increase)
  - Cross-region queries are expensive
  - Rebalancing shards is complex
```

### Application-level conflict resolution

```go
package multiregion

import "time"

// Conflict resolution strategies

// Strategy 1: Last-Write-Wins (LWW)
type LWWRecord struct {
    ID        string
    Value     string
    Timestamp time.Time
    Region    string
}

func (r *LWWRecord) MergeWith(other *LWWRecord) *LWWRecord {
    if other.Timestamp.After(r.Timestamp) {
        return other
    }
    return r
}

// Strategy 2: Custom business logic
type BankAccount struct {
    ID            string
    Balance       int
    Version       int
    LastModified  time.Time
    LastRegion    string
}

func (a *BankAccount) MergeWith(other *BankAccount) (*BankAccount, error) {
    // Custom merge: sum deposits, take max of withdrawals
    // This is domain-specific!

    if a.Version == other.Version {
        // True conflict - both modified same version
        return nil, errors.New("conflict: requires manual resolution")
    }

    // Take newer version
    if other.Version > a.Version {
        return other, nil
    }
    return a, nil
}

// Strategy 3: CRDT (Conflict-free Replicated Data Type)
type GCounter struct {
    Counts map[string]int // region -> count
}

func (g *GCounter) Increment(region string, delta int) {
    g.Counts[region] += delta
}

func (g *GCounter) MergeWith(other *GCounter) *GCounter {
    merged := &GCounter{Counts: make(map[string]int)}

    // Take max count per region
    for region, count := range g.Counts {
        merged.Counts[region] = max(count, other.Counts[region])
    }

    for region, count := range other.Counts {
        if _, exists := merged.Counts[region]; !exists {
            merged.Counts[region] = count
        }
    }

    return merged
}

func (g *GCounter) Total() int {
    sum := 0
    for _, count := range g.Counts {
        sum += count
    }
    return sum
}
```

### Key insight box
> The data tier is where active-active lives or dies. Application tier is easy (stateless). Data tier requires careful consistency modeling.

### Challenge question
How do you test active-active data synchronization? Simulating network partitions between regions is hard.

---

##  Routing users to regions - DNS, Anycast, or Application?

### Scenario
You have servers in US and EU. User in New York makes a request. Where should it go?

### Think about it
What's the goal?
- Lowest latency (nearest region)
- Load balancing (evenly distribute)
- Failover (route away from unhealthy region)

### Interactive question (pause and think)
Which routing approach gives you all three?

A. DNS-based (Route53 with latency-based routing)
B. Anycast (BGP announces same IP from multiple regions)
C. Application-level (smart client chooses region)
D. None - trade-offs required

### Progressive reveal
Answer: D - each approach has trade-offs.

### Routing approaches comparison

**DNS-based routing**
```yaml
approach: route53_latency_routing
how_it_works:
  1. Client queries DNS for api.example.com
  2. Route53 measures client's latency to each region
  3. Returns IP of lowest-latency region

pros:
  - Simple to implement
  - Works with any client
  - Supports health checks (failover)

cons:
  - DNS caching (60+ sec to failover)
  - Client might ignore DNS TTL
  - No session affinity by default

best_for: Web applications, mobile apps (not real-time)
```

**Anycast routing**
```yaml
approach: bgp_anycast
how_it_works:
  1. All regions announce same IP (e.g., 1.2.3.4)
  2. Internet routes traffic to "nearest" region (BGP hops)
  3. Automatic failover (if region dies, traffic re-routes)

pros:
  - Fast failover (seconds)
  - No DNS caching issues
  - Truly "nearest" region

cons:
  - Requires BGP control (complex)
  - Session affinity hard (connection might shift mid-session)
  - Not all clouds support it easily

best_for: DDoS mitigation, DNS servers, CDNs
```

**Application-level routing**
```yaml
approach: smart_client
how_it_works:
  1. Client app has logic to choose region
  2. Measures latency to all regions on startup
  3. Routes requests to fastest region
  4. Retries in another region on failure

pros:
  - Full control over routing logic
  - Session affinity easy (client remembers region)
  - Can implement complex strategies

cons:
  - Requires client changes (not always possible)
  - Each client needs region discovery
  - More complex debugging

best_for: Mobile apps, thick clients, SDKs
```

**Hybrid approach (common in practice)**
```yaml
approach: layered_routing
layers:
  1_dns:
    - Route53 returns multiple IPs (all regions)
    - Client gets list of healthy regions

  2_application:
    - Client tests latency to all IPs
    - Chooses lowest latency
    - Stores preference in local cache

  3_failover:
    - If chosen region fails, try next in list
    - Update preference

example:
  regions: [us-east, us-west, eu-west]
  dns_response: [1.2.3.4, 5.6.7.8, 9.10.11.12]
  client_test: us-east = 50ms, us-west = 80ms, eu-west = 150ms
  client_choice: us-east (cached for 5 minutes)
```

### Production insight: How Netflix routes globally

Netflix uses:
1. DNS returns multiple region IPs
2. Client device tests each region (background)
3. Client chooses best region based on throughput test
4. Caches choice for session duration
5. Automatically retries other regions on failure

### Key insight box
> No single routing approach is perfect. Most production systems use DNS for coarse routing + application logic for fine-grained decisions.

### Challenge question
User is in New York but their data is sharded to EU region (data residency). Should routing send them to US (low latency) or EU (where their data lives)?

---

##  Handling regional failures - detection and recovery

### Scenario
Your EU region goes down. How quickly can you detect it and route traffic away?

### Failure detection challenges

**Challenge 1: What does "down" mean?**
```yaml
partial_failures:
  - Network partition (region alive but unreachable)
  - Database down (app healthy, data tier broken)
  - Elevated latency (not down, but degraded)
  - Elevated error rate (some requests succeed)

question: At what threshold do you failover?
  - 100% error rate? (too late, users already impacted)
  - 50% error rate? (might be transient)
  - Elevated latency > 5 seconds? (degraded but not down)
```

**Challenge 2: Distributed failure detection**
```text
Who detects the failure?

Option 1: Health checks from other regions
  US region pings EU region every 10 seconds
  If 3 consecutive failures → mark EU as down

  Problem: Network partition might affect health checks but not user traffic

Option 2: Traffic-based detection
  Monitor actual user request success rate to EU
  If error rate > 50% for 60 seconds → mark as down

  Problem: Slow to detect, users already impacted

Option 3: Hybrid
  Both health checks AND traffic monitoring
  Failover if either signals failure
```

### Failover strategies

**Strategy 1: DNS failover (slow)**
```yaml
dns_failover:
  detection: Route53 health checks every 30 seconds
  action: Remove failed region from DNS responses
  recovery_time: 60+ seconds (DNS TTL + propagation)

  flow:
    1. EU region fails
    2. Route53 detects failure (30 sec)
    3. Removes EU IP from DNS (instant)
    4. Clients respect TTL and re-query (30-60 sec)
    5. New requests go to US only

  impact: 1-2 minutes of partial failures
```

**Strategy 2: Anycast failover (fast)**
```yaml
anycast_failover:
  detection: BGP withdraw (automatic on region failure)
  action: Internet re-routes traffic to next-nearest region
  recovery_time: 10-30 seconds (BGP convergence)

  flow:
    1. EU region fails
    2. BGP stops announcing EU route
    3. Internet automatically re-routes to US

  impact: 10-30 seconds of failures
```

**Strategy 3: Application-level retry (fastest)**
```yaml
application_retry:
  detection: Client receives error from region
  action: Client immediately retries request to alternate region
  recovery_time: < 1 second (one request timeout)

  flow:
    1. Client sends request to EU
    2. EU times out (1 second)
    3. Client immediately retries to US
    4. US responds successfully

  impact: 1-2 seconds per request (transparent to user)
```

### Failover decision matrix

```go
package failover

import "time"

type RegionHealth struct {
    Region         string
    ErrorRate      float64
    Latency        time.Duration
    HealthCheck    bool
    LastUpdate     time.Time
}

type FailoverPolicy struct {
    ErrorRateThreshold    float64
    LatencyThreshold      time.Duration
    HealthCheckRequired   bool
    ConsecutiveFailures   int
}

func (p *FailoverPolicy) ShouldFailover(health RegionHealth) bool {
    // Multiple signals required for failover

    // Signal 1: Health check failed
    if p.HealthCheckRequired && !health.HealthCheck {
        return true
    }

    // Signal 2: Error rate exceeded
    if health.ErrorRate > p.ErrorRateThreshold {
        return true
    }

    // Signal 3: Latency degraded significantly
    if health.Latency > p.LatencyThreshold {
        return true
    }

    return false
}

// Example policy
var AggressiveFailover = FailoverPolicy{
    ErrorRateThreshold:  0.10, // 10% errors
    LatencyThreshold:    2 * time.Second,
    HealthCheckRequired: true,
    ConsecutiveFailures: 2,
}

var ConservativeFailover = FailoverPolicy{
    ErrorRateThreshold:  0.50, // 50% errors
    LatencyThreshold:    10 * time.Second,
    HealthCheckRequired: true,
    ConsecutiveFailures: 5,
}
```

### Avoiding failover oscillation (flapping)

```yaml
problem: flapping_regions
scenario:
  1. EU region degraded → failover to US
  2. US now overloaded (handling 2x traffic) → slow
  3. EU recovers → failover back to EU
  4. EU overloaded → failover to US again
  5. Infinite loop of failover

solution: hysteresis
  - Failover threshold: error rate > 50%
  - Recovery threshold: error rate < 10% (much lower)
  - Cooldown period: 5 minutes minimum between failovers

code:
  if health.ErrorRate > 0.50 and not recently_failed_over():
      failover()
      set_cooldown(5 * time.Minute)

  if health.ErrorRate < 0.10 and cooldown_expired():
      failback()
```

### Key insight box
> Fast failure detection is worthless if you don't have fast recovery. Focus on application-level retries for the best user experience.

### Challenge question
During failover, what happens to in-flight requests? Do they fail, get retried, or complete successfully?

---

##  The hidden costs of active-active

### Scenario
Your CFO asks: "Why did our cloud bill double when we went active-active?"

You're running twice the infrastructure, but is that the only cost?

### Cost dimensions

**1. Infrastructure costs (obvious)**
```yaml
single_region: $100K/month
  - 100 servers
  - 1 database cluster
  - 1 load balancer

multi_region_active_active: $250K/month (not 2x!)
  - 200 servers (2x)
  - 2 database clusters (2x)
  - 2 load balancers (2x)
  - Cross-region data transfer ($$$)
  - Global routing (Route53, GSLB)
  - Additional monitoring/observability

breakdown:
  infrastructure: $200K (2x)
  data_transfer: $30K (new cost)
  tooling: $20K (new cost)
```

**2. Data transfer costs (hidden)**
```yaml
cross_region_transfer:
  database_replication:
    - 1TB/day of database changes
    - $0.02/GB cross-region
    - $20/day × 30 days = $600/month

  application_traffic:
    - Users in EU accessing US data
    - 10TB/month cross-region
    - $0.02/GB = $200/month

  total: $800/month (8% of infrastructure cost)
```

**3. Operational costs (often underestimated)**
```yaml
operational_overhead:
  complexity:
    - 2x deployments (or coordinated multi-region deploys)
    - 2x monitoring dashboards
    - Cross-region debugging (harder)
    - Conflict resolution (manual intervention)

  team_impact:
    - Longer incident response (which region?)
    - More runbooks (failover procedures)
    - Additional on-call load
    - Training on distributed systems

  estimated_cost: +30% engineering time
```

**4. Performance costs (sometimes)**
```yaml
latency_impact:
  synchronous_replication:
    - Write to US region
    - Wait for EU region to acknowledge
    - Additional latency: 100-150ms (cross-Atlantic)

  result:
    - p99 write latency: 200ms → 350ms
    - User-perceived slowness
    - May need to adjust consistency model
```

### Cost-benefit analysis framework

```yaml
question: Is active-active worth it for your workload?

benefits:
  - Zero-downtime regional failover: $X million saved/year (outage cost)
  - Lower latency for global users: +Y% conversion rate
  - Compliance (data residency): Required for EU expansion

costs:
  - Infrastructure: +$150K/year
  - Data transfer: +$10K/year
  - Operational overhead: +$200K/year (30% of 2 engineers)

total_cost: $360K/year
total_benefit: $X million (depends on outage frequency and impact)

decision:
  if expected_outage_cost > $360K/year:
    active_active = justified
  else:
    consider active_passive or single-region with backups
```

### When NOT to use active-active

```yaml
anti_patterns:

  low_traffic_services:
    - < 1000 requests/second
    - Can tolerate 1 hour downtime
    - Single region sufficient

  data_heavy_infrequent_writes:
    - Analytics pipelines
    - Batch processing
    - Active-passive or async replication better

  strong_consistency_required_everywhere:
    - Financial ledger (synchronous replication kills latency)
    - Better: single-region with HA, backups

  early_stage_startups:
    - Premature optimization
    - Focus on product-market fit first
    - Add active-active when revenue justifies cost
```

### Key insight box
> Active-active is expensive (money, complexity, operational overhead). Only use it when downtime costs exceed those expenses.

### Challenge question
Your active-active setup costs $300K/year extra. Your biggest outage last year cost $50K. Should you downgrade to active-passive?

---

##  Final synthesis - Design your active-active architecture

### Synthesis challenge
You're the architect for a global SaaS collaboration platform (think: Slack, Microsoft Teams).

Requirements:
- 10 million users across US (40%), EU (35%), Asia (25%)
- Real-time messaging (< 100ms latency critical)
- File storage (documents, images)
- User presence (online/offline status)
- Search (messages, files)
- Compliance: EU data must stay in EU (GDPR)

Constraints:
- Current: Single US region, experiencing EU latency complaints
- Budget: Can double infrastructure costs
- Team: 15 engineers, limited distributed systems expertise

### Your tasks (pause and think)
1. Choose regions and topology (how many regions? which cloud zones?)
2. Define data residency strategy (what data goes where?)
3. Choose consistency model per feature (strong vs eventual)
4. Design routing strategy (how do users reach their region?)
5. Plan failover strategy (what happens when a region fails?)
6. Estimate costs and justify to CFO

Write down your architecture.

### Progressive reveal (one possible solution)

**1. Regions and topology:**
```yaml
regions:
  us_east: # Primary for US users
    provider: AWS us-east-1
    capacity: 40% of total load

  eu_west: # Primary for EU users
    provider: AWS eu-west-1
    capacity: 35% of total load

  ap_south: # Primary for Asia users
    provider: AWS ap-south-1
    capacity: 25% of total load

topology: active_active
  - All regions can serve reads and writes
  - Data replicated across regions (asynchronously)
  - Each region can survive independently
```

**2. Data residency strategy:**
```yaml
data_sharding:
  user_accounts:
    shard_key: user_id_region_prefix
    strategy: geo_sharding
    example:
      - user-us-12345 → stored in US region
      - user-eu-67890 → stored in EU region
    compliance: GDPR-compliant (EU data in EU)

  messages:
    strategy: shard_by_workspace
    workspace-us-abc → stored in US
    workspace-eu-xyz → stored in EU
    replication: none (messages stay local)

  files:
    storage: S3 with regional buckets
    strategy: upload to local region, CDN for global access

  presence:
    strategy: eventually consistent global state
    tech: Redis with cross-region replication
    latency: < 1 second propagation

  search_index:
    strategy: ElasticSearch per region
    replication: async (5 minute lag acceptable)
```

**3. Consistency model per feature:**
```yaml
consistency_requirements:

  real_time_messaging:
    model: eventual_consistency
    rationale: Message ordering per-channel, eventual global view
    tech: Kafka per region, async replication
    acceptable_lag: < 5 seconds

  user_authentication:
    model: strong_consistency
    rationale: Login must work globally, can't have stale passwords
    tech: CockroachDB (global consensus)
    tradeoff: Slightly higher latency (100-200ms)

  file_uploads:
    model: read_your_writes
    rationale: User must see their own uploads immediately
    tech: S3 regional buckets + session affinity

  presence_status:
    model: eventual_consistency
    rationale: OK if status is stale by 1-2 seconds
    tech: Redis global replication

  workspace_settings:
    model: strong_consistency
    rationale: Avoid conflicts on admin changes
    tech: DynamoDB with global tables (last-write-wins)
```

**4. Routing strategy:**
```yaml
routing_layers:

  layer_1_dns:
    provider: Route53
    policy: geolocation_routing
    mapping:
      - North America → us-east-1
      - Europe → eu-west-1
      - Asia → ap-south-1
    health_check: HTTP /health every 30 seconds

  layer_2_application:
    - Client app tests latency to all regions on startup
    - Chooses lowest latency region
    - Caches choice for session duration

  layer_3_data_affinity:
    - If user's data is in EU (user-eu-XXX)
    - Route to EU region even if user is in US
    - Accept higher latency for data residency

  failover:
    - Primary region down → DNS removes from rotation
    - Client automatically retries in alternate region
    - User session preserved (shared session store)
```

**5. Failover strategy:**
```yaml
failover_scenarios:

  scenario_1_region_total_failure:
    detection:
      - Route53 health checks fail (3 consecutive)
      - Application error rate > 80% for 2 minutes
    action:
      - Remove region from DNS (automatic)
      - Clients retry in next-nearest region
      - Data still accessible (replicated to other regions)
    recovery_time: 2-3 minutes

  scenario_2_database_failure:
    detection:
      - Database connection pool exhausted
      - Query latency > 10 seconds
    action:
      - Failover to read replica in same region
      - If replica also down, route queries to other region's DB
    recovery_time: 30 seconds

  scenario_3_network_partition:
    detection:
      - Cross-region replication lag > 5 minutes
      - Health checks from other regions fail
    action:
      - Each region operates independently (partition tolerance)
      - Accept temporary inconsistency
      - Reconcile after partition heals
    recovery_time: automatic when partition heals

  testing:
    - Monthly Game Day (simulate region failure)
    - Automated chaos engineering (random instance failures)
    - Quarterly full-region failover drill
```

**6. Cost estimate:**
```yaml
cost_breakdown:

  current_single_region: $200K/month
    - Compute: $120K
    - Database: $50K
    - Storage: $20K
    - Networking: $10K

  projected_multi_region: $450K/month
    - Compute: $300K (2.5x, accounting for overhead)
    - Database: $100K (2x)
    - Storage: $40K (2x, but S3 multi-region)
    - Cross-region data transfer: $30K (new)
    - Global routing: $5K
    - Monitoring/observability: $15K

  additional_costs:
    - Engineering time: +30% (1.5 engineers dedicated)
    - Training: $50K one-time
    - Migration project: $200K one-time

  total_annual_increase: $3M + $250K one-time

  justification_to_CFO:
    - Current revenue: $50M/year
    - EU expansion blocked by latency (potential +$20M/year)
    - Last outage cost: $2M in lost revenue + customer churn
    - Compliance: GDPR required for EU customers
    - Competitive advantage: 99.99% uptime SLA vs competitors' 99.9%

    ROI: $20M revenue + $2M risk mitigation - $3M cost = $19M net benefit
```

### Key insight box
> Active-active is a business decision, not just a technical one. The architecture must be justified by revenue, compliance, or risk mitigation.

### Final challenge question
Your active-active architecture is live. Both US and EU regions write to the same workspace simultaneously (conflict). How do you resolve: last-write-wins, manual merge, or reject one write?

---

## Appendix: Quick checklist (printable)

**Active-active design checklist:**
- [ ] Choose regions based on user distribution (not just "US and EU")
- [ ] Define data residency requirements (GDPR, data sovereignty)
- [ ] Pick consistency model per feature (strong vs eventual)
- [ ] Design conflict resolution strategy (LWW, CRDTs, manual)
- [ ] Plan routing strategy (DNS, anycast, application-level)
- [ ] Define failover thresholds and procedures
- [ ] Estimate costs (infrastructure, data transfer, operational)

**Data tier checklist:**
- [ ] Choose database technology (multi-master, distributed, sharded)
- [ ] Configure replication (async vs sync)
- [ ] Set replication lag SLOs (how stale is acceptable?)
- [ ] Implement conflict detection and resolution
- [ ] Plan schema migration across regions (coordinated rollout)
- [ ] Test failure scenarios (partition, region down, lag spike)

**Operational checklist:**
- [ ] Deploy monitoring per region (plus global aggregation)
- [ ] Set up cross-region tracing (distributed traces)
- [ ] Create runbooks for common failures
- [ ] Practice failover (monthly Game Days)
- [ ] Monitor replication lag (alert on thresholds)
- [ ] Track cross-region data transfer costs

**Testing checklist:**
- [ ] Test cross-region writes (verify replication)
- [ ] Test conflict scenarios (simultaneous writes)
- [ ] Test regional failures (simulate region down)
- [ ] Test network partitions (split-brain scenarios)
- [ ] Load test each region independently
- [ ] Test failover time (measure actual vs target)

**Cost optimization checklist:**
- [ ] Right-size regional capacity (based on actual traffic)
- [ ] Optimize cross-region data transfer (compress, dedupe)
- [ ] Use reserved instances (if traffic predictable)
- [ ] Implement data lifecycle policies (archive old data)
- [ ] Monitor cost per region (detect anomalies)

**Red flags (reassess architecture):**
- [ ] Replication lag consistently > 10 seconds (data tier not scaling)
- [ ] Cross-region transfer costs > 10% of infrastructure (over-replicating)
- [ ] Frequent conflicts requiring manual resolution (sharding strategy wrong)
- [ ] Failover takes > 5 minutes (routing strategy inadequate)
- [ ] Operational overhead exceeds engineering capacity (too complex)
