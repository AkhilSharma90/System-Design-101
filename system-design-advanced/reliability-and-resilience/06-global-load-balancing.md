---
slug: global-load-balancing
title: Global Load Balancing
readTime: 20 min
orderIndex: 5
premium: false
---

# Global Load Balancing (GSLB) - Routing Traffic Across the Planet

> Audience: infrastructure engineers and network architects managing global traffic distribution.

This article assumes:
- Your users are distributed globally across continents and time zones.
- Network conditions vary: latency, packet loss, routing policies differ by geography.
- Load balancers within a region aren't enough - you need cross-region intelligence.
- DNS alone is too slow and too dumb for modern traffic management.

---

## Challenge: Your "global" load balancer isn't actually global

### Scenario
You deploy to three AWS regions: US, EU, and Asia.

You set up Route53 with latency-based routing. "We're globally load balanced!" you announce.

Then reality hits:

- Asian users get routed to US during an AWS routing table glitch
- EU region is overloaded but DNS keeps sending traffic there (60-second TTL delay)
- A DDoS attack in US region affects all users (DNS doesn't detect application-layer attacks)
- Your "global" setup failed the first real test

### Interactive question (pause and think)
What's the difference between:
1. DNS-based routing (Route53, Cloudflare DNS)
2. Anycast-based routing (BGP announces)
3. True Global Server Load Balancing (GSLB)

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: Increasing levels of intelligence and control.

1. **DNS routing**: Dumb, slow, client-controlled
2. **Anycast**: Fast, automatic, but no application awareness
3. **GSLB**: Intelligent, application-aware, dynamic routing decisions

### Real-world analogy (airport routing)
Imagine routing passengers to airports:

**DNS routing**: At ticket purchase time, you assign them to "nearest airport" based on their home address. If that airport closes, they're stuck with outdated ticket.

**Anycast**: Passengers head to any flight with the same flight number. Airline network automatically routes them to an airport that can serve them.

**GSLB**: Intelligent routing system considers: airport capacity, weather, security delays, even passenger preferences. Dynamically reroutes in real-time.

### Key insight box
> GSLB is the control plane for global traffic distribution, making routing decisions based on real-time health, capacity, and performance.

### Challenge question
If GSLB is so much better, why does anyone still use basic DNS routing?

---

##  Mental model - GSLB as traffic orchestrator

### Scenario
You have three data centers: US-West, US-East, and EU.

Traditional load balancing (within region):
- Client → Regional LB → Pick backend server

Global load balancing (across regions):
- Client → GSLB → Pick region → Regional LB → Pick server

### Interactive question (pause and think)
What information does GSLB need to make good routing decisions?

A. Server CPU/memory utilization
B. Network latency to each region
C. Regional health and capacity
D. All of the above, plus business logic

### Progressive reveal
Answer: D.

GSLB is a decision engine that considers:
- **Client location** (geography, network)
- **Endpoint health** (is region up? degraded?)
- **Capacity** (can region handle more load?)
- **Performance** (latency, throughput)
- **Business rules** (compliance, cost, SLAs)

### Mental model
Think of GSLB as:

- **Air traffic control** for network packets
- **Smart router** with real-time world view
- **Policy engine** enforcing business rules

The goal: Route each request to the "best" endpoint at that moment.

### Real-world parallel (emergency dispatch)
When you call 911, the dispatcher doesn't just send the "nearest" ambulance. They consider:
- Which ambulances are available (capacity)
- Which are already en route to other emergencies (load)
- Which hospital has the right facilities (capability)
- Traffic conditions (latency)

GSLB does the same for network traffic.

### Key insight box
> GSLB moves routing intelligence from static configuration (DNS) to dynamic decision-making based on real-time telemetry.

### Challenge question
Should GSLB routing decisions prioritize latency, cost, or reliability? Or does it depend on the request type?

---

##  Understanding GSLB architectures - DNS vs proxy vs hybrid

### Scenario
Your team debates: "Should our GSLB be DNS-based, proxy-based, or hybrid?"

Different architectures, different trade-offs.

### GSLB architecture patterns

**Pattern 1: DNS-based GSLB**
```text
How it works:
  Client queries: api.example.com
  ↓
  GSLB DNS server (intelligent DNS)
  ├─ Checks: client location, regional health, capacity
  ├─ Decision: route to US-East
  └─ Returns: IP address of US-East LB
  ↓
  Client connects directly to US-East

Pros:
  ✓ Client connects directly to backend (no proxy overhead)
  ✓ Scales horizontally (DNS is lightweight)
  ✓ Works with any client (no special software needed)

Cons:
  ✗ DNS caching (TTL delays, client might ignore)
  ✗ No per-request intelligence (same IP for all requests in TTL window)
  ✗ Limited observability (DNS can't see HTTP headers, response codes)
  ✗ Slow failover (30-60 seconds typical)

Best for:
  - Web applications
  - Mobile apps
  - Long-lived connections (WebSocket)

Example tech:
  - AWS Route53 with health checks
  - Cloudflare Load Balancing
  - F5 GTM (Global Traffic Manager)
```

**Pattern 2: Proxy-based GSLB (Reverse proxy)**
```text
How it works:
  Client connects to: gslb-proxy.example.com
  ↓
  GSLB Proxy (Envoy, HAProxy, nginx)
  ├─ Terminates TLS connection
  ├─ Inspects HTTP request (headers, path, method)
  ├─ Makes routing decision per-request
  └─ Proxies to chosen backend region
  ↓
  Backend responds through proxy

Pros:
  ✓ Per-request routing (can route /api/v1 to US, /api/v2 to EU)
  ✓ Full visibility (sees all HTTP traffic)
  ✓ Instant failover (no DNS caching)
  ✓ Advanced features (rate limiting, A/B testing, canary)

Cons:
  ✗ Proxy overhead (latency, compute cost)
  ✗ Single point of failure (must make proxy highly available)
  ✗ Bandwidth costs (all traffic proxied)
  ✗ Complexity (additional infrastructure layer)

Best for:
  - API gateways
  - Applications needing per-request routing
  - Advanced traffic management (A/B, canary, blue-green)

Example tech:
  - Envoy with global control plane
  - Cloudflare Workers (edge proxy)
  - AWS Global Accelerator (anycast + proxy)
```

**Pattern 3: Hybrid (DNS + Proxy)**
```text
How it works:
  Client queries: api.example.com
  ↓
  DNS GSLB: returns IP of nearest edge proxy
  ↓
  Edge Proxy (regional)
  ├─ Terminates connection locally (low latency TLS)
  ├─ Routes to optimal backend (within region or cross-region)
  └─ Proxies request

Pros:
  ✓ Low TLS latency (edge termination)
  ✓ Per-request intelligence (proxy features)
  ✓ Regional resilience (multiple edge proxies)
  ✓ Best of both worlds

Cons:
  ✗ Most complex architecture
  ✗ Highest operational overhead
  ✗ Requires edge proxy infrastructure

Best for:
  - CDN-like workloads
  - Global SaaS platforms
  - High-scale applications (Netflix, Stripe)

Example tech:
  - Cloudflare (DNS + edge workers)
  - Fastly (DNS + edge compute)
  - AWS CloudFront + Lambda@Edge
```

### Visual comparison

```text
┌─────────────────────────────────────────────────────────┐
│          GSLB ARCHITECTURE COMPARISON                   │
└─────────────────────────────────────────────────────────┘

DNS-BASED:
  Client → [DNS GSLB] → returns IP → Client → Backend

  Latency: LOW (direct connection)
  Flexibility: LOW (DNS only)
  Observability: LOW (no request visibility)
  Cost: LOW ($)

PROXY-BASED:
  Client → [Proxy GSLB] ↔ Backend

  Latency: MEDIUM (proxy hop)
  Flexibility: HIGH (per-request routing)
  Observability: HIGH (full traffic visibility)
  Cost: HIGH ($$$)

HYBRID:
  Client → [DNS] → Edge Proxy → Backend

  Latency: LOW-MEDIUM (edge TLS termination)
  Flexibility: HIGH (proxy features)
  Observability: HIGH (full traffic visibility)
  Cost: MEDIUM-HIGH ($$)
```

### Interactive question
You're building a global API serving mobile apps. Which GSLB pattern?

A. DNS-based (simple, cheap)
B. Proxy-based (full control)
C. Hybrid (best of both)

### Progressive reveal
Answer: Start with A (DNS-based), migrate to C (hybrid) as you scale.

Why? Mobile apps can handle DNS TTL caching better than browsers. Start simple, add complexity only when needed.

### Key insight box
> GSLB architecture is not one-size-fits-all. Start with DNS, add proxy layers as requirements demand per-request intelligence.

### Challenge question
Can you implement GSLB without any special infrastructure, using only standard DNS and load balancers?

---

##  GSLB health checks and failover - the detection problem

### Scenario
Your EU region is degraded (high latency, elevated errors). When should GSLB stop routing traffic there?

### Health check strategies

**Strategy 1: Binary health checks (up/down)**
```yaml
simple_health_check:
  endpoint: https://eu-region.example.com/health
  interval: 10 seconds
  timeout: 2 seconds
  healthy_threshold: 2 consecutive successes
  unhealthy_threshold: 3 consecutive failures

  decision:
    if response.status == 200:
      mark_healthy()
    else:
      mark_unhealthy()

  pros:
    - Simple logic
    - Easy to implement

  cons:
    - Binary view (doesn't capture degradation)
    - Synthetic check might succeed while real traffic fails
    - Doesn't consider capacity or latency
```

**Strategy 2: Weighted health (capacity-aware)**
```yaml
weighted_health_check:
  endpoint: https://eu-region.example.com/health
  response_expected:
    {
      "healthy": true,
      "capacity_percent": 75,  # 75% of capacity available
      "latency_p99": 450       # p99 latency in ms
    }

  decision:
    weight = capacity_percent / 100.0
    if latency_p99 > 1000:
      weight *= 0.5  # Penalize high latency

    send_proportional_traffic(region, weight)

  example:
    US: 100% capacity, p99=200ms → weight=1.0 → 50% traffic
    EU: 75% capacity, p99=450ms → weight=0.75 → 37.5% traffic
    Asia: 50% capacity, p99=300ms → weight=0.5 → 12.5% traffic
```

**Strategy 3: Active vs passive health checks**
```yaml
active_health_checks:
  type: synthetic_probes
  how: GSLB sends periodic health checks
  frequency: every 10 seconds

  pros:
    - Detects failures even with zero traffic
    - Predictable, consistent

  cons:
    - Might not match real user experience
    - Additional load on backends

passive_health_checks:
  type: traffic_analysis
  how: GSLB monitors actual user requests
  measurement:
    - Success rate (% 2xx responses)
    - Latency (p50, p99)
    - Error rate (% 5xx responses)

  pros:
    - Matches real user experience
    - No additional load

  cons:
    - Requires traffic to detect issues
    - Slower to detect (needs sample size)

best_practice: Use both
  - Active: catch issues before user impact
  - Passive: validate real user experience
```

### Failover decision logic

```go
package gslb

import (
    "time"
)

type RegionHealth struct {
    Region           string
    IsHealthy        bool
    CapacityPercent  float64
    LatencyP99       time.Duration
    ErrorRate        float64
    LastChecked      time.Time
}

type RoutingPolicy struct {
    LatencyThreshold   time.Duration
    ErrorRateThreshold float64
    MinCapacity        float64
}

func (p *RoutingPolicy) CalculateWeight(h RegionHealth) float64 {
    // Start with full weight
    weight := 1.0

    // Binary health check
    if !h.IsHealthy {
        return 0.0  // Completely remove from rotation
    }

    // Capacity-based weight
    weight *= h.CapacityPercent / 100.0

    // Latency penalty
    if h.LatencyP99 > p.LatencyThreshold {
        latencyPenalty := float64(h.LatencyP99) / float64(p.LatencyThreshold)
        weight /= latencyPenalty
    }

    // Error rate penalty
    if h.ErrorRate > p.ErrorRateThreshold {
        weight *= 0.5  // Heavy penalty for errors
    }

    // Floor at zero
    if weight < 0 {
        weight = 0
    }

    return weight
}

// Example usage
func main() {
    policy := RoutingPolicy{
        LatencyThreshold:   500 * time.Millisecond,
        ErrorRateThreshold: 0.01,  // 1% error rate
        MinCapacity:        0.20,  // 20% capacity minimum
    }

    regions := []RegionHealth{
        {Region: "us-east", IsHealthy: true, CapacityPercent: 80, LatencyP99: 200 * time.Millisecond, ErrorRate: 0.001},
        {Region: "eu-west", IsHealthy: true, CapacityPercent: 60, LatencyP99: 800 * time.Millisecond, ErrorRate: 0.005},
        {Region: "ap-south", IsHealthy: false, CapacityPercent: 100, LatencyP99: 300 * time.Millisecond, ErrorRate: 0.5},
    }

    for _, region := range regions {
        weight := policy.CalculateWeight(region)
        // us-east: weight ≈ 0.80 (high)
        // eu-west: weight ≈ 0.375 (penalized for latency)
        // ap-south: weight = 0.0 (unhealthy)
    }
}
```

### Avoiding thundering herd during failover

```yaml
problem: thundering_herd
scenario:
  1. EU region fails health check
  2. GSLB removes EU from rotation
  3. All EU traffic (35% of total) shifts to US
  4. US overloads instantly (not enough capacity)
  5. US starts failing health checks
  6. Now both regions down

solution: gradual_failover
  step_1:
    - Detect EU degradation (latency high)
    - Reduce EU weight from 100% to 75%

  step_2:
    - EU continues degrading
    - Reduce weight from 75% to 50%

  step_3:
    - EU fully unhealthy
    - Remove from rotation (0%)
    - But US has had time to auto-scale

  implementation:
    - Failover over 5-10 minutes (not instant)
    - Allow auto-scaling to catch up
    - Monitor receiving region capacity
```

### Key insight box
> GSLB health checks are not just "is it up?" but "can it handle more load, and how well is it performing?"

### Challenge question
Your health check shows region is healthy, but real user requests are failing. What went wrong?

---

##  GSLB routing policies - latency vs cost vs compliance

### Scenario
You have three regions: US ($), EU ($$), Asia ($$$).

User in Japan makes a request. Where should GSLB route it?

### Think about it
Factors to consider:
- **Latency**: Asia is nearest (50ms), US is 150ms, EU is 200ms
- **Cost**: Asia is 3x more expensive than US
- **Compliance**: User data might need to stay in specific region
- **Capacity**: Asia might be at 90% capacity, US at 50%

### Interactive question (pause and think)
Which routing policy?

A. Always route to nearest region (minimize latency)
B. Always route to cheapest region (minimize cost)
C. Balance latency and cost (weighted decision)
D. Different policy per request type (API vs static assets)

### Progressive reveal
Answer: Usually C or D.

One-size-fits-all routing is almost always wrong.

### GSLB routing policies catalog

**Policy 1: Latency-based (performance-first)**
```yaml
latency_based_routing:
  goal: minimize user-perceived latency

  decision:
    measure_latency_to_each_region()
    route_to(region_with_lowest_latency)

  example:
    user_in: Tokyo
    latency:
      us-west: 120ms
      eu-west: 250ms
      ap-south: 40ms
    decision: route to ap-south

  use_cases:
    - Real-time APIs (trading, gaming)
    - User-facing applications
    - Mobile apps

  tradeoff: Higher cost (might route to expensive region)
```

**Policy 2: Cost-based (spend-first)**
```yaml
cost_based_routing:
  goal: minimize infrastructure spend

  decision:
    costs = {
      'us-east': 1.0,
      'eu-west': 1.5,
      'ap-south': 2.0
    }
    route_to(cheapest_region)

  example:
    user_in: Tokyo
    decision: route to us-east (cheapest)
    tradeoff: 120ms latency vs 40ms

  use_cases:
    - Batch processing
    - Analytics pipelines
    - Non-latency-sensitive workloads
```

**Policy 3: Geographic (compliance-first)**
```yaml
geographic_routing:
  goal: data residency compliance

  rules:
    eu_users: must route to eu-west (GDPR)
    us_users: must route to us-east (data sovereignty)
    asia_users: can route to any region

  example:
    user_in: Germany
    decision: route to eu-west (required by law)
    tradeoff: Cannot optimize for cost or latency
```

**Policy 4: Weighted (multi-objective)**
```yaml
weighted_routing:
  goal: balance latency, cost, capacity

  scoring_function:
    score(region) =
      (latency_weight * latency_score) +
      (cost_weight * cost_score) +
      (capacity_weight * capacity_score)

  example:
    weights:
      latency: 0.6  # Latency matters most
      cost: 0.3     # Cost matters somewhat
      capacity: 0.1 # Capacity is tie-breaker

    user_in: Tokyo
    scores:
      us-east: (0.6 * 0.5) + (0.3 * 1.0) + (0.1 * 0.8) = 0.68
      eu-west: (0.6 * 0.3) + (0.3 * 0.7) + (0.1 * 0.6) = 0.45
      ap-south: (0.6 * 1.0) + (0.3 * 0.3) + (0.1 * 0.2) = 0.71

    decision: route to ap-south (highest score)
```

**Policy 5: Request-aware (path-based)**
```yaml
request_aware_routing:
  goal: different routing per request type

  rules:
    '/api/payment': route to nearest region (latency-critical)
    '/api/analytics': route to cheapest region (cost-sensitive)
    '/static/*': route to CDN (cached, geography-agnostic)
    '/api/eu-data': route to EU only (compliance)

  implementation:
    - Requires proxy-based GSLB (inspect HTTP path)
    - Not possible with DNS-only GSLB
```

### Real-world example: Stripe's routing

```yaml
stripe_routing_strategy:

  payment_apis:
    policy: latency_based + geographic compliance
    reason: Sub-100ms latency critical, some EU data must stay in EU

  dashboard_ui:
    policy: latency_based
    reason: User-facing, latency matters

  webhooks_delivery:
    policy: cost_based
    reason: Batch delivery, latency less critical

  billing_reports:
    policy: cost_based + batching
    reason: Generated offline, users don't wait
```

### Key insight box
> The best GSLB routing policy depends on request type. API calls, static assets, and batch jobs should use different policies.

### Challenge question
Can you dynamically adjust routing weights based on time of day (e.g., route to cheaper regions during low-traffic hours)?

---

##  Anycast and BGP - GSLB's secret weapon

### Scenario
You want instant failover (no DNS caching) and you want users to reach "nearest" region automatically.

Enter: Anycast with BGP.

### What is Anycast?

```text
Traditional Unicast:
  Each IP address → One server
  1.2.3.4 → Server in US-East only

Anycast:
  Same IP address → Multiple servers (different locations)
  1.2.3.4 → Server in US-East AND EU-West AND Asia

  Internet routes to "nearest" based on BGP hop count
```

### How Anycast works with GSLB

```text
┌─────────────────────────────────────────────────────────┐
│              ANYCAST + GSLB ARCHITECTURE                │
└─────────────────────────────────────────────────────────┘

Setup:
  1. Allocate anycast IP: 1.2.3.4
  2. Announce 1.2.3.4 from US, EU, Asia regions via BGP
  3. Internet routers choose "nearest" announcement

User in New York:
  ├─ Queries: api.example.com
  ├─ DNS returns: 1.2.3.4 (anycast IP)
  ├─ User connects to 1.2.3.4
  └─ BGP routes to US region (nearest)

User in London:
  ├─ Same DNS query: 1.2.3.4
  ├─ User connects to 1.2.3.4
  └─ BGP routes to EU region (nearest)

Failover:
  If US region fails:
    1. Stop announcing 1.2.3.4 from US
    2. BGP reconverges (10-30 seconds)
    3. Traffic automatically routes to EU or Asia
    4. No DNS change needed
```

### Anycast implementation

```yaml
anycast_setup:

  step_1_ip_allocation:
    provider: AWS, GCP, or ISP
    anycast_block: 1.2.3.0/24
    per_region_ip: 1.2.3.4 (same IP in all regions)

  step_2_bgp_announcement:
    us_region:
      router: announce 1.2.3.0/24 via BGP
      as_path: [64512, 174, 3356]  # Example AS path

    eu_region:
      router: announce 1.2.3.0/24 via BGP
      as_path: [64513, 174, 3356]

    ap_region:
      router: announce 1.2.3.0/24 via BGP
      as_path: [64514, 174, 3356]

  step_3_health_checks:
    if region_unhealthy:
      action: withdraw_bgp_announcement
      result: internet_routes_to_next_nearest_region

  step_4_traffic_routing:
    client_in_new_york:
      bgp_routes_to: us_region (shortest AS path)

    client_in_london:
      bgp_routes_to: eu_region (shortest AS path)
```

### Anycast + GSLB hybrid

```yaml
# Best of both worlds
hybrid_architecture:

  layer_1_dns:
    query: api.example.com
    response: 1.2.3.4 (anycast IP, shared across regions)
    ttl: 300 seconds (can be higher, anycast handles failover)

  layer_2_anycast:
    ip: 1.2.3.4
    announced_from: [us-east, eu-west, ap-south]
    routes_to: nearest region (automatically)

  layer_3_regional_lb:
    each_region:
      - Anycast routes to regional load balancer
      - Regional LB distributes to backend servers

  failover:
    detection: Health check fails in us-east
    action: Withdraw BGP announcement from us-east
    result: Traffic re-routes to eu-west/ap-south
    time: 10-30 seconds (BGP convergence)
```

### Anycast limitations

```yaml
limitations:

  session_affinity:
    problem: BGP routes can change mid-session
    example:
      - User connects to US region
      - BGP routing changes (peering issue)
      - Next packet routes to EU region
      - Connection breaks (different server, no state)

    solution: Use anycast for stateless protocols (DNS, HTTP)
              Or use connection-level state (TCP anycast OK if stateless app)

  asymmetric_routing:
    problem: Request to US, response from EU (asymmetric)
    result: Firewalls might block (unexpected source)

  limited_control:
    problem: BGP routing is coarse (can't do per-request routing)
    solution: Use anycast to reach edge proxy, proxy does intelligent routing

  cost:
    problem: Requires BGP peering, ASN allocation, ISP coordination
    result: Complex, expensive (enterprise/cloud provider feature)
```

### Key insight box
> Anycast solves the "instant failover" problem that DNS can't. But it requires BGP expertise and works best for stateless protocols.

### Challenge question
Can you use anycast for a database connection (stateful TCP)? What breaks?

---

##  GSLB observability - can you see what it's deciding?

### Scenario
Your GSLB routes traffic globally. Users in EU complain about slow response times.

Questions:
- Are they being routed to EU region or somewhere else?
- Is EU region healthy or degraded?
- Is GSLB making the right decision?

Without observability, you're blind.

### GSLB metrics to track

**Routing decision metrics:**
```yaml
metrics_per_region:
  traffic_distribution:
    - Requests routed to us-east: 10K/sec
    - Requests routed to eu-west: 7K/sec
    - Requests routed to ap-south: 3K/sec

  decision_reasons:
    - Routed by latency: 60%
    - Routed by capacity: 25%
    - Routed by compliance: 15%

  health_scores:
    - us-east: weight=1.0 (healthy, full capacity)
    - eu-west: weight=0.6 (healthy, but degraded latency)
    - ap-south: weight=0.3 (low capacity)
```

**Client perspective metrics:**
```yaml
per_client_metrics:
  - Client IP: 203.0.113.5
  - Geolocation: Tokyo, Japan
  - Routed to: ap-south
  - DNS response: 1.2.3.4
  - Actual connection: ap-south (verified)
  - Latency: 45ms
  - Outcome: Success (200 OK)
```

**Failover metrics:**
```yaml
failover_events:
  - Timestamp: 2024-01-04 14:32:15 UTC
  - Region: eu-west
  - Event: Marked unhealthy (3 consecutive health check failures)
  - Action: Removed from rotation
  - Traffic redistributed:
      - us-east: +4K requests/sec
      - ap-south: +3K requests/sec
  - Recovery: 2024-01-04 14:47:30 UTC (15 minutes downtime)
```

### GSLB debugging dashboard

```text
┌─────────────────────────────────────────────────────────┐
│              GSLB OBSERVABILITY DASHBOARD               │
└─────────────────────────────────────────────────────────┘

TRAFFIC DISTRIBUTION (real-time):
  ████████████████████ US-East: 50% (10K req/sec)
  ██████████████ EU-West: 30% (6K req/sec)
  ████████ AP-South: 20% (4K req/sec)

REGIONAL HEALTH:
  US-East:  [✓] Healthy | Capacity: 60% | Latency: 180ms
  EU-West:  [⚠] Degraded | Capacity: 85% | Latency: 650ms
  AP-South: [✓] Healthy | Capacity: 40% | Latency: 220ms

ROUTING DECISIONS (last 1000 requests):
  Latency-based: 650 (65%)
  Geographic:    200 (20%)
  Cost-based:    100 (10%)
  Capacity:       50 (5%)

RECENT FAILOVER EVENTS:
  14:32 - EU-West marked unhealthy (high error rate)
  14:35 - Traffic redistributed to US-East, AP-South
  14:47 - EU-West recovered, gradual traffic restoration

CLIENT GEO DISTRIBUTION:
  North America: 40%
  Europe: 35%
  Asia: 20%
  Other: 5%

ALERTS:
  [CRITICAL] EU-West p99 latency > 500ms threshold
  [WARNING] AP-South capacity at 90%
```

### Distributed tracing for GSLB

```yaml
trace_example:
  trace_id: abc-123-def-456

  span_1_dns_query:
    service: route53
    operation: query api.example.com
    result: returns [1.2.3.4, 5.6.7.8]
    duration: 15ms

  span_2_gslb_decision:
    service: gslb
    operation: select_region
    input:
      client_ip: 203.0.113.5
      client_geo: Tokyo
    decision: ap-south (latency: 45ms, capacity: 40%)
    duration: 2ms

  span_3_regional_lb:
    service: ap-south-lb
    operation: route_to_backend
    selected_backend: server-42
    duration: 1ms

  span_4_backend_processing:
    service: api-server
    operation: handle_request
    duration: 120ms
    result: 200 OK

  total_latency: 15 + 2 + 1 + 120 = 138ms
```

### Key insight box
> GSLB without observability is a black box. Instrument every routing decision so you can debug user complaints and optimize over time.

### Challenge question
User reports: "I'm in London but your API is slow." How do you use GSLB metrics to debug where they're actually being routed?

---

##  Final synthesis - Design your GSLB strategy

### Synthesis challenge
You're the infrastructure lead for a global video streaming platform.

Requirements:
- 100 million users across North America (45%), Europe (30%), Asia (20%), South America (5%)
- Video content delivery (high bandwidth)
- Real-time playback start (< 2 second buffering)
- Cost-sensitive (bandwidth is expensive)
- Compliance: EU content must stay in EU for licensing

Constraints:
- CDN costs: $0.05/GB in US, $0.08/GB in EU, $0.12/GB in Asia
- Origin servers: US-East, EU-West, Asia-Pacific
- Budget: Must reduce bandwidth costs by 20%
- Current: No GSLB, users manually select region

### Your tasks (pause and think)
1. Choose GSLB architecture (DNS, proxy, hybrid?)
2. Define routing policies (latency vs cost?)
3. Design health check strategy
4. Plan failover approach
5. Implement cost optimization
6. Define observability requirements

Write down your design.

### Progressive reveal (one possible solution)

**1. GSLB architecture:**
```yaml
chosen_architecture: hybrid (DNS + CDN edge)

reasoning:
  - Video streaming = high bandwidth, low latency critical
  - CDN edge for content delivery (cache at edge)
  - DNS for coarse-grained routing to nearest CDN PoP
  - Edge compute for fine-grained routing decisions

implementation:
  layer_1_dns:
    provider: Cloudflare DNS
    policy: latency-based routing
    returns: IP of nearest CDN PoP

  layer_2_cdn:
    provider: Cloudflare CDN (or Fastly, Akamai)
    purpose: Cache video chunks at edge
    routing: intelligent origin selection per request

  layer_3_origin:
    regions: [us-east, eu-west, ap-south]
    purpose: video encoding, storage
```

**2. Routing policies:**
```yaml
routing_policies:

  video_playback_initial:
    policy: latency-based
    goal: Start playback within 2 seconds
    decision: Route to nearest CDN PoP (100-200ms latency)

  video_chunks_ongoing:
    policy: cost + latency hybrid
    goal: Minimize bandwidth costs while maintaining quality
    decision:
      - If latency < 500ms to cheaper region: route there
      - Else: route to nearest region

    example:
      user_in: São Paulo
      nearest: SA region (200ms, $0.10/GB)
      alternative: US-East (400ms, $0.05/GB)
      decision: US-East (latency acceptable, 50% cost savings)

  eu_licensed_content:
    policy: geographic (compliance)
    goal: EU content stays in EU
    decision: Force EU-West origin, no exceptions
```

**3. Health check strategy:**
```yaml
health_checks:

  active_checks:
    type: synthetic_video_request
    frequency: every 30 seconds
    regions: [us-east, eu-west, ap-south]
    test:
      - Request /api/health
      - Stream 10-second test video
      - Measure: latency, throughput, buffering

    thresholds:
      healthy: latency < 500ms, throughput > 5 Mbps
      degraded: latency < 1000ms, throughput > 2 Mbps
      unhealthy: else

  passive_checks:
    type: real_user_monitoring
    metrics:
      - Video start time (target: < 2 sec)
      - Buffering events (target: < 1%)
      - Playback failures (target: < 0.1%)

    action:
      if buffering_rate > 5%:
        reduce_region_weight by 50%
```

**4. Failover approach:**
```yaml
failover_strategy:

  detection:
    - Active health check fails (3 consecutive)
    - OR passive metrics degrade (buffering > 10%)

  action_gradual:
    step_1:
      - Reduce region weight from 100% to 50%
      - Monitor for 2 minutes

    step_2:
      - If still unhealthy, reduce to 25%
      - Monitor for 2 minutes

    step_3:
      - If still unhealthy, remove from rotation (0%)

  recovery:
    - Health check succeeds (5 consecutive)
    - Gradual restoration: 25% → 50% → 100%
    - Each step: 5 minutes observation
```

**5. Cost optimization:**
```yaml
cost_reduction_strategies:

  strategy_1_cdn_caching:
    - Cache popular content at edge (95% cache hit rate)
    - Reduces origin bandwidth by 95%
    - Savings: $0.05/GB → $0.01/GB effective

  strategy_2_intelligent_origin:
    - Route to cheaper region if latency acceptable
    - Example: Route SA users to US (not SA) when latency OK
    - Savings: 20-30% bandwidth cost

  strategy_3_time_based_routing:
    during_low_traffic_hours:
      - Route more aggressively to cheaper regions
      - Users more tolerant of slight latency increase
      - Example: 2 AM - 6 AM, route 80% to US-East

  strategy_4_tiered_routing:
    free_tier_users: always route to cheapest region
    paid_tier_users: route to nearest region (best experience)

  projected_savings:
    baseline: $10M/year bandwidth costs
    after_optimization: $8M/year (20% reduction achieved)
```

**6. Observability requirements:**
```yaml
observability:

  metrics:
    - Routing decisions per region per minute
    - CDN cache hit rate per PoP
    - Origin bandwidth per region
    - Video start time (p50, p99)
    - Buffering rate per region
    - Cost per GB per region

  dashboards:
    - Global traffic distribution map
    - Regional health scores
    - Cost tracking (real-time spend)
    - Failover event timeline

  alerts:
    - Region unhealthy (page on-call)
    - Buffering rate > 5% (warning)
    - Bandwidth cost spike > 20% (warning)
    - Failover event (notification)

  distributed_tracing:
    - Trace each video request: DNS → CDN → Origin
    - Identify bottlenecks in playback pipeline
```

### Key insight box
> GSLB for video streaming is about balancing three constraints: latency (user experience), cost (bandwidth), and compliance (licensing).

### Final challenge question
During a live sports event, traffic to EU region spikes 10x. Your GSLB routes overflow traffic to US region. EU users complain about licensing errors ("content not available in your region"). How do you fix the routing policy to handle spikes without violating licensing?

---

## Appendix: Quick checklist (printable)

**GSLB architecture selection:**
- [ ] Define requirements (latency, cost, compliance, observability)
- [ ] Choose architecture (DNS, proxy, or hybrid)
- [ ] Consider anycast for fast failover (if BGP available)
- [ ] Plan for CDN integration (if content delivery)
- [ ] Estimate costs (infrastructure, bandwidth, operational)

**Routing policy design:**
- [ ] Define policies per request type (API, video, static)
- [ ] Balance latency, cost, capacity in scoring function
- [ ] Implement geographic routing for compliance
- [ ] Add time-based routing for cost optimization
- [ ] Support user-tier differentiation (free vs paid)

**Health checking:**
- [ ] Implement active synthetic checks
- [ ] Add passive real-user monitoring
- [ ] Define healthy/degraded/unhealthy thresholds
- [ ] Set check frequency (balance timeliness vs load)
- [ ] Create escalation triggers (when to page humans)

**Failover planning:**
- [ ] Define failure detection criteria
- [ ] Implement gradual failover (not instant)
- [ ] Add hysteresis to prevent flapping
- [ ] Plan failback strategy (gradual restoration)
- [ ] Test failover monthly (Game Days)

**Observability:**
- [ ] Track routing decisions (where is traffic going?)
- [ ] Monitor health scores per region
- [ ] Measure cost per region (bandwidth, compute)
- [ ] Implement distributed tracing (end-to-end visibility)
- [ ] Create dashboards for ops team

**Cost optimization:**
- [ ] Leverage CDN caching (reduce origin bandwidth)
- [ ] Route to cheaper regions when latency allows
- [ ] Implement time-based cost optimization
- [ ] Monitor spend per region in real-time
- [ ] Set budget alerts

**Red flags (reassess GSLB):**
- [ ] Failover takes > 60 seconds (DNS TTL too high, or no anycast)
- [ ] Users routed to wrong region frequently (policy broken)
- [ ] Bandwidth costs increased after GSLB (inefficient routing)
- [ ] Regional health checks unreliable (synthetic != real traffic)
- [ ] GSLB is a black box (no observability)
