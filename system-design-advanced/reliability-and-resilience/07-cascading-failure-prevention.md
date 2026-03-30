---
slug: cascading-failure-prevention
title: Cascading Failure Prevention
readTime: 20 min
orderIndex: 6
premium: false
---




# Cascading Failure Prevention (Stop the Domino Effect)

> Audience: system architects and SREs building resilient distributed systems that must survive partial failures.

This article assumes:
- Failures in distributed systems are inevitable and often correlated.
- A small failure can trigger a catastrophic chain reaction if unchecked.
- Your system's weakest link will be discovered during your busiest hour.
- Prevention is cheaper than recovery.

---

## Challenge: One microservice takes down your entire platform

### Scenario
Black Friday, 10 AM. Your recommendation service has a memory leak.

Timeline:
- **10:00**: Recommendation service starts running out of memory
- **10:02**: Service slows down (GC thrashing)
- **10:03**: API gateway timeouts waiting for recommendations
- **10:04**: API gateway connection pool fills up
- **10:05**: All API requests fail (checkout, search, login - everything)
- **10:07**: Full platform outage

Your entire e-commerce site down because of a non-critical recommendation feature.

### Interactive question (pause and think)
What failed here?

1. The recommendation service (it had the bug)
2. The API gateway (it couldn't handle timeouts)
3. The system architecture (failure cascaded)
4. The monitoring (it didn't alert fast enough)

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (3) - architecture failure.

Bugs happen. The system should contain them, not amplify them.

### Real-world analogy (forest fire)
A campfire in a forest:

**Without firebreaks**: Small campfire → spreads to tree → spreads to neighboring trees → entire forest burns.

**With firebreaks**: Small campfire → spreads to tree → hits firebreak (cleared area) → fire contained to small section.

Your distributed system needs firebreaks.

### Key insight box
> Cascading failures happen when one component's failure propagates to healthy components, creating a domino effect that exceeds the original blast radius.

### Challenge question
Name three architectural patterns that would have prevented this cascade. Can you implement all three without significantly increasing latency?

---

## Mental model - Failures amplify through dependency chains

### Scenario
Your system has a dependency graph:

```
User → Frontend → API Gateway → Service A → Service B → Database
```

Service B slows down. Who gets affected?

### Interactive question (pause and think)
If Service B starts taking 10 seconds instead of 100ms, what happens upstream?

A. Only Service A is affected (direct dependency)
B. API Gateway, Service A affected (upstream dependencies)
C. Everything upstream fails (Frontend, API Gateway, Service A)
D. Even unrelated services fail (shared resources)

### Progressive reveal
Answer: Often C or D.

### Mental model
Think of cascading failures as:

- **Dependency amplification**: One slow service makes everything slow
- **Resource exhaustion**: Waiting threads consume memory/connections
- **Retry storms**: Failed requests get retried, making overload worse
- **Shared fate**: Components share infrastructure (thread pools, databases)

The cascade pattern:
```
Slow service → Thread exhaustion → Backpressure → Queue overflow →
Connection pool exhaustion → Load balancer gives up → Full outage
```

### Real-world parallel (traffic jam)
One slow car on a single-lane road:

1. Car slows to 20 mph
2. Cars behind it slow down (backpressure)
3. More cars arrive than can pass (queue builds)
4. Queue grows until blocking the highway entrance
5. Now the entire highway system is gridlocked

One slow component can gridlock an entire system.

### Key insight box
> Cascading failures exploit dependencies and shared resources. The failure propagates faster than you can respond.

### Challenge question
Can you design a system where Service B's failure has ZERO impact on Service A? What would you have to sacrifice?

---

## Understanding cascade amplification factors

### Scenario
Service B handles 100 requests/second normally.

It degrades to 10 requests/second (10x slower).

How much does this affect upstream services?

### Amplification factors

**Factor 1: Retry amplification**
```yaml
scenario:
  service_b_capacity: 100 req/sec
  service_b_degraded: 10 req/sec (90% failing)

  without_retries:
    failures: 90 req/sec fail
    impact: 90% error rate to users

  with_3_retries:
    - Initial request: 100 req/sec
    - 90 fail → retry: +90 req/sec
    - 81 fail (90% of 90) → retry: +81 req/sec
    - 73 fail → retry: +73 req/sec

    total_load: 100 + 90 + 81 + 73 = 344 req/sec
    service_b_receives: 344 req/sec (was only 100)
    impact: 3.4x amplification

  result: Retries make the overload WORSE
```

**Factor 2: Timeout amplification**
```yaml
scenario:
  service_a_threads: 100
  service_b_latency: 10 seconds (was 100ms)
  request_rate: 10 req/sec

  calculation:
    - Each request holds thread for 10 seconds
    - 10 req/sec × 10 seconds = 100 threads needed
    - Thread pool exhausted

  result:
    - After 10 seconds, all threads stuck waiting
    - Service A can't handle ANY requests
    - 10x slower service causes 100% outage upstream
```

**Factor 3: Connection pool amplification**
```yaml
scenario:
  api_gateway_connections: 1000 max
  service_b_slow: 5 seconds per request (was 100ms)

  normal:
    requests_per_sec: 100
    avg_connection_time: 100ms
    concurrent_connections: 100 × 0.1 = 10

  degraded:
    requests_per_sec: 100 (same demand)
    avg_connection_time: 5 seconds
    concurrent_connections: 100 × 5 = 500

  after_2_seconds:
    concurrent_connections: 1000 (pool exhausted)
    new_requests: fail immediately

  result: 50x slower service exhausts connection pool
```

### Visual: cascade timeline

```text
┌─────────────────────────────────────────────────────────┐
│           CASCADING FAILURE TIMELINE                    │
└─────────────────────────────────────────────────────────┘

T+0 sec:  Service B degrades (memory leak, slow queries)
          └─ Latency: 100ms → 2 seconds

T+5 sec:  Service A thread pool fills up
          └─ All 100 threads waiting for Service B

T+10 sec: Service A stops responding
          └─ Health checks start failing

T+15 sec: API Gateway times out on Service A
          └─ Connection pool starts filling

T+20 sec: API Gateway connection pool exhausted
          └─ Can't accept new connections

T+25 sec: Load balancer marks API Gateway unhealthy
          └─ Removes from rotation

T+30 sec: Traffic shifts to remaining API Gateway instances
          └─ Overload spreads to healthy instances

T+35 sec: All API Gateway instances failing
          └─ Full platform outage

T+40 sec: Database overwhelmed by connection attempts
          └─ Cascading failure reaches data layer

Impact:
  - Started with 1 slow service (Service B)
  - Ended with full platform outage
  - Cascade took only 40 seconds
  - Affected services that don't even depend on Service B
```

### Key insight box
> Cascading failures amplify through retries, timeouts, and resource exhaustion. A 10x slowdown can cause a 100% outage.

### Challenge question
If retries make cascades worse, should you disable retries entirely? What's the right retry strategy?

---

## Core cascade prevention patterns

### Scenario
You're designing a system that must survive partial failures.

What patterns prevent cascades?

### Pattern 1: Timeouts (fail fast)

```go
package cascadeprevention

import (
    "context"
    "net/http"
    "time"
)

// BAD: No timeout, waits forever
func CallServiceBad(url string) (*http.Response, error) {
    return http.Get(url)
}

// GOOD: Timeout prevents thread exhaustion
func CallServiceGood(url string) (*http.Response, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    return http.DefaultClient.Do(req)
}

// Timeout configuration per dependency
var timeouts = map[string]time.Duration{
    "service-b":       1 * time.Second,  // Critical path
    "recommendations": 500 * time.Millisecond, // Non-critical
    "analytics":       100 * time.Millisecond, // Best-effort
}
```

**Timeout best practices:**
```yaml
timeout_strategy:

  per_service_timeouts:
    critical_path: 1-2 seconds
    non_critical: 200-500ms
    best_effort: 100ms or less

  timeout_hierarchy:
    client_timeout: 5 seconds
    api_gateway_timeout: 4 seconds (< client)
    service_a_timeout: 3 seconds (< gateway)
    service_b_timeout: 2 seconds (< service_a)

    reason: Each layer times out before upstream layer

  dynamic_timeouts:
    - Measure p99 latency
    - Set timeout = p99 + buffer
    - Adjust based on recent performance
```

### Pattern 2: Circuit breakers (stop trying)

```go
package circuitbreaker

import (
    "errors"
    "sync"
    "time"
)

type CircuitBreaker struct {
    maxFailures  int
    resetTimeout time.Duration

    failures     int
    lastFailTime time.Time
    state        State
    mu           sync.Mutex
}

type State int

const (
    StateClosed   State = iota // Normal: allow all requests
    StateOpen                   // Failing: reject all requests
    StateHalfOpen              // Testing: allow limited requests
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()

    // Check if we should transition to half-open
    if cb.state == StateOpen {
        if time.Since(cb.lastFailTime) > cb.resetTimeout {
            cb.state = StateHalfOpen
            cb.failures = 0
        } else {
            cb.mu.Unlock()
            return ErrCircuitOpen // Fail fast without calling service
        }
    }

    cb.mu.Unlock()

    // Call the actual service
    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()

        if cb.failures >= cb.maxFailures {
            cb.state = StateOpen // Open circuit
        }
    } else {
        if cb.state == StateHalfOpen {
            cb.state = StateClosed // Recovery confirmed
        }
        cb.failures = 0
    }

    return err
}
```

**Circuit breaker benefits:**
```yaml
without_circuit_breaker:
  scenario: Service B is down
  behavior:
    - Service A keeps calling Service B
    - Every call times out (1 second)
    - 100 requests/sec × 1 second = 100 threads blocked
    - Thread pool exhausted

  result: Service A fails even though it's healthy

with_circuit_breaker:
  scenario: Service B is down
  behavior:
    - First 5 requests fail (open circuit)
    - Circuit opens
    - Subsequent requests fail immediately (< 1ms)
    - Threads not blocked

  result: Service A stays healthy, returns errors but doesn't cascade
```

### Pattern 3: Bulkheads (isolate resources)

```go
package bulkhead

import (
    "context"
    "errors"
)

// Separate thread pools per dependency
type Bulkhead struct {
    semaphore chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        semaphore: make(chan struct{}, maxConcurrent),
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.semaphore <- struct{}{}:
        defer func() { <-b.semaphore }()
        return fn()
    case <-ctx.Done():
        return ctx.Err()
    default:
        return errors.New("bulkhead full")
    }
}

// Per-service bulkheads
var (
    ServiceBBulkhead = NewBulkhead(50)  // Max 50 concurrent calls to Service B
    DatabaseBulkhead = NewBulkhead(100) // Max 100 concurrent DB queries
    CacheBulkhead    = NewBulkhead(200) // Max 200 concurrent cache calls
)

func CallServiceB() error {
    return ServiceBBulkhead.Execute(context.Background(), func() error {
        // Call Service B
        return nil
    })
}
```

**Bulkhead benefits:**
```yaml
without_bulkheads:
  scenario: Service B slow, Database healthy
  shared_thread_pool: 100 threads total

  result:
    - Service B consumes all 100 threads
    - Database calls can't get threads
    - Everything fails

with_bulkheads:
  scenario: Service B slow, Database healthy
  service_b_bulkhead: 50 threads
  database_bulkhead: 50 threads

  result:
    - Service B consumes its 50 threads
    - Database still has 50 threads available
    - Database operations continue working
```

### Pattern 4: Load shedding (reject when overloaded)

```go
package loadshedding

import (
    "errors"
    "sync/atomic"
)

type LoadShedder struct {
    maxConcurrent int64
    current       int64
}

func NewLoadShedder(max int64) *LoadShedder {
    return &LoadShedder{maxConcurrent: max}
}

var ErrOverloaded = errors.New("service overloaded, request rejected")

func (ls *LoadShedder) Execute(fn func() error) error {
    current := atomic.LoadInt64(&ls.current)
    if current >= ls.maxConcurrent {
        return ErrOverloaded // Reject immediately, don't queue
    }

    atomic.AddInt64(&ls.current, 1)
    defer atomic.AddInt64(&ls.current, -1)

    return fn()
}
```

**Load shedding strategies:**
```yaml
strategy_1_fixed_threshold:
  if concurrent_requests > 1000:
    reject_new_requests(http_503_service_unavailable)

strategy_2_adaptive:
  target_latency: 200ms
  current_latency: measure_p99()

  if current_latency > target_latency * 2:
    reject_percentage: 50%
  elif current_latency > target_latency * 1.5:
    reject_percentage: 25%

strategy_3_priority_based:
  tiers:
    critical: always accept (payments, checkout)
    normal: accept if capacity available
    best_effort: accept only if < 50% capacity
```

### Pattern 5: Backpressure (slow down upstream)

```go
package backpressure

import (
    "context"
    "time"
)

// Token bucket for rate limiting
type TokenBucket struct {
    capacity int
    tokens   chan struct{}
}

func NewTokenBucket(capacity int, refillRate time.Duration) *TokenBucket {
    tb := &TokenBucket{
        capacity: capacity,
        tokens:   make(chan struct{}, capacity),
    }

    // Fill bucket initially
    for i := 0; i < capacity; i++ {
        tb.tokens <- struct{}{}
    }

    // Refill periodically
    go func() {
        ticker := time.NewTicker(refillRate)
        for range ticker.C {
            select {
            case tb.tokens <- struct{}{}:
            default:
                // Bucket full
            }
        }
    }()

    return tb
}

func (tb *TokenBucket) Wait(ctx context.Context) error {
    select {
    case <-tb.tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**Backpressure patterns:**
```yaml
pattern_1_queue_with_limit:
  queue_size: 1000

  if queue_full:
    reject_new_requests()
  else:
    enqueue_request()

  benefit: Bounded memory usage

pattern_2_exponential_backoff:
  attempt: 1
  wait_time: 100ms

  on_failure:
    wait_time *= 2  # 100ms → 200ms → 400ms → 800ms
    retry_after(wait_time)

  benefit: Gives downstream time to recover
```

### Key insight
> Cascade prevention requires multiple defenses: timeouts (fail fast), circuit breakers (stop trying), bulkheads (isolate), load shedding (reject), and backpressure (slow down).

### Challenge question
You implement all five patterns. Your system is now "failure-proof." What happens when all regions fail simultaneously (correlated failure)?

---

## Retry storms - when fixing makes it worse

### Scenario
Service B is overloaded. Clients start seeing errors.

Clients retry. Now Service B gets 2x traffic (original + retries).

Service B falls over completely.

### Think about it
Retries are supposed to help with transient failures. Why do they make overload worse?

### Interactive question (pause and think)
Which retry strategy prevents retry storms?

A. Retry immediately on failure
B. Retry with exponential backoff
C. Retry with jitter (randomization)
D. Don't retry on overload errors (503)

### Progressive reveal
Answer: B, C, and D together.

### Retry storm amplification

```text
WITHOUT PROPER RETRY STRATEGY:

Service capacity: 100 req/sec
Normal load: 100 req/sec (at capacity)

Service degrades:
  ├─ 50 req/sec succeed
  └─ 50 req/sec fail

Clients retry immediately:
  ├─ 50 failed requests retry
  └─ +100 new requests (still coming in)

Load on service: 50 (in-flight) + 50 (retries) + 100 (new) = 200 req/sec

Service now at 200% capacity → more failures → more retries

Result: Exponential cascade
```

### Safe retry strategies

**Strategy 1: Exponential backoff with jitter**
```go
package retry

import (
    "math"
    "math/rand"
    "time"
)

func RetryWithBackoff(fn func() error, maxAttempts int) error {
    var err error

    for attempt := 0; attempt < maxAttempts; attempt++ {
        err = fn()
        if err == nil {
            return nil
        }

        if attempt == maxAttempts-1 {
            break
        }

        // Exponential backoff: 100ms, 200ms, 400ms, 800ms...
        backoff := time.Duration(math.Pow(2, float64(attempt))) * 100 * time.Millisecond

        // Add jitter to prevent thundering herd
        jitter := time.Duration(rand.Int63n(int64(backoff / 2)))

        wait := backoff + jitter
        time.Sleep(wait)
    }

    return err
}
```

**Strategy 2: Don't retry on overload**
```go
func ShouldRetry(err error) bool {
    // Don't retry if service is overloaded
    if errors.Is(err, ErrServiceOverloaded) {
        return false
    }

    // Don't retry on client errors (4xx)
    if errors.Is(err, ErrBadRequest) {
        return false
    }

    // Retry on transient errors (timeouts, network issues)
    if errors.Is(err, ErrTimeout) || errors.Is(err, ErrNetworkFailure) {
        return true
    }

    return false
}
```

**Strategy 3: Retry budget**
```go
type RetryBudget struct {
    maxRetries       int
    retriesUsed      int
    windowStart      time.Time
    windowDuration   time.Duration
}

func (rb *RetryBudget) CanRetry() bool {
    // Reset window if expired
    if time.Since(rb.windowStart) > rb.windowDuration {
        rb.windowStart = time.Now()
        rb.retriesUsed = 0
    }

    // Check if budget allows retry
    if rb.retriesUsed >= rb.maxRetries {
        return false // Retry budget exhausted
    }

    rb.retriesUsed++
    return true
}

// Usage: max 100 retries per minute
var budget = &RetryBudget{
    maxRetries:     100,
    windowDuration: 1 * time.Minute,
}
```

### Production insight: Google's retry strategy

```yaml
google_sre_retry_guidelines:

  do_retry:
    - Timeouts (network or service)
    - 5xx errors (server errors)
    - Connection failures

  dont_retry:
    - 4xx errors (client errors - retrying won't help)
    - 503 Service Unavailable (overload - retrying makes it worse)
    - 429 Too Many Requests (rate limited)

  retry_policy:
    max_attempts: 3
    base_delay: 100ms
    max_delay: 10 seconds
    backoff_multiplier: 2
    jitter: random(0, base_delay)
```

### Key insight box
> Retries amplify failures. Safe retry strategies use exponential backoff, jitter, and don't retry on overload signals.

### Challenge question
Your service receives a retry storm (1000 req/sec retries on top of 1000 req/sec normal traffic). How do you detect and mitigate it in real-time?

---

## Graceful degradation - failing partially instead of completely

### Scenario
Your recommendation service is down. Should your entire product page fail?

### Graceful degradation strategies

**Strategy 1: Feature flags for non-critical features**
```go
package degradation

func RenderProductPage(productID string) (Page, error) {
    page := Page{ProductID: productID}

    // Critical: product details (must succeed)
    details, err := GetProductDetails(productID)
    if err != nil {
        return Page{}, err // Fail entire page
    }
    page.Details = details

    // Non-critical: recommendations (degrade gracefully)
    if features.IsEnabled("recommendations") {
        recs, err := GetRecommendations(productID)
        if err != nil {
            log.Warn("Recommendations failed, rendering without them")
            page.Recommendations = nil // Degrade, don't fail
        } else {
            page.Recommendations = recs
        }
    }

    // Non-critical: user reviews (degrade gracefully)
    reviews, err := GetReviews(productID)
    if err != nil {
        log.Warn("Reviews failed, using cached version")
        page.Reviews = GetCachedReviews(productID)
    } else {
        page.Reviews = reviews
    }

    return page, nil
}
```

**Strategy 2: Fallback values**
```yaml
service_with_fallbacks:

  primary: call_recommendations_service()
  fallback_1: get_from_cache()
  fallback_2: return_popular_items()
  fallback_3: return_empty_list()

  flow:
    try primary
    if fail: try fallback_1
    if fail: try fallback_2
    if fail: try fallback_3 (always succeeds)
```

**Strategy 3: Static fallback content**
```go
func GetRecommendations(userID string) []Product {
    // Try live recommendations
    recs, err := recommendationService.Get(userID)
    if err == nil {
        return recs
    }

    // Fallback: personalized cache
    cached := cache.Get(userID)
    if cached != nil {
        return cached
    }

    // Fallback: generic popular products
    return []Product{
        {ID: "popular-1", Name: "Bestseller #1"},
        {ID: "popular-2", Name: "Bestseller #2"},
    }
}
```

**Strategy 4: Partial responses**
```yaml
api_response:
  request: GET /dashboard

  full_response:
    {
      "user": {...},
      "recent_orders": [...],
      "recommendations": [...],
      "notifications": [...]
    }

  partial_response_on_failure:
    {
      "user": {...},
      "recent_orders": [...],
      "recommendations": null,  # Service down
      "_errors": [
        "recommendations service unavailable"
      ]
    }

  benefit: User gets most of dashboard, just missing recommendations
```

### Criticality classification

```yaml
feature_criticality:

  tier_1_critical:
    - User authentication
    - Payment processing
    - Order creation
    - Inventory checks
    policy: MUST succeed or fail entire request

  tier_2_important:
    - Product search
    - Shopping cart
    - Order history
    policy: Retry with fallback, degrade if necessary

  tier_3_nice_to_have:
    - Recommendations
    - User reviews
    - Analytics tracking
    policy: Fail silently, use fallback or omit
```

### Key insight box
> Not all features are equally critical. Graceful degradation means failing non-critical features without affecting critical paths.

### Challenge question
You've classified features into critical/important/nice-to-have. During an incident, should you proactively disable nice-to-have features to preserve capacity for critical features?

---

## Correlated failures - when everything fails at once

### Scenario
All your circuit breakers, timeouts, and bulkheads are perfect.

Then: AWS us-east-1 has a major outage. All regions depend on it for authentication.

Your entire platform is down. No amount of cascade prevention helped.

### Correlated failure patterns

**Pattern 1: Shared infrastructure**
```yaml
shared_infrastructure_failure:

  example_1_shared_database:
    - All regions use same Postgres cluster
    - Database fails
    - All regions down

  example_2_shared_auth:
    - All regions call auth service in us-east-1
    - us-east-1 fails
    - All regions can't authenticate users

  example_3_shared_dns:
    - All services use same DNS provider
    - DNS provider down
    - No service reachable
```

**Pattern 2: Thundering herd after recovery**
```yaml
thundering_herd:
  scenario:
    - Database down for 5 minutes
    - 10,000 requests queued up
    - Database recovers
    - All 10,000 requests hit at once
    - Database overwhelmed again

  solution:
    - Don't queue during outage (reject instead)
    - Gradually increase traffic after recovery
    - Circuit breaker half-open state (limited requests)
```

**Pattern 3: Deployment-induced cascades**
```yaml
deployment_cascade:
  scenario:
    - Deploy new version to all regions simultaneously
    - New version has a bug (memory leak)
    - All regions fail at the same time

  solution:
    - Canary deployments (deploy to one region first)
    - Blue-green deployments (gradual traffic shift)
    - Automated rollback on health check failures
```

### Defense against correlated failures

**Defense 1: Redundant infrastructure**
```yaml
redundancy:

  authentication:
    primary: Auth service in us-east-1
    secondary: Auth service in eu-west-1
    policy: Failover to secondary if primary down

  database:
    primary: Postgres in us-east-1
    replica: Postgres in eu-west-1
    policy: Read from replica if primary slow/down

  dns:
    primary: Route53
    secondary: Cloudflare
    policy: Use secondary if primary resolution fails
```

**Defense 2: Shuffle sharding**
```yaml
shuffle_sharding:
  concept: Each customer uses a unique subset of infrastructure

  example:
    total_servers: 100
    servers_per_customer: 10

    customer_a: uses servers [1, 5, 12, 18, 25, 33, 47, 56, 72, 89]
    customer_b: uses servers [3, 7, 14, 22, 31, 40, 51, 65, 78, 94]

  benefit:
    - If servers [1, 5, 12] fail → only customer_a affected
    - customer_b uses different servers → unaffected

  result: Reduced blast radius
```

**Defense 3: Chaos engineering**
```yaml
chaos_engineering:
  practice: Regularly inject failures to test resilience

  scenarios:
    - Kill random instances
    - Simulate network partitions
    - Inject latency to dependencies
    - Take down entire regions

  goal: Find correlated failure modes before they happen in production
```

### Key insight box
> Cascades can be prevented, but correlated failures require architectural redundancy. Build systems where failures are independent.

### Challenge question
Your system survives individual component failures perfectly. How do you test that it survives multiple simultaneous failures?

---

## Final synthesis - Design a cascade-resistant architecture

### Synthesis challenge
You're the architect for a global payment processing platform.

Requirements:
- 100K transactions/second peak
- 99.99% uptime SLA (52 minutes downtime/year)
- Must survive: database failures, region outages, dependency timeouts
- Payment must succeed or fail deterministically (no "maybe" states)

Constraints:
- Complex dependency graph: 15 microservices
- Shared database for transactions (consistency required)
- Third-party dependencies: fraud detection, KYC, card networks
- Cannot tolerate retry storms (could double-charge customers)

### Your tasks (pause and think)
1. Draw dependency graph and identify cascade risks
2. Apply timeout strategy per service
3. Design circuit breaker policy
4. Implement bulkheads for critical paths
5. Define retry policy that won't cause double-charges
6. Plan graceful degradation for non-critical features
7. Design defense against correlated failures

Write down your architecture.

### Progressive reveal (one possible solution)

**1. Dependency graph and cascade risks:**
```yaml
dependency_graph:

  customer_request:
    → api_gateway
      → authentication_service
        → user_database (shared risk)
      → payment_service (critical path)
        → fraud_detection (external, may be slow)
        → kyc_verification (external, may be slow)
        → card_network (external, may fail)
        → transaction_database (shared, must be consistent)
      → notification_service (non-critical)
        → email_service (external)

cascade_risks:
  high_risk:
    - transaction_database: if slow, blocks all payments
    - card_network: if down, no payments possible
    - api_gateway: if overwhelmed, everything fails

  medium_risk:
    - fraud_detection: if slow, could block payments
    - authentication: if down, no one can pay

  low_risk:
    - notification_service: if down, payments still work
    - email_service: if down, just skip notifications
```

**2. Timeout strategy:**
```yaml
timeouts:

  api_gateway:
    total_timeout: 10 seconds
    includes: all downstream calls

  payment_service:
    fraud_detection: 2 seconds (must be fast)
    kyc_verification: 3 seconds (can be slower)
    card_network: 5 seconds (external, variable)
    transaction_database: 1 second (should be fast)

  notification_service:
    email_service: 500ms (non-critical, fail fast)

  hierarchical_constraint:
    - api_gateway (10s) > payment_service (5s) > card_network (5s)
    - Each layer times out before parent layer
```

**3. Circuit breaker policy:**
```yaml
circuit_breakers:

  fraud_detection:
    failure_threshold: 50% error rate over 10 seconds
    timeout: 30 seconds
    half_open_requests: 5
    fallback: allow transaction (fraud check async post-payment)

  kyc_verification:
    failure_threshold: 50% error rate over 10 seconds
    timeout: 30 seconds
    half_open_requests: 5
    fallback: use cached KYC status (if available)

  card_network:
    failure_threshold: 30% error rate over 5 seconds
    timeout: 60 seconds
    half_open_requests: 10
    fallback: none (can't process without card network)

  email_service:
    failure_threshold: 80% error rate over 30 seconds
    timeout: 5 minutes
    fallback: queue for later delivery (non-critical)
```

**4. Bulkheads:**
```yaml
bulkheads:

  api_gateway_thread_pools:
    payment_endpoints: 500 threads (critical)
    dashboard_endpoints: 200 threads (important)
    analytics_endpoints: 100 threads (nice-to-have)

  payment_service_connection_pools:
    transaction_database: 200 connections (critical)
    fraud_detection: 100 connections
    kyc_verification: 100 connections
    card_network: 200 connections

  isolation_benefit:
    - If fraud_detection slow → only 100 connections blocked
    - transaction_database still has 200 connections available
    - Payments can continue (with fraud check fallback)
```

**5. Retry policy:**
```yaml
retry_policy:

  payment_transaction:
    idempotency_key: required (UUID per transaction)
    retries: 0 (NEVER retry payments automatically)
    reason: Risk of double-charge
    client_responsibility: Client must retry with same idempotency key

  fraud_detection:
    retries: 2
    backoff: exponential with jitter
    idempotent: yes (same request → same response)

  kyc_verification:
    retries: 2
    backoff: exponential with jitter
    idempotent: yes

  notification:
    retries: 5
    backoff: exponential (100ms, 200ms, 400ms, 800ms, 1600ms)
    async_queue: true (retry later if all fail)
```

**6. Graceful degradation:**
```yaml
degradation_tiers:

  critical_no_degradation:
    - Payment processing
    - Transaction recording
    - User authentication
    policy: Fail entire request if these fail

  important_with_fallback:
    - Fraud detection
      fallback: Allow transaction, flag for review
    - KYC verification
      fallback: Use cached status (if recent)

  non_critical_omit:
    - Email notifications
      fallback: Queue for later
    - Real-time analytics
      fallback: Skip, collect asynchronously
    - Recommendation engine
      fallback: Return empty list
```

**7. Correlated failure defenses:**
```yaml
correlated_failure_defenses:

  database_redundancy:
    primary: transaction_db in us-east-1
    replica: read replica in eu-west-1
    policy:
      - Writes: always primary (consistency required)
      - Reads: replica if primary slow/down
      - Failover: manual (can't auto-failover with consistency requirement)

  multi_region:
    regions: [us-east-1, eu-west-1, ap-south-1]
    deployment: canary (deploy to us-east-1, then others)
    data: sharded by customer geography
    benefit: Regional failure only affects 33% of customers

  shuffle_sharding:
    card_network_connections:
      - 10 connection pools to card network
      - Each customer assigned to 2 pools (primary + backup)
      - If 1 pool fails → 10% customer impact, not 100%

  chaos_testing:
    monthly_game_days:
      - Simulate fraud_detection timeout
      - Simulate transaction_database slow queries
      - Simulate card_network 50% error rate
      - Verify graceful degradation works
```

### Key insight box
> Cascade prevention in payment systems requires defense-in-depth: timeouts, circuit breakers, bulkheads, careful retry policies, and graceful degradation.

### Final challenge question
Your payment system is cascade-resistant. But what if the cascade starts in YOUR system and propagates to your CUSTOMERS' systems? How do you prevent your failures from cascading downstream?

---

## Appendix: Quick checklist (printable)

**Cascade prevention design:**
- [ ] Map dependency graph (understand what depends on what)
- [ ] Identify shared resources (database, cache, auth)
- [ ] Classify features by criticality (critical, important, nice-to-have)
- [ ] Design failure modes (what should fail together, what should isolate)
- [ ] Plan fallback strategies (cache, default values, omit)

**Timeout configuration:**
- [ ] Set timeouts on all network calls (no infinite waits)
- [ ] Use hierarchical timeouts (child < parent)
- [ ] Configure per-dependency timeouts (critical vs non-critical)
- [ ] Test timeout behavior (simulate slow dependencies)
- [ ] Monitor timeout frequency (alert on unusual spikes)

**Circuit breaker setup:**
- [ ] Implement circuit breakers on external dependencies
- [ ] Set failure thresholds (50% error rate typical)
- [ ] Configure timeout before half-open (30-60 seconds)
- [ ] Define fallback behavior (cache, default, error)
- [ ] Monitor circuit state changes (closed, open, half-open)

**Bulkhead implementation:**
- [ ] Separate thread pools per dependency
- [ ] Separate connection pools per service
- [ ] Size bulkheads based on capacity (don't over-allocate)
- [ ] Monitor bulkhead utilization (alert when full)
- [ ] Test bulkhead isolation (one full bulkhead shouldn't affect others)

**Retry strategy:**
- [ ] Use exponential backoff (not immediate retry)
- [ ] Add jitter (prevent thundering herd)
- [ ] Don't retry on overload (503, 429)
- [ ] Implement retry budgets (limit retries per time window)
- [ ] Use idempotency keys (prevent duplicate side effects)

**Graceful degradation:**
- [ ] Define fallback for each non-critical feature
- [ ] Use cached data when fresh data unavailable
- [ ] Return partial responses (not all-or-nothing)
- [ ] Skip optional features under load
- [ ] Communicate degradation to users (if visible)

**Monitoring:**
- [ ] Track timeout frequency per service
- [ ] Monitor circuit breaker state transitions
- [ ] Alert on bulkhead saturation
- [ ] Measure retry rates (detect retry storms)
- [ ] Track cascading failure metrics (time-to-cascade, blast radius)

**Red flags (redesign needed):**
- [ ] Cascading failures happen frequently (poor isolation)
- [ ] Same failure affects unrelated features (shared fate)
- [ ] Retry storms observed regularly (bad retry strategy)
- [ ] Circuit breakers always open (dependency not reliable enough)
- [ ] Timeouts too aggressive (false positives) or too lenient (thread exhaustion)
