---
slug: blast-radius-and-failure-domain-isolation
title: Blast Radius and Failure Domain Isolation
readTime: 20 min
orderIndex: 3
premium: false
---





# Blast Radius and Failure Domain Isolation (Containing the Blast)

> Audience: platform engineers and architects designing fault-tolerant distributed systems.

This article assumes:
- Failures are inevitable: services crash, networks partition, dependencies timeout.
- Failures cascade: one component failure can trigger a domino effect.
- Your system is larger than any one person understands completely.
- Customers don't care about your internal architecture - they care about their experience.

---

## Challenge: One microservice takes down your entire platform

### Scenario
It's Black Friday. Your recommendation service crashes due to a memory leak.

Within 2 minutes:

- The API gateway times out waiting for recommendations
- The timeout causes the gateway's connection pool to fill up
- New requests to ANY service (checkout, search, login) start failing
- Your entire platform is down

All because of one non-critical feature: product recommendations.

### Interactive question (pause and think)
What failed here?

1. The recommendation service (it had the bug)
2. The API gateway (it couldn't handle timeouts)
3. The system architecture (one failure cascaded)
4. The developers (they wrote buggy code)

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (3) - architecture failure.

The bug is inevitable. The cascade is not.

### Real-world analogy (ship bulkheads)
When the Titanic hit the iceberg, water flooded ONE compartment. But the bulkheads (walls) between compartments weren't tall enough. Water spilled over, compartment by compartment, until the ship sank.

Modern ships have watertight compartments that seal completely. One flooded room doesn't sink the ship.

Your system needs the same isolation.

### Key insight box
> Blast radius is the scope of impact when a component fails. Failure domain isolation limits that scope through architectural boundaries.

### Challenge question
If you could only add ONE isolation mechanism to your system right now, would you choose: timeouts, circuit breakers, or separate deployment units?

---

## Mental model - Failures want to spread; isolation contains them

### Scenario
Your system has three layers: frontend, backend, database.

Two design philosophies:

- **Tight coupling**: "Fast is efficient. Share everything: connections, memory, threads."
- **Loose coupling**: "Isolated is resilient. Assume dependencies will fail."

### Interactive question (pause and think)
Which statement is true?

A. "Isolation adds complexity, tight coupling is simpler."
B. "Isolation wastes resources, sharing is more efficient."
C. "Isolation trades some efficiency for resilience."

### Progressive reveal
Answer: C.

- A is only true for trivial systems.
- B measures the wrong efficiency (CPU/memory vs. customer impact).
- C recognizes the trade-off explicitly.

### Mental model
Think of failure domain isolation as:

- **Firebreaks** in a forest (stop fire spread)
- **Circuit breakers** in electrical systems (prevent overload cascades)
- **Quarantine zones** in disease control (contain outbreaks)

The goal: when something fails, the failure stays local.

### Real-world parallel (power grid)
The 2003 Northeast Blackout started with one overloaded transmission line in Ohio. It cascaded to 50 million people across 8 states and Canada.

Modern grids use isolation: circuit breakers segment regions. One grid section can fail without taking down neighboring states.

### Key insight box
> In distributed systems, components fail independently. In poorly designed systems, they fail together.

### Challenge question
Is it possible to have TOO MUCH isolation? What are the costs of over-isolating?

---

## Understanding blast radius dimensions

### Scenario
Your database goes down. What's the blast radius?

It depends on WHICH dimension you're measuring.

### Blast radius dimensions

**Customer impact:**
- How many users affected?
- Which features broken?
- For how long?

**Revenue impact:**
- How much money lost per minute?
- Which revenue streams affected?

**Component impact:**
- How many services degraded/failing?
- Which data inconsistent?

**Geographic impact:**
- One region? Multi-region?
- Which compliance zones affected?

**Time impact:**
- How long to detect?
- How long to mitigate?
- How long to fully recover?

### Visual: blast radius dimensions

```text
┌─────────────────────────────────────────────────────────┐
│              BLAST RADIUS DIMENSIONS                    │
└─────────────────────────────────────────────────────────┘

                    │
        Customer    │    100%  ──────  Complete outage
         Impact     │     50%  ──────  Major degradation
                    │     10%  ──────  Partial feature loss
                    │      1%  ──────  Minor impact
                    └──────────────────────────────────────►
                                                        Time

Containment strategies by dimension:

Customer Impact:
  ├─ Segment by: geography, tier, feature
  └─ Tools: feature flags, traffic routing, A/B infrastructure

Component Impact:
  ├─ Segment by: service boundaries, data sharding
  └─ Tools: bulkheads, separate deployments

Geographic Impact:
  ├─ Segment by: region, availability zone
  └─ Tools: multi-region, cell-based architecture

Time Impact:
  ├─ Segment by: detection speed, rollback speed
  └─ Tools: monitoring, automated rollback, chaos testing
```

### Interactive question
Your payment service has a bug. Which blast radius dimension should you minimize FIRST?

1. Customer impact (how many customers)
2. Revenue impact (how much money)
3. Time impact (how fast to fix)

### Progressive reveal
Answer: Trick question - minimize (1) ENABLES minimizing (2), but (3) is HOW you achieve it.

Fast detection and mitigation reduce all blast radius dimensions.

### Key insight box
> Blast radius isn't a single number - it's a multi-dimensional surface. You optimize different dimensions with different techniques.

### Challenge question
How do you measure blast radius BEFORE an incident happens? (Hint: Game Days)

---

##  Core isolation techniques (and their trade-offs)

### Scenario
You're designing a multi-tenant SaaS platform. 1000 customers, ranging from small businesses to enterprises.

How do you prevent one customer's bad behavior from impacting others?

### Isolation techniques catalog

**1. Process isolation (bulkheads)**
- **What**: Separate thread pools, connection pools, or processes per tenant/feature
- **Prevents**: Resource exhaustion cascade
- **Cost**: Memory/CPU overhead

**2. Network isolation (segmentation)**
- **What**: Separate VPCs, subnets, or service meshes per tenant/region
- **Prevents**: Network-level blasts (DDoS, routing failures)
- **Cost**: Infrastructure complexity

**3. Data isolation (sharding)**
- **What**: Separate database instances, schemas, or tables per tenant
- **Prevents**: Data corruption cascade, query load cascade
- **Cost**: Operational overhead, harder queries

**4. Deployment isolation (cells/clusters)**
- **What**: Separate Kubernetes clusters or server groups per region/tier
- **Prevents**: Deployment failures, configuration errors
- **Cost**: Duplication, slower rollouts

**5. Circuit breakers and timeouts**
- **What**: Fail fast when dependencies are slow/down
- **Prevents**: Waiting cascades, thread exhaustion
- **Cost**: False positives during recovery

### Visual: isolation techniques comparison

```text
Technique            │ Blast Radius │ Resource │ Complexity │ Best For
                     │  Reduction   │   Cost   │            │
─────────────────────┼──────────────┼──────────┼────────────┼─────────
Process isolation    │     HIGH     │  MEDIUM  │    LOW     │ Multi-tenant
(bulkheads)          │              │          │            │ services
─────────────────────┼──────────────┼──────────┼────────────┼─────────
Network isolation    │     HIGH     │   HIGH   │   MEDIUM   │ Security,
(VPC/subnets)        │              │          │            │ compliance
─────────────────────┼──────────────┼──────────┼────────────┼─────────
Data isolation       │   VERY HIGH  │   HIGH   │    HIGH    │ Strict tenant
(separate DBs)       │              │          │            │ separation
─────────────────────┼──────────────┼──────────┼────────────┼─────────
Deployment isolation │   VERY HIGH  │ VERY HIGH│    HIGH    │ Regulated
(separate clusters)  │              │          │            │ industries
─────────────────────┼──────────────┼──────────┼────────────┼─────────
Circuit breakers     │    MEDIUM    │   LOW    │   MEDIUM   │ Dependency
(fail fast)          │              │          │            │ failures
```

### Example: process isolation with thread pools

```go
package isolation

import (
    "context"
    "sync"
)

// Bulkhead pattern: separate thread pools per tenant or feature
type Bulkhead struct {
    name     string
    maxConcurrent int
    semaphore chan struct{}
}

func NewBulkhead(name string, maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        name:          name,
        maxConcurrent: maxConcurrent,
        semaphore:     make(chan struct{}, maxConcurrent),
    }
}

func (b *Bulkhead) Execute(ctx context.Context, task func() error) error {
    select {
    case b.semaphore <- struct{}{}:
        // Acquired slot
        defer func() { <-b.semaphore }()
        return task()
    case <-ctx.Done():
        return ctx.Err()
    default:
        // Bulkhead full - fail fast instead of queuing
        return ErrBulkheadFull{Bulkhead: b.name}
    }
}

// Usage: isolate different features
var (
    CheckoutBulkhead = NewBulkhead("checkout", 100)
    AnalyticsBulkhead = NewBulkhead("analytics", 20)
    SearchBulkhead = NewBulkhead("search", 50)
)

func HandleCheckout(ctx context.Context) error {
    return CheckoutBulkhead.Execute(ctx, func() error {
        // Critical checkout logic
        // Even if analytics/search are overloaded,
        // checkout has dedicated resources
        return processCheckout(ctx)
    })
}

func HandleAnalytics(ctx context.Context) error {
    return AnalyticsBulkhead.Execute(ctx, func() error {
        // Non-critical analytics
        // Can fail without impacting checkout
        return processAnalytics(ctx)
    })
}
```

### Production insight: Netflix's approach
Netflix uses "swim lanes" - separate connection pools per dependency. If the recommendation service is slow, only recommendation requests are affected. Checkout, search, and playback continue normally.

### Key insight box
> Isolation is about accepting some failures to prevent catastrophic failures. Trade local degradation for global stability.

### Challenge question
You isolate tenants into separate database shards. One tenant's query runs a table scan that locks up their shard. Other tenants are fine. Is this good isolation or bad user experience?

---

##  Circuit breakers - failing fast to prevent cascades

### Scenario
Your service calls a payment provider API. The provider is having issues:
- 50% of requests timeout (30 seconds each)
- Your service has 100 threads
- Requests are coming in at 10/sec

### Think about it
Without circuit breakers, what happens?

### Interactive question (pause and think)
How long until your service is completely unresponsive?

A. 30 seconds
B. 3 minutes
C. Never - timeouts prevent thread exhaustion
D. 10 seconds

Pause and calculate.

### Progressive reveal
Answer: D (10 seconds).

Math:
- 100 threads available
- 10 requests/sec
- 50% timeout for 30 seconds each
- 10 * 0.5 = 5 requests/sec tie up threads for 30 sec
- 5 requests/sec * 30 sec = 150 threads needed
- You only have 100 threads → saturated in 100/5 = 20 sec
- But realistically, users start seeing failures at ~80% saturation = 10 seconds

### Circuit breaker states

```text
┌─────────────────────────────────────────────────────────┐
│             CIRCUIT BREAKER STATE MACHINE               │
└─────────────────────────────────────────────────────────┘

                    ┌──────────┐
                    │  CLOSED  │  ◄── Normal operation
                    │ (normal) │       All requests go through
                    └────┬─────┘
                         │
                         │ Failure threshold exceeded
                         │ (e.g., 50% failures in 10 sec)
                         ▼
                    ┌──────────┐
         ┌──────────┤   OPEN   │  ◄── Fast-fail mode
         │          │ (failing)│       All requests rejected immediately
         │          └────┬─────┘       No backend calls
         │               │
         │               │ After timeout period
         │               │ (e.g., 30 seconds)
         │               ▼
         │          ┌──────────┐
         │          │HALF-OPEN │  ◄── Testing recovery
         │          │ (testing)│       Limited requests allowed
         │          └────┬─────┘
         │               │
         │               ├──► If requests succeed
         │               │    → CLOSED (recovered)
         │               │
         └───────────────┴──► If requests fail
                              → OPEN (still broken)

Failure examples:
  - Timeouts (after 1 sec)
  - HTTP 5xx errors
  - Connection refused
  - DNS failures

Success examples:
  - HTTP 2xx, 3xx, 4xx (4xx = client error, not circuit trip)
  - Fast responses (< timeout)
```

### Circuit breaker implementation

```go
package circuitbreaker

import (
    "errors"
    "sync"
    "time"
)

type State int

const (
    StateClosed State = iota  // Normal
    StateOpen                 // Failing fast
    StateHalfOpen            // Testing recovery
)

type CircuitBreaker struct {
    mu             sync.Mutex
    state          State
    failures       int
    successes      int
    lastFailTime   time.Time

    // Configuration
    failureThreshold int           // Open after N failures
    successThreshold int           // Close after N successes in half-open
    timeout          time.Duration // Time before half-open attempt
}

var ErrCircuitOpen = errors.New("circuit breaker is open")

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()

    // Check if we should transition to half-open
    if cb.state == StateOpen {
        if time.Since(cb.lastFailTime) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            cb.failures = 0
        } else {
            cb.mu.Unlock()
            return ErrCircuitOpen  // Fail fast
        }
    }

    cb.mu.Unlock()

    // Execute the function
    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.onFailure()
    } else {
        cb.onSuccess()
    }

    return err
}

func (cb *CircuitBreaker) onFailure() {
    cb.failures++
    cb.lastFailTime = time.Now()

    if cb.state == StateHalfOpen {
        // Failed during testing - back to open
        cb.state = StateOpen
        cb.failures = 0
        return
    }

    if cb.failures >= cb.failureThreshold {
        cb.state = StateOpen
    }
}

func (cb *CircuitBreaker) onSuccess() {
    if cb.state == StateHalfOpen {
        cb.successes++
        if cb.successes >= cb.successThreshold {
            // Recovery confirmed
            cb.state = StateClosed
            cb.failures = 0
        }
    }

    if cb.state == StateClosed {
        // Reset failure count on success
        cb.failures = 0
    }
}

// Usage
var paymentProviderCB = &CircuitBreaker{
    failureThreshold: 5,
    successThreshold: 2,
    timeout:          30 * time.Second,
}

func CallPaymentProvider() error {
    return paymentProviderCB.Call(func() error {
        // Actual payment API call
        resp, err := http.Get("https://payments.example.com/charge")
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        if resp.StatusCode >= 500 {
            return errors.New("server error")
        }
        return nil
    })
}
```

### Real-world failure: No circuit breaker

```text
Scenario: E-commerce site, payment gateway times out

Without circuit breaker:
  00:00 - Gateway starts timing out (30s per request)
  00:10 - All app threads stuck waiting
  00:15 - Health checks start failing (no threads available)
  00:16 - Load balancer removes instances
  00:17 - Traffic shifts to remaining instances
  00:18 - Remaining instances also saturate
  00:20 - Full outage

With circuit breaker:
  00:00 - Gateway starts timing out
  00:05 - Circuit opens after 5 failures
  00:06 - Requests fail immediately with "payment unavailable"
  00:35 - Circuit enters half-open, tests recovery
  00:36 - Gateway recovered, circuit closes

Impact difference:
  - Without: 20 min full outage
  - With: 30 sec degradation (payment only), graceful fallback
```

### Key insight box
> Circuit breakers don't prevent failures - they prevent cascading failures. Fail fast to stay available.

### Challenge question
Your circuit breaker opens. How do you communicate this to users: "Payment temporarily unavailable" or "Internal server error"?

---

##  Cell-based architecture - the ultimate isolation

### Scenario
You run a global SaaS platform. A bad deployment takes down your entire us-west region.

Impact: 40% of customers offline.

### Interactive question (pause and think)
How can you deploy updates without risking 40% of your customers?

A. Better testing (catch bugs before production)
B. Canary deployments (gradually roll out)
C. Cell architecture (customers isolated into groups)

### Progressive reveal
Answer: All three, but C provides the strongest isolation guarantee.

### What is cell-based architecture?

**Traditional architecture:**
- All customers share the same infrastructure pool
- One bad deployment affects everyone
- Load balancer sprays traffic across all instances

**Cell-based architecture:**
- Customers assigned to isolated "cells" (separate clusters)
- Each cell is a complete, independent stack
- Cells can fail independently

### Visual: monolithic vs cell-based

```text
MONOLITHIC POOL:
┌───────────────────────────────────────────────┐
│  Load Balancer                                │
│     │  │  │  │  │  │                          │
│     ▼  ▼  ▼  ▼  ▼  ▼                          │
│  ┌────────────────────────────┐               │
│  │  Shared Instance Pool      │               │
│  │  (All customers mixed)     │               │
│  │                            │               │
│  │  [CustomerA, CustomerB,    │               │
│  │   CustomerC, ...]          │               │
│  │                            │               │
│  └────────────────────────────┘               │
│     │                                         │
│     ▼                                         │
│  ┌────────────────┐                          │
│  │ Shared Database│                          │
│  └────────────────┘                          │
│                                               │
│  BLAST RADIUS: 100% customers                │
└───────────────────────────────────────────────┘

CELL-BASED:
┌───────────────────────────────────────────────┐
│  Global Router (routes by customer ID)        │
│     │        │         │         │            │
│     ▼        ▼         ▼         ▼            │
│  Cell-1   Cell-2   Cell-3   Cell-4           │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐         │
│  │Cust │  │Cust │  │Cust │  │Cust │         │
│  │A,B,C│  │D,E,F│  │G,H,I│  │J,K,L│         │
│  │     │  │     │  │     │  │     │         │
│  │ DB  │  │ DB  │  │ DB  │  │ DB  │         │
│  └─────┘  └─────┘  └─────┘  └─────┘         │
│                                               │
│  BLAST RADIUS: 25% customers (one cell)      │
└───────────────────────────────────────────────┘

Deployment strategy:
  1. Deploy to Cell-1 (canary)
  2. Monitor for 1 hour
  3. If healthy, deploy to Cell-2
  4. If healthy, deploy to Cell-3, Cell-4

If Cell-1 fails:
  - 25% customers impacted
  - Rollback Cell-1 only
  - 75% customers never affected
```

### Cell design considerations

**Cell sizing:**
- **Too small** (10 customers/cell): high overhead, many cells to manage
- **Too large** (10,000 customers/cell): large blast radius
- **Sweet spot**: 100-1,000 customers/cell (1-10% blast radius)

**Customer assignment:**
- **By ID hash**: evenly distributed load
- **By tier**: Enterprise customers in dedicated cells
- **By geography**: comply with data residency
- **By feature usage**: High-load customers in separate cells

**Deployment strategy:**
```yaml
deployment:
  canary_cell: cell-1
  canary_duration: 2h

  rollout:
    - stage: 1
      cells: [cell-1]
      wait: 2h

    - stage: 2
      cells: [cell-2, cell-3]
      wait: 1h

    - stage: 3
      cells: [cell-4, cell-5, cell-6, cell-7]
      wait: 30m

    - stage: 4
      cells: [remaining]

  rollback_triggers:
    - error_rate > 1%
    - latency_p99 > 500ms
    - customer_complaints > 10
```

### Production insight: AWS's use of cells

AWS uses "shuffle sharding" - each customer's requests are routed to a unique subset of cells. Even if one cell fails, most customers' other requests go to healthy cells.

Effective blast radius: much less than 1/N where N = number of cells.

### Key insight box
> Cell-based architecture is the gold standard for blast radius isolation. Cost: operational complexity and reduced resource sharing efficiency.

### Challenge question
You have 1000 servers. Monolithic pool vs 10 cells of 100 servers each. Which design can handle more total load? Which is more resilient?

---

##  The hidden cost of isolation - operational complexity

### Scenario
Your SRE team is thrilled. You've implemented:
- Cell-based architecture (10 cells)
- Separate database clusters per cell
- Separate Kubernetes clusters per cell
- Circuit breakers on all dependencies

Then your CEO asks: "Why did our AWS bill triple?"

### Interactive question (pause and think)
Isolation costs money. Where does the cost come from?

A. Duplicate infrastructure (less efficient sharing)
B. Operational overhead (more things to manage)
C. Reduced economies of scale
D. All of the above

### Progressive reveal
Answer: D - isolation trades efficiency for resilience.

### Hidden costs of isolation

**1. Infrastructure duplication:**
```
Monolithic:
  - 1 database cluster (10 instances, high utilization)
  - 1 Kubernetes cluster
  - Total: $50K/month

Cell-based (10 cells):
  - 10 database clusters (100 instances total, lower per-instance utilization)
  - 10 Kubernetes clusters
  - Total: $120K/month (2.4x cost for same throughput)
```

**2. Operational overhead:**
- 10x more things to monitor
- 10x more things to upgrade
- 10x more configuration management
- 10x more toil for runbooks

**3. Cross-cell operations:**
- Customer wants to query all their data → scatter-gather across cells
- Analytics pipeline needs data from all cells → complex fan-in
- Global rate limiting → requires cross-cell coordination

**4. Reduced caching efficiency:**
```
Monolithic cache:
  - 1 Redis cluster, 1TB total
  - Cache hit rate: 85%

Cell-based cache:
  - 10 Redis clusters, 100GB each
  - Cache hit rate: 60% (data not shared)
  - Increased DB load
```

### When isolation is worth it

**DO heavily isolate:**
- Regulated industries (finance, healthcare)
- Multi-tenant SaaS with enterprise SLAs
- Systems with catastrophic failure costs (life-critical, high revenue)

**DON'T over-isolate:**
- Early-stage startups (premature optimization)
- Internal tools (lower availability requirements)
- Read-heavy systems with low mutation rate

### Visual: isolation cost/benefit curve

```text
        │
Benefit │     ╱─────────  (diminishing returns)
(Resil- │    ╱
ience)  │   ╱
        │  ╱
        │ ╱_______________
        └─────────────────────►
               Cost (complexity + $)

Sweet spot: somewhere in the middle

Over-isolated:
  - 100 cells for 100 customers
  - Separate infrastructure per request
  - Gold-plated resilience, bankruptcy bills

Under-isolated:
  - Single global shared-everything
  - Efficient until one failure takes down everything
  - Cost-effective until it's catastrophic
```

### Key insight box
> Isolation is not free. The art is finding the minimum isolation that meets your resilience and compliance requirements.

### Challenge question
How would you measure if you're over-isolated or under-isolated? What metrics indicate the right balance?

---

##  Quota management - isolating greedy neighbors

### Scenario
You run a multi-tenant API platform. CustomerA is running a badly written script that hammers your API:
- 1000 requests/sec (10x normal)
- 80% of your compute capacity
- Other customers seeing latency spikes

Without quotas, one customer destroys the experience for everyone.

### Think about it
How do you prevent this without manually blocking CustomerA?

### Interactive question (pause and think)
Which quota strategy is most fair?

A. Hard limit (1000 req/sec per customer, block excess)
B. Rate limiting (X req/sec, queue excess with timeout)
C. Fair queuing (each customer gets equal share of capacity)
D. Tiered quotas (enterprise gets more than free tier)

### Progressive reveal
Answer: Depends on product requirements, but typically D with B.

### Quota implementation patterns

**1. Token bucket (smooth rate limiting)**
```go
type TokenBucket struct {
    capacity     int       // Max burst
    refillRate   int       // Tokens per second
    tokens       int       // Current available
    lastRefill   time.Time
    mu           sync.Mutex
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    // Refill tokens based on time elapsed
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    newTokens := int(elapsed * float64(tb.refillRate))

    tb.tokens = min(tb.capacity, tb.tokens + newTokens)
    tb.lastRefill = now

    if tb.tokens > 0 {
        tb.tokens--
        return true
    }

    return false
}

// Usage: per-customer limits
var customerQuotas = map[string]*TokenBucket{
    "customer-A": {capacity: 100, refillRate: 10},  // 10/sec, burst to 100
    "customer-B": {capacity: 1000, refillRate: 100}, // Enterprise tier
}
```

**2. Fair queuing (weighted fair share)**
```go
type FairQueue struct {
    queues map[string]chan Request
    weights map[string]int  // Higher = more share
}

func (fq *FairQueue) Schedule() Request {
    // Weighted round-robin
    // Customer with weight=10 gets 10x more CPU than weight=1
    for {
        for customerID, weight := range fq.weights {
            queue := fq.queues[customerID]

            // Dequeue up to 'weight' requests
            for i := 0; i < weight; i++ {
                select {
                case req := <-queue:
                    return req
                default:
                    break
                }
            }
        }
    }
}
```

**3. Quota enforcement layers**
```text
Request flow with quota enforcement:

Client Request
    │
    ▼
┌─────────────────┐
│ API Gateway     │
│ Rate Limit:     │ ◄── Layer 1: Protect infrastructure
│ 10K req/sec     │     (DDoS defense)
│ per IP          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Auth Layer      │
│ Identify        │ ◄── Layer 2: Identify customer
│ Customer ID     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Quota Manager   │
│ Per-customer    │ ◄── Layer 3: Fair resource allocation
│ limits based    │
│ on tier         │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Service         │
│ Process request │ ◄── Layer 4: Process if quota allows
└─────────────────┘
```

### Production insight: Stripe's quota approach
- Free tier: 100 requests/sec
- Growth tier: 1,000 requests/sec
- Enterprise: custom, but still rate-limited (prevents accidents)
- Burst allowance: 2x sustained rate for 60 seconds
- Graceful degradation: return HTTP 429 with Retry-After header

### Key insight box
> Quotas are isolation for time-based resources. Without them, one customer's spike becomes everyone's outage.

### Challenge question
A customer hits their quota at 11:59 PM. Their critical business process runs at midnight. Should your system allow burst overages for brief periods?

---

##  Measuring blast radius - knowing your exposure

### Scenario
Your VP asks: "If our payment service goes down, how many customers are affected and for how long?"

You don't know. You've never measured it.

### Interactive question (pause and think)
Which metric best represents blast radius?

A. Number of servers impacted
B. Percentage of customer requests failing
C. Revenue lost per minute
D. Time to detect + time to recover

### Progressive reveal
Answer: B and C together, adjusted by D.

Blast radius = (% customers impacted) × (revenue impact rate) × (downtime duration)

### Blast radius measurement framework

**Pre-incident (design time):**
```yaml
service: payment-processor
blast_radius_analysis:

  failure_mode: "Service completely down"
  customer_impact:
    percentage: 100%  # All customers use payments
    features_affected: [checkout, subscriptions, refunds]

  revenue_impact:
    lost_revenue_per_minute: $50,000
    reputation_damage: HIGH

  mttr_estimate: 15 minutes

  total_blast_radius:
    max_revenue_loss: $750,000  # 15 min * $50K
    customer_count: 1,000,000

  mitigation_strategies:
    - Multi-provider fallback (reduces to 10% impact)
    - Circuit breaker (reduces MTTR to 2 min)
    - Regional isolation (reduces to 30% customers)

  revised_blast_radius:
    impact: 30% customers
    mttr: 2 minutes
    max_revenue_loss: $3,000  # Much better
```

**During-incident (real-time monitoring):**
```go
type BlastRadiusMonitor struct {
    StartTime      time.Time
    ImpactedCustomers []CustomerID
    FailedRequests  int
    TotalRequests   int
}

func (br *BlastRadiusMonitor) CurrentImpact() BlastRadiusReport {
    elapsed := time.Since(br.StartTime)
    errorRate := float64(br.FailedRequests) / float64(br.TotalRequests)

    return BlastRadiusReport{
        Duration:           elapsed,
        ErrorRate:          errorRate,
        CustomersImpacted:  len(br.ImpactedCustomers),
        EstimatedRevenueLoss: br.calculateRevenueLoss(),
    }
}

// Alert if blast radius exceeds threshold
func (br *BlastRadiusMonitor) CheckThresholds() {
    if br.CurrentImpact().ErrorRate > 0.05 {
        alert("Blast radius exceeds 5% error rate - trigger incident response")
    }

    if br.CurrentImpact().Duration > 5*time.Minute {
        alert("Blast radius duration exceeds 5 min - escalate to senior team")
    }
}
```

**Post-incident (retrospective):**
```text
Blast Radius Post-Mortem:

Incident: Payment service timeout cascade
Date: 2024-12-15
Duration: 23 minutes

Impact Analysis:
├─ Customer Impact:
│  ├─ Total customers: 1,000,000
│  ├─ Impacted customers: 340,000 (34%)
│  ├─ Geographic distribution:
│  │  ├─ US-EAST: 100% impact
│  │  ├─ US-WEST: 0% impact (isolated)
│  │  └─ EU: 20% impact (partial cascade)
│  └─ Feature degradation:
│     ├─ Checkout: 100% failed
│     ├─ Subscription renewal: 80% failed
│     └─ Browse/search: 0% impact (isolated)
│
├─ Business Impact:
│  ├─ Revenue lost: $1,150,000
│  ├─ Customer support tickets: 4,200
│  └─ Reputation: trending #3 on Twitter
│
├─ Technical Impact:
│  ├─ Services affected: 5 (payment, order, shipping, analytics, notification)
│  ├─ Databases degraded: 2 (orders DB, payments DB)
│  └─ Infrastructure: 60% CPU saturation on web tier
│
└─ Recovery Timeline:
   ├─ Detection: 4 minutes (too slow - alert tuning needed)
   ├─ Mitigation: 8 minutes (circuit breaker manual trigger)
   ├─ Full recovery: 23 minutes (database connection pool recovery)
   └─ MTTR target: 5 minutes (MISSED)

Blast Radius Reduction Opportunities:
1. Regional isolation prevented EU-WEST and US-WEST impact ✓
2. Circuit breakers reduced cascade to analytics/notification ✓
3. MISSED: Should have failed fast at 1 min instead of 4 min
4. MISSED: US-EAST should have been a smaller blast radius cell

Action Items:
[ ] Implement automated circuit breaker (remove manual trigger)
[ ] Split US-EAST into 3 cells
[ ] Improve detection (alert on slow requests, not just failures)
[ ] Add chaos testing for timeout scenarios
```

### Key insight box
> You can't reduce what you don't measure. Blast radius measurement must happen at design time, incident time, and retrospective time.

### Challenge question
Two services: Service A has 10% error rate for 1 hour. Service B has 1% error rate for 10 hours. Which has a bigger blast radius?

---

##  Final synthesis - Design a resilient multi-tenant platform

### Synthesis challenge
You're the architect for a new B2B SaaS analytics platform.

Requirements:
- 10,000 customers (range: 10-employee startups to 50,000-employee enterprises)
- 99.95% uptime SLA (22 min downtime/month)
- Must support real-time and batch analytics
- Compliance: some customers require data residency (EU, US)
- Cost-sensitive: investors want efficient infrastructure spend

Constraints:
- Limited ops team (5 SREs)
- Can't afford 10,000 isolated cells

### Your tasks (pause and think)
1. Design your isolation strategy (what layers, what technique?)
2. Define failure domains (what should fail together vs separately?)
3. Choose cell sizing and customer assignment strategy
4. Define blast radius SLO per failure mode
5. Plan quota management approach
6. Describe how you measure and improve blast radius over time

Write down your design.

### Progressive reveal (one possible solution)

**1. Isolation strategy (layered):**
- **Tier 1**: Geography (US, EU separate regions for data residency)
- **Tier 2**: Customer size (Enterprise in dedicated cells, SMB shared)
- **Tier 3**: Feature isolation (real-time vs batch separate compute)
- **Tier 4**: Circuit breakers on all cross-service calls

**2. Failure domains:**
| Domain | Isolation Level | Blast Radius |
|--------|----------------|--------------|
| Enterprise customer data | Dedicated cell | 1 customer |
| SMB customer data | Shared cell (100 customers) | 1% customers |
| Real-time ingestion | Separate from batch | 50% features |
| Batch processing | Separate from real-time | 50% features |
| Authentication service | Regional (US, EU) | 50% customers |

**3. Cell sizing:**
- Enterprise tier: 1 customer per cell (largest 50 customers)
- Growth tier: 100 customers per cell (next 950 customers)
- Startup tier: 1000 customers per cell (remaining 9,000 customers)
- Total: 50 + 10 + 9 = 69 cells (manageable by 5 SREs)

**4. Blast radius SLOs:**
```yaml
blast_radius_slos:

  single_cell_failure:
    max_customer_impact: 1% (100 customers)
    max_duration: 5 minutes
    recovery: automated

  regional_failure:
    max_customer_impact: 50% (one region)
    max_duration: 15 minutes
    recovery: failover to other region

  dependency_timeout:
    max_feature_impact: "analytics dashboard only"
    max_customer_impact: 100% (graceful degradation)
    recovery: circuit breaker auto-opens

  data_corruption:
    max_customer_impact: 1 customer (isolated cell)
    recovery: restore from backup
```

**5. Quota management:**
- Token bucket per customer (tier-based rates)
- Enterprise: 1000 req/sec sustained, 5000 burst
- Growth: 100 req/sec sustained, 500 burst
- Startup: 10 req/sec sustained, 50 burst
- Fair queuing within shared cells
- HTTP 429 with Retry-After for quota exceeded

**6. Blast radius improvement plan:**
```
Month 1-2: Baseline
  - Instrument blast radius metrics on all services
  - Run 3 Game Days (database failure, dependency timeout, cell crash)
  - Measure actual vs designed blast radius

Month 3-4: Isolate
  - Implement circuit breakers on top 10 dependencies
  - Add regional failover capability
  - Split largest shared cell into 2 cells

Month 5-6: Automate
  - Auto-scaling per cell
  - Automated circuit breaker tuning
  - Chaos engineering (automated weekly tests)

Ongoing:
  - Monthly Game Day (different scenario each time)
  - Quarterly blast radius review (adjust cell sizing)
  - Continuous monitoring and alerting improvements
```

### Key insight box
> Blast radius design is a balancing act: isolation vs cost, resilience vs complexity, safety vs efficiency. The right answer depends on your SLAs and risk tolerance.

### Final challenge question
Your design achieves 99.95% uptime (22 min downtime/month). Should you over-engineer for 99.99% (4 min/month) or invest those resources elsewhere? How do you make that trade-off?

---

## Appendix: Quick checklist (printable)

**Blast radius design checklist:**
- [ ] Identify failure modes (enumerate what can go wrong)
- [ ] Define failure domains (what should fail together?)
- [ ] Choose isolation techniques (bulkheads, cells, circuit breakers)
- [ ] Size cells appropriately (balance blast vs ops overhead)
- [ ] Implement quotas (prevent greedy neighbors)
- [ ] Add circuit breakers (prevent cascades)
- [ ] Define blast radius SLOs (measure what matters)

**Operational checklist:**
- [ ] Monitor blast radius in real-time during incidents
- [ ] Run Game Days to validate isolation design
- [ ] Measure actual blast radius vs designed limits
- [ ] Track isolation costs (infra $ + operational overhead)
- [ ] Review and adjust quarterly (as system evolves)

**Circuit breaker checklist:**
- [ ] Timeout threshold (e.g., 1 second)
- [ ] Failure threshold (e.g., 50% error rate over 10 sec)
- [ ] Half-open test period (e.g., 30 seconds)
- [ ] Success threshold to close (e.g., 3 consecutive successes)
- [ ] Metrics and dashboards (state changes, trip events)
- [ ] Alerts on circuit open (human awareness)

**Cell-based architecture checklist:**
- [ ] Cell size determined (customer count or load-based)
- [ ] Customer assignment strategy (hash, tier, geo)
- [ ] Deployment strategy (canary cell, staged rollout)
- [ ] Cross-cell operations minimized (data locality)
- [ ] Observability per cell (separate dashboards/alerts)
- [ ] Runbooks for cell-specific operations

**Red flags (reassess isolation):**
- [ ] Same failure keeps taking down multiple cells
- [ ] Operational overhead exceeds team capacity
- [ ] Infrastructure costs 3x+ comparable monolithic design
- [ ] Cross-cell operations are frequent and slow
- [ ] Isolation is preventing needed features
