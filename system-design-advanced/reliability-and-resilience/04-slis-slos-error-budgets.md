---
slug: slis-slos-error-budgets
title: SLIS SLOS && Error Budgets
readTime: 20 min
orderIndex: 4
premium: false
---




# SLIs, SLOs, and Error Budgets (The Mathematics of Reliability)

> Audience: SRE teams, product managers, and engineering leaders defining and measuring service reliability.

This article assumes:
- Perfect reliability (100% uptime) is impossible and economically irrational.
- Users care about their experience, not your internal metrics.
- Reliability work competes with feature development for engineering time.
- What you measure shapes what you optimize (Goodhart's Law applies).

---

##  Challenge: Your "five 9s" promise is meaningless

### Scenario
Your sales team promises customers "99.999% uptime" (five 9s).

Then reality hits:

- Your monitoring shows 99.999% uptime ✓
- But customers complain the service is "always down"
- Your CEO asks: "Why are customers angry if we're meeting our SLA?"

What went wrong?

### Interactive question (pause and think)
Which is the most likely culprit?

1. Customers are wrong (service was actually up)
2. Monitoring measures the wrong thing
3. 99.999% doesn't mean what sales thinks it means
4. All of the above

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (2) and (3).

Classic failures:
- Monitoring checks health endpoint (returns 200), but payment processing is broken
- "Uptime" measured as "server responds" not "user can complete their task"
- 99.999% allows 5 minutes downtime/year, but if it all happens during Black Friday...

### Real-world analogy (airline on-time performance)
An airline claims "99% on-time departure."

But what if:
- They measure "door closed on time" not "wheels up on time"
- Flights delayed by 59 minutes count as "on time" (60 min = late)
- They only measure the specific route you don't fly

The metric is technically accurate but completely misleading.

### Key insight
> SLIs (Service Level Indicators) must measure what users actually care about, not what's easy to measure.

### Challenge question
If you could only measure ONE metric to represent reliability, what would it be?

---

##  Mental model - Error budgets as a reliability currency

### Scenario
Your product team wants to ship a new feature. Your SRE team says "We can't deploy - we're out of error budget."

Product team: "What? We have a deadline!"

SRE team: "We agreed to 99.9% uptime. We've already had 99.85% this month. We're out of budget."

### Interactive question (pause and think)
Who's right?

A. Product team (business needs matter more than arbitrary targets)
B. SRE team (reliability is non-negotiable)
C. Both (they need a better framework for trade-offs)

### Progressive reveal
Answer: C.

This is precisely what error budgets solve: a shared framework for reliability vs velocity trade-offs.

### Mental model
Think of error budgets as:

- **Financial budget**: You have $X to spend; spend wisely
- **Risk budget**: You can afford Y failures; don't waste them
- **Innovation currency**: Reliability headroom enables experimentation

The goal: Make reliability a measurable trade-off, not a religious war.

### Real-world parallel (construction project buffers)
A building project has a time buffer. If weather delays use up the buffer, the project stops adding risky features (complex facades, custom materials). They focus on finishing reliably.

If the buffer is intact, they can afford some experimentation.

### Key insight
> Error budgets formalize the trade-off: reliability is not "maximize uptime" but "maintain agreed reliability while moving fast."

### Challenge question
If you have 99.9% SLO (43 minutes downtime/month), should you spend your error budget on:
- Testing risky new features in production
- Planned maintenance windows
- Neither (save it for unexpected failures)

---

##  What SLI, SLO, SLA actually mean (and why people confuse them)

### Scenario
Your team debates reliability targets. Three acronyms get thrown around interchangeably.

Let's clarify.

### Definitions

**SLI (Service Level Indicator):**
- **What it is**: A quantitative measure of service behavior
- **Examples**: Request success rate, latency percentile, throughput
- **Key point**: The metric itself, not the target

**SLO (Service Level Objective):**
- **What it is**: A target value or range for an SLI
- **Examples**: "99.9% of requests succeed", "p99 latency < 500ms"
- **Key point**: Internal goal, sets expectations

**SLA (Service Level Agreement):**
- **What it is**: A business contract with consequences for missing SLO
- **Examples**: "99.9% uptime or credits", "p99 < 500ms or refund"
- **Key point**: Legal/financial commitment

### Visual: relationship between SLI, SLO, SLA

```text
┌─────────────────────────────────────────────────────────┐
│         SLI → SLO → SLA RELATIONSHIP                    │
└─────────────────────────────────────────────────────────┘

SLI (Measurement):
  ├─ "What we measure"
  ├─ Request success rate (%)
  ├─ Data: 99.94%, 99.87%, 99.96%, 99.91%
  └─ No target, just observation

SLO (Target):
  ├─ "What we aim for"
  ├─ Goal: ≥ 99.9% success rate
  ├─ Internal commitment
  ├─ Drives error budget
  └─ Informs SLA

SLA (Contract):
  ├─ "What we promise legally"
  ├─ Guarantee: ≥ 99.5% success rate
  ├─ Consequence: 10% credit if violated
  ├─ More conservative than SLO
  └─ Has financial/legal teeth

Relationship:
  SLI < SLO < SLA (buffer between each)

Example:
  SLI: Measured 99.87% last month
  SLO: Target 99.9% (not met, ate error budget)
  SLA: Promised 99.5% (still met, no customer penalty)

Buffer: SLO is tighter than SLA to catch problems early
```

### Interactive question
Your SLI shows 99.6% uptime. Your SLO is 99.9%. Your SLA is 99.5%.

What happens?

A. You violated the SLA (customer gets refund)
B. You missed the SLO (internal alarm)
C. Nothing (still above SLA)
D. B and C

### Progressive reveal
Answer: D.

- Missed SLO → internal response (freeze risky changes, focus on reliability)
- Still met SLA → no customer impact, no refunds
- The buffer between SLO and SLA is intentional

### Common confusions

**Mistake 1: SLO = SLA**
- Wrong: "Our SLO is our customer promise"
- Right: "Our SLA is our customer promise; our SLO is stricter to give us early warning"

**Mistake 2: More nines = better**
- Wrong: "99.999% is better than 99.9%"
- Right: "99.999% costs exponentially more and may not improve user experience"

**Mistake 3: Uptime is the only SLI that matters**
- Wrong: "We have 99.9% uptime so we're reliable"
- Right: "Uptime doesn't measure latency, data correctness, or user-perceived availability"

### Key insight
> SLI is the thermometer. SLO is the target temperature. SLA is the contract saying "we'll keep it above freezing."

### Challenge question
Should your SLO be tighter than your SLA? By how much? What's the cost of the buffer?

---

##  Choosing good SLIs - what to measure

### Scenario
You're defining SLIs for an e-commerce checkout service.

Bad SLI: "Server CPU < 80%"
- Users don't care about your CPU
- CPU can be fine while checkout is broken

Good SLI: "Checkout success rate > 99.9%"
- Directly maps to user experience

### Interactive question (pause and think)
For a payment processing service, which SLI is most valuable?

A. Server uptime percentage
B. Request success rate (2xx HTTP responses)
C. Payment authorization success rate
D. Database query latency

### Progressive reveal
Answer: C.

- A: Servers can be "up" while payments fail
- B: 200 OK doesn't mean payment succeeded (could be 200 {"error": "declined"})
- C: Directly measures user intent
- D: Internal metric, doesn't map to user experience

### SLI selection framework

**The "user-centric" test:**
Ask: "Would a user notice if this SLI degraded?"

✓ Good SLIs (user-visible):
- Request success rate (from user's perspective)
- Request latency (time to complete user action)
- Data freshness (stale data = bad UX)
- Throughput (can system handle user load)

✗ Bad SLIs (invisible to users):
- CPU utilization (infrastructure metric)
- Memory usage (internal resource)
- Log error rate (unless it correlates with user impact)

**The "actionable" test:**
Ask: "If this SLI degrades, can we take meaningful action?"

✓ Good: "Checkout success rate dropped to 95%" → investigate payment gateway
✗ Bad: "System health score dropped to 80%" → what does that even mean?

### SLI categories

```text
┌─────────────────────────────────────────────────────────┐
│              SLI CATEGORY FRAMEWORK                     │
└─────────────────────────────────────────────────────────┘

AVAILABILITY SLIs:
  ├─ Request success rate (% of requests that succeed)
  ├─ Data availability (% of time data is accessible)
  └─ Feature availability (% of time feature works)

  Measurement:
    success_rate = successful_requests / total_requests

  Typical targets:
    - Critical path (checkout): 99.9%+
    - Nice-to-have (recommendations): 99%

LATENCY SLIs:
  ├─ p50 latency (median experience)
  ├─ p99 latency (worst-case user experience)
  ├─ p99.9 latency (tail latency, affects some users)

  Measurement:
    p99_latency = 99th percentile of response times

  Typical targets:
    - Synchronous API: p99 < 500ms
    - Search results: p50 < 100ms
    - Batch jobs: p99 < 5 seconds

THROUGHPUT SLIs:
  ├─ Requests per second (RPS)
  ├─ Transactions per second (TPS)
  └─ Data transfer rate (GB/sec)

  Measurement:
    throughput = operations / time_window

  Typical targets:
    - API: Handle 10K RPS
    - Database: Support 50K TPS

CORRECTNESS SLIs:
  ├─ Data accuracy (% of correct results)
  ├─ Processing errors (% of jobs that fail)
  └─ Data durability (% of data preserved)

  Measurement:
    correctness = correct_results / total_results

  Typical targets:
    - Financial data: 100% (zero tolerance)
    - Analytics: 99.99%
    - Recommendations: 95% (ML imperfect)

FRESHNESS SLIs:
  ├─ Data staleness (age of data)
  ├─ Replication lag (time to propagate)
  └─ Cache hit ratio (fresh data served)

  Measurement:
    freshness = % of queries served with data < X seconds old

  Typical targets:
    - Real-time dashboards: 99% of data < 5 sec old
    - Analytics reports: 99% of data < 1 hour old
```

### Example: Multi-SLI service

```yaml
service: checkout_api

slis:
  availability:
    name: "Checkout success rate"
    measurement: |
      (requests with final status = 'ORDER_PLACED') /
      (total checkout attempts)
    slo_target: 99.9%
    measurement_window: 30 days

  latency:
    name: "Checkout latency"
    measurement: "p99 time from 'begin_checkout' to 'order_placed'"
    slo_target: < 2 seconds
    measurement_window: 30 days

  correctness:
    name: "Payment accuracy"
    measurement: |
      (successful charges matching order total) /
      (total charge attempts)
    slo_target: 99.99%
    measurement_window: 30 days

# Why multiple SLIs?
# - Availability: measures if checkout works at all
# - Latency: measures if checkout is fast enough
# - Correctness: measures if we charged the right amount
# All three matter to users!
```

### Production insight: Google's "The Four Golden Signals"

Google SRE recommends focusing on:
1. **Latency**: How long requests take
2. **Traffic**: How much demand
3. **Errors**: Rate of failed requests
4. **Saturation**: How "full" the service is

Start simple. Add complexity only when needed.

### Key insight
> The best SLI is one that, when it degrades, a user has a bad experience. If users don't care, it's not an SLI - it's an internal metric.

### Challenge question
For a video streaming service, should you have separate SLIs for "start time" and "buffering rate" or combine them into one SLI?

---

##  Setting SLO targets - the Goldilocks problem

### Scenario
You need to set an SLO target for your API.

Too high (99.999%): Engineers burn out chasing impossible reliability. Costs skyrocket.
Too low (95%): Customers churn. Product team can't sell.

How do you find the right target?

### Think about it
What's the cost of each additional "9" of reliability?

### Interactive question (pause and think)
Your current reliability is 99.5%. Customers are complaining. What's your next SLO target?

A. 99.99% (skip ahead, be the best)
B. 99.9% (incremental improvement)
C. 99.5% (match current reality, then improve)

### Progressive reveal
Answer: Usually B, sometimes C.

Why not A? Going from 99.5% → 99.99% is ~10x harder than 99.5% → 99.9%. Start with achievable targets.

### The cost curve of reliability

```text
          │
  Cost    │                             ╱│
  (log    │                          ╱  │
  scale)  │                       ╱     │
          │                    ╱        │
          │                 ╱           │
          │              ╱              │
          │           ╱                 │
          │        ╱                    │
          │     ╱                       │
          │  ╱                          │
          └──────────────────────────────►
            90%   99%  99.9% 99.99% 99.999%

Key observations:
- 90% → 95%: Easy (basic monitoring + retries)
- 95% → 99%: Moderate (load balancing + failover)
- 99% → 99.9%: Expensive (multi-region + automation)
- 99.9% → 99.99%: Very expensive (complex systems)
- 99.99% → 99.999%: Extreme cost (diminishing returns)

Example: AWS uptime SLAs
- S3: 99.9% (designed for 99.999999999% durability)
- EC2: 99.99% (zone-level, not instance-level)
- DynamoDB: 99.999% (Global Tables, very expensive)
```

### SLO target selection framework

**Step 1: Measure current performance (4 weeks)**
```
Current reality:
  p99 latency: 450ms
  Success rate: 99.6%

Baseline established.
```

**Step 2: Talk to users**
```
User research:
  - "Anything under 1 second feels instant" (latency tolerance)
  - "I don't mind occasional errors, but checkout must work" (criticality)
  - "I can retry search, I can't retry payment" (feature prioritization)
```

**Step 3: Analyze impact of outages**
```
Outage analysis (last 6 months):
  - 3 outages affecting checkout (critical!)
  - 10 outages affecting recommendations (users didn't notice)
  - 1 outage affecting admin dashboard (internal only)

Conclusion: Checkout needs higher SLO than other features.
```

**Step 4: Set targets with buffer**
```
SLO targets:
  Checkout success rate: 99.9% (stretch from 99.6%)
  Checkout latency: p99 < 500ms (achievable, buffer under 1sec)
  Recommendations: 99% (lower priority)
  Admin dashboard: 95% (internal tool)
```

**Step 5: Calculate error budget**
```
99.9% SLO = 0.1% error budget = 43 minutes downtime/month

How to spend:
  - Planned maintenance: 10 min
  - Risky deployments: 20 min
  - Unknown failures: 13 min buffer
```

### Common SLO target mistakes

**Mistake 1: Matching competitor SLOs blindly**
- Wrong: "AWS promises 99.99%, we should too"
- Right: "Our users need 99.9%, and we can deliver it reliably"

**Mistake 2: One-size-fits-all SLOs**
- Wrong: "All our services have 99.9% uptime SLO"
- Right: "Critical: 99.9%, important: 99%, nice-to-have: 95%"

**Mistake 3: Aspirational SLOs**
- Wrong: "We're at 99%, let's set SLO to 99.99% to motivate the team"
- Right: "We're at 99%, let's set SLO to 99.5% and actually achieve it, then increase"

**Mistake 4: Forgetting dependencies**
- Wrong: "We'll be 99.99% reliable" (but your database vendor only promises 99.9%)
- Right: "Our SLO must account for dependency SLOs" (can't exceed weakest link)

### Key insight
> Your SLO should be: (1) Achievable with current architecture, (2) Meaningful to users, (3) Tighter than your SLA. If it's not all three, revisit.

### Challenge question
Your SLO is 99.9%, but you're consistently achieving 99.95%. Should you tighten your SLO or keep the buffer?

---

##  Error budgets - the engine of risk-taking

### Scenario
Month starts:
- SLO: 99.9% success rate
- Error budget: 43 minutes of downtime allowed

By mid-month:
- 30 minutes of downtime already burned (bad deployment)
- 13 minutes remaining

Product team wants to ship a major feature.

### Interactive question (pause and think)
What should you do?

A. Ship anyway (business pressure)
B. Freeze deploys until next month (protect SLO)
C. Ship only low-risk changes (burn budget carefully)
D. Revise SLO downward (adjust expectations)

### Progressive reveal
Answer: C, with team agreement.

Error budgets are meant to be spent, not hoarded. But spend wisely.

### Error budget mechanics

**Definition:**
```
Error budget = (1 - SLO) × time period

Example:
  SLO: 99.9%
  Time: 30 days
  Error budget: 0.1% × 30 days × 24 hours × 60 min
              = 43.2 minutes
```

**Consumption tracking:**
```
Error budget consumed = (actual errors / total requests) × time period

Real-time tracking:
  Hour 1: 99.95% success → consumed 0.05% × 1 hour = 0.03 min
  Hour 2: 99.80% success → consumed 0.20% × 1 hour = 0.12 min
  ...
  Month total: 30 minutes consumed, 13 minutes remaining
```

**Policy framework:**
```yaml
error_budget_policy:

  # When >50% budget remains (healthy):
  burn_rate: < 50%
  actions:
    - Normal deployment cadence
    - Can test risky features in prod
    - Innovation encouraged

  # When 25-50% budget remains (caution):
  burn_rate: 25-50%
  actions:
    - Reduce deployment frequency
    - Extra review for risky changes
    - Focus on stabilization

  # When <25% budget remains (critical):
  burn_rate: < 25%
  actions:
    - Freeze feature deploys
    - Only reliability improvements or critical fixes
    - Incident post-mortems for all outages
    - Consider declaring incident (proactive)

  # When budget exhausted:
  burn_rate: 100%
  actions:
    - Full deploy freeze (except rollbacks)
    - Focus on reliability improvements only
    - Root cause analysis for all incidents
    - Report to leadership
```

### Example: Error budget in practice

```go
package slo

import (
    "time"
)

type ErrorBudget struct {
    SLO               float64 // e.g., 0.999 for 99.9%
    Window            time.Duration
    TotalRequests     int64
    FailedRequests    int64
    BudgetStart       time.Time
}

func (eb *ErrorBudget) AllowedFailures() int64 {
    return int64(float64(eb.TotalRequests) * (1 - eb.SLO))
}

func (eb *ErrorBudget) RemainingBudget() int64 {
    return eb.AllowedFailures() - eb.FailedRequests
}

func (eb *ErrorBudget) BudgetPercentRemaining() float64 {
    if eb.AllowedFailures() == 0 {
        return 0
    }
    return float64(eb.RemainingBudget()) / float64(eb.AllowedFailures())
}

func (eb *ErrorBudget) CanAffordFailure(expectedFailures int64) bool {
    return expectedFailures <= eb.RemainingBudget()
}

// Usage
func main() {
    budget := &ErrorBudget{
        SLO:            0.999,  // 99.9%
        Window:         30 * 24 * time.Hour,
        TotalRequests:  10_000_000,
        FailedRequests: 7_500,  // 0.075% error rate so far
        BudgetStart:    time.Now().Add(-15 * 24 * time.Hour), // 15 days in
    }

    allowedFailures := budget.AllowedFailures()
    // 10M * (1 - 0.999) = 10,000 allowed failures

    remaining := budget.RemainingBudget()
    // 10,000 - 7,500 = 2,500 failures remaining

    percentRemaining := budget.BudgetPercentRemaining()
    // 2,500 / 10,000 = 25% budget remaining

    // Can we afford a risky deploy that might cause 3,000 failures?
    if budget.CanAffordFailure(3000) {
        // No - would exceed budget
        println("Deploy blocked: insufficient error budget")
    }
}
```

### Burn rate alerting

```text
BURN RATE ALERTING:

Fast burn (critical):
  If SLO is 99.9% (0.1% error budget/month)
  And error rate is 1% (10x the budget rate)
  Then budget will exhaust in 3 days

  Alert: "Critical: 10x burn rate, budget exhausted in 3 days"

Slow burn (warning):
  If error rate is 0.2% (2x the budget rate)
  Then budget will exhaust in 15 days

  Alert: "Warning: 2x burn rate, budget exhausted in 15 days"

Multi-window alerting (Google SRE approach):
  - 1 hour window: catch fast burns
  - 6 hour window: catch medium burns
  - 3 day window: catch slow burns

  Alert if: (burn_rate > threshold) in ANY window
```

### Production insight: How Netflix uses error budgets

Netflix:
- Each service has error budget
- Teams can spend budget on experiments (chaos, canary, A/B tests)
- When budget low, automated policies kick in (deploy freeze)
- Incentivizes reliability: more reliable = more freedom to innovate

### Key insight
> Error budgets convert reliability from a vague goal into a concrete currency. You can spend it on innovation or hoard it. Choose wisely.

### Challenge question
Your error budget is 43 min/month. Should you spend it all on: (1) one big risky feature launch, or (2) many small experiments? What's the trade-off?

---

##  Common SLO anti-patterns

### Scenario
You've set SLOs. But things feel off. Users are still unhappy. Engineers are frustrated.

### SLO anti-patterns catalog

**Anti-pattern 1: Vanity SLOs**
```yaml
# Bad
slo: "99.999% uptime"

Reality:
  - Actual: 99.5%
  - Team can't achieve 99.999%
  - SLO is aspirational, not achievable
  - Error budget permanently exhausted
  - Team morale crushed
```

**Anti-pattern 2: Invisible SLOs**
```yaml
# Bad
slo: "System performance is good"

Problems:
  - Not measurable
  - No clear target
  - Can't calculate error budget
  - Can't hold anyone accountable
```

**Anti-pattern 3: Inside-out SLOs**
```yaml
# Bad
slo: "Server CPU < 70%"

Why bad:
  - Users don't care about CPU
  - CPU can be fine while service is broken
  - Optimizing for wrong thing
```

**Anti-pattern 4: One-size-fits-all**
```yaml
# Bad
all_services:
  slo: 99.9% uptime

Reality:
  - Checkout needs 99.95%
  - Analytics can tolerate 95%
  - Treating them same wastes money
```

**Anti-pattern 5: No SLO enforcement**
```yaml
# Bad
slo: 99.9%
policy: "Try our best"

Problems:
  - SLO violated regularly
  - No error budget policy
  - No accountability
  - SLO becomes theatre
```

**Anti-pattern 6: Missing user-journey SLOs**
```yaml
# Bad - component-level only
api_gateway: 99.9%
service_a: 99.9%
service_b: 99.9%
database: 99.9%

# Good - user-journey SLO
checkout_end_to_end: 99.9%
  # Accounts for all component failures combined
```

### Real-world failure: Compound SLO violations

```text
The false confidence scenario:

Service architecture:
  User → API Gateway → Service A → Service B → Database

Individual SLOs (all 99.9%):
  - API Gateway: 99.9%
  - Service A: 99.9%
  - Service B: 99.9%
  - Database: 99.9%

Expected end-to-end reliability:
  0.999 × 0.999 × 0.999 × 0.999 = 0.996 = 99.6%

Reality is worse:
  - Failures are NOT independent
  - Cascade failures (if B fails, A times out, Gateway times out)
  - Actual: 99.2%

User experience:
  - Users promised 99.9% (SLA)
  - Actual: 99.2%
  - SLA violated
  - Customer churn

Fix:
  - Measure end-to-end user journey SLO
  - Set component SLOs tighter to account for composition
```

### Key insight
> Bad SLOs are worse than no SLOs. They create false confidence and misallocate engineering effort.

### Challenge question
If your SLO is consistently exceeded (always 99.99% when target is 99.9%), is that good or bad?

---

##  Measuring SLIs in practice - implementation patterns

### Scenario
You've defined your SLIs. Now you need to actually measure them.

Where do you measure? How often? What's the source of truth?

### Measurement approaches

**Client-side measurement:**
```javascript
// Measure from user's browser
const start = performance.now();

fetch('/api/checkout')
  .then(response => {
    const latency = performance.now() - start;
    const success = response.ok;

    // Send to analytics
    track('checkout_request', {
      latency_ms: latency,
      success: success,
      user_id: getCurrentUser()
    });
  });
```

✓ **Pros:**
- Sees actual user experience (network latency, client-side rendering)
- Accounts for CDN, geographic distribution

✗ **Cons:**
- Can't track requests that never arrive (network failures)
- Sampling bias (only successful page loads report)
- User privacy concerns

**Server-side measurement:**
```go
// Measure from backend
func HandleCheckout(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    defer func() {
        latency := time.Since(start)
        status := w.StatusCode

        metrics.RecordCheckout(
            latency,
            status >= 200 && status < 300,
        )
    }()

    // Process checkout
    result := processCheckout(r)
    w.WriteHeader(result.StatusCode)
    json.NewEncoder(w).Encode(result)
}
```

✓ **Pros:**
- Complete picture (all requests, including failures)
- No sampling bias
- Easier to implement at scale

✗ **Cons:**
- Doesn't see client-side latency (network, rendering)
- May miss user-perceived failures (e.g., JS errors)

**Synthetic monitoring (probes):**
```yaml
# Periodic health checks from multiple regions
synthetic_check:
  endpoint: https://api.example.com/checkout
  interval: 60 seconds
  timeout: 5 seconds
  regions: [us-east, us-west, eu-west, ap-south]

  assertions:
    - status_code: 200
    - latency_p99: < 500ms
    - body_contains: "success"
```

✓ **Pros:**
- Detects issues before users do
- Consistent baseline (not affected by traffic patterns)
- Works even when no user traffic

✗ **Cons:**
- May not match real user behavior
- Can't measure user-specific features (auth, personalization)
- Creates artificial load

### Recommended: Multi-layer measurement

```text
┌─────────────────────────────────────────────────────────┐
│         MEASUREMENT LAYERS (REDUNDANT)                  │
└─────────────────────────────────────────────────────────┘

Layer 1: Synthetic probes (always-on baseline)
  ├─ Purpose: Early warning, catch issues before users
  ├─ Coverage: Core API endpoints
  └─ Frequency: Every 60 seconds

Layer 2: Server-side instrumentation (ground truth)
  ├─ Purpose: Authoritative SLI measurement
  ├─ Coverage: All requests (100% sample)
  └─ What: Success rate, latency, throughput

Layer 3: Client-side RUM (Real User Monitoring)
  ├─ Purpose: True user experience
  ├─ Coverage: 5% sample (performance overhead)
  └─ What: Page load time, JS errors, user flows

Layer 4: Load balancer logs (backup)
  ├─ Purpose: Redundancy if app crashes
  ├─ Coverage: All HTTP traffic
  └─ What: Status codes, latency, paths

All layers feed into SLI dashboard
If layers disagree → investigate (usually Layer 3 sees problems first)
```

### SLI calculation methods

**Method 1: Request-based (most common)**
```sql
-- Calculate success rate SLI
SELECT
  DATE_TRUNC('hour', timestamp) as hour,
  COUNT(*) as total_requests,
  SUM(CASE WHEN status_code < 500 THEN 1 ELSE 0 END) as successful,
  (SUM(CASE WHEN status_code < 500 THEN 1 ELSE 0 END)::float /
   COUNT(*)::float) * 100 as success_rate_pct
FROM requests
WHERE timestamp >= NOW() - INTERVAL '30 days'
  AND endpoint = '/api/checkout'
GROUP BY hour
ORDER BY hour;

-- Calculate p99 latency SLI
SELECT
  DATE_TRUNC('day', timestamp) as day,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99_latency_ms
FROM requests
WHERE timestamp >= NOW() - INTERVAL '30 days'
  AND endpoint = '/api/checkout'
GROUP BY day
ORDER BY day;
```

**Method 2: Windows-based (for batch/streaming)**
```python
# Calculate data freshness SLI for analytics pipeline
def calculate_freshness_sli(window_hours=24):
    """
    SLI: 99% of analytics queries should return data < 1 hour old
    """
    queries = get_analytics_queries(last_n_hours=window_hours)

    fresh_count = 0
    total_count = len(queries)

    for query in queries:
        data_age = query.execution_time - query.data_timestamp
        if data_age < timedelta(hours=1):
            fresh_count += 1

    freshness_sli = (fresh_count / total_count) * 100
    return freshness_sli

# Target: 99%
# Measured: 97.4%
# Error budget consumed: 2.6% vs 1% allowed
```

**Method 3: Burn rate (for error budgets)**
```go
func CalculateBurnRate(slo float64, window time.Duration) float64 {
    // Burn rate = (actual error rate) / (error budget rate)
    errorBudgetRate := 1 - slo

    actualErrorRate := getActualErrorRate(window)

    burnRate := actualErrorRate / errorBudgetRate
    // burnRate > 1 means burning faster than sustainable

    return burnRate
}

// Example:
// SLO: 99.9% (error budget: 0.1%)
// Actual error rate: 1%
// Burn rate: 1% / 0.1% = 10x
// At this rate, error budget exhausted in 3 days
```

### Key insight
> SLI measurement is not "set it and forget it." Continuously validate: does this metric reflect user experience? If not, adjust.

### Challenge question
Your server-side SLI shows 99.9% success rate, but your client-side RUM shows 95% success rate. What's happening?

---

## Final synthesis - Design a complete SLO framework

### Synthesis challenge
You're the SRE lead for a ride-sharing platform.

Requirements:
- 50 million rides/month
- Mobile app (iOS + Android)
- Backend services: ride matching, payments, notifications, maps
- Users care about: finding a ride quickly, accurate pricing, reliable payment

Current state:
- No SLOs defined
- Monitoring exists but ad-hoc
- Incidents happen but no clear severity criteria
- Engineers debate "is this reliable enough?" constantly

### Your tasks (pause and think)
1. Define 3-5 key SLIs (what to measure?)
2. Set SLO targets for each SLI (what's good enough?)
3. Calculate error budgets (how much failure is acceptable?)
4. Design error budget policy (what happens when budget runs low?)
5. Implement measurement strategy (how to track?)
6. Define escalation triggers (when to page someone?)

Write down your framework.

### Progressive reveal (one possible solution)

**1. Key SLIs:**
```yaml
slis:

  ride_request_success:
    definition: >
      % of ride requests that result in matched driver
      within 2 minutes
    measurement: >
      (ride_requests_matched_within_2min / total_ride_requests) * 100
    user_impact: CRITICAL (core product function)

  payment_success:
    definition: >
      % of payment attempts that successfully charge
      customer and pay driver
    measurement: >
      (successful_charges / total_charge_attempts) * 100
    user_impact: CRITICAL (revenue + user trust)

  app_latency:
    definition: >
      p99 time from "request ride" button tap to
      "driver assigned" notification
    measurement: >
      PERCENTILE_99(driver_assigned_time - request_time)
    user_impact: HIGH (user experience)

  map_rendering:
    definition: >
      % of map loads that complete within 3 seconds
    measurement: >
      (map_loads_under_3sec / total_map_loads) * 100
    user_impact: MEDIUM (nice-to-have)

  notification_delivery:
    definition: >
      % of push notifications delivered within 10 seconds
    measurement: >
      (notifications_delivered_under_10sec / total_notifications) * 100
    user_impact: MEDIUM (information, not critical)
```

**2. SLO targets:**
```yaml
slo_targets:

  ride_request_success:
    target: 99.5%
    rationale: >
      - Current: 98.2%
      - User research shows tolerance up to 99%
      - 99.5% is achievable improvement
      - More stringent than 99% competitor average

  payment_success:
    target: 99.9%
    rationale: >
      - Current: 99.7%
      - Payment failures = lost revenue + support cost
      - Stricter than ride matching (financial impact)

  app_latency:
    target: p99 < 10 seconds
    rationale: >
      - Current: p99 = 15 seconds
      - User research: 10 sec feels acceptable
      - Accounts for network variance + backend processing

  map_rendering:
    target: 95%
    rationale: >
      - Current: 92%
      - Nice-to-have feature
      - Users can retry
      - Lower priority than core features

  notification_delivery:
    target: 99%
    rationale: >
      - Current: 97%
      - Important but not blocking (user can check app)
      - Mobile platform limitations (iOS/Android delivery not guaranteed)
```

**3. Error budgets:**
```yaml
error_budgets:

  ride_request_success:
    slo: 99.5%
    window: 30 days
    total_requests_per_month: 50_000_000
    allowed_failures: 250_000  # 0.5% of 50M
    error_budget_minutes: 216  # 0.5% of 43,200 min/month

  payment_success:
    slo: 99.9%
    window: 30 days
    total_payments_per_month: 45_000_000  # not all rides require payment
    allowed_failures: 45_000  # 0.1% of 45M
    error_budget_minutes: 43  # 0.1% of 43,200 min/month

  app_latency:
    slo: p99 < 10 seconds
    window: 30 days
    # Budget: 1% of requests can exceed 10s
    # (99% must be under 10s, so 1% budget)
    allowed_slow_requests: 500_000  # 1% of 50M
```

**4. Error budget policy:**
```yaml
error_budget_policy:

  thresholds:
    healthy: budget_remaining > 50%
    caution: budget_remaining 25-50%
    critical: budget_remaining < 25%
    exhausted: budget_remaining = 0%

  actions:

    healthy:
      - Normal deployment cadence (multiple deploys/day)
      - Can run experiments (A/B tests, feature flags)
      - Innovation encouraged
      - No restrictions

    caution:
      - Reduce deploy frequency (max 1/day)
      - Extra code review for risky changes
      - Require rollback plan for all deploys
      - Postmortem for every incident

    critical:
      - Deploy freeze (only critical fixes + rollbacks)
      - Emergency change advisory board approval required
      - All hands on stability
      - Daily SLO review meetings

    exhausted:
      - Full deploy freeze
      - Escalate to VP Engineering
      - Public incident declared
      - Focus 100% on reliability
      - Resume deploys only after error budget partially restored

  burn_rate_alerts:
    fast_burn:
      window: 1 hour
      threshold: 10x normal burn rate
      action: Page on-call immediately

    medium_burn:
      window: 6 hours
      threshold: 5x normal burn rate
      action: Alert SRE team (not page)

    slow_burn:
      window: 3 days
      threshold: 2x normal burn rate
      action: Email engineering leadership
```

**5. Measurement strategy:**
```yaml
measurement:

  primary_source: server_side
    location: API gateway + backend services
    coverage: 100% of requests
    latency: < 10ms overhead
    storage: TimescaleDB (time-series optimized)

  secondary_source: client_side_rum
    location: Mobile app (iOS + Android SDK)
    coverage: 5% sample (reduce client battery drain)
    metrics: [app_latency, map_rendering, errors]

  synthetic_monitoring:
    frequency: every 60 seconds
    regions: [us-east, us-west, eu-west, ap-south]
    endpoints:
      - /health (expect 200 OK)
      - /api/ride/request (synthetic ride request)
      - /api/payment/health

  sli_dashboard:
    - Grafana dashboard with all SLIs
    - 30-day rolling window
    - Real-time error budget consumption
    - Burn rate graphs
    - Incident annotations

  alerting:
    - PagerDuty for critical SLO violations
    - Slack for warnings
    - Email for slow burns
```

**6. Escalation triggers:**
```yaml
escalation:

  sev1_page_immediately:
    - ride_request_success < 95% for 5 minutes
    - payment_success < 99% for 5 minutes
    - Error budget burn rate > 10x for 1 hour

  sev2_alert_team:
    - ride_request_success < 99% for 15 minutes
    - app_latency p99 > 15 seconds for 30 minutes
    - Error budget < 25% remaining

  sev3_email_stakeholders:
    - SLO violated (but recovered)
    - Error budget < 50% remaining
    - Slow burn (2x normal rate for 3 days)

  incident_commander_triggers:
    - Multiple SLOs violated simultaneously
    - Error budget exhausted
    - Customer escalations correlated with metrics
```

### Key insight
> A complete SLO framework turns reliability from art to science: measurable, predictable, and blameless.

### Final challenge question
Your SLOs are met (99.9% success rate), but your customer NPS (Net Promoter Score) is dropping. What's wrong with your SLO framework?

---

## Appendix: Quick checklist (printable)

**SLI definition checklist:**
- [ ] Measures something users care about (not just infrastructure)
- [ ] Quantifiable (can calculate a percentage or latency)
- [ ] Actionable (degradation leads to clear debugging)
- [ ] Fresh enough (measured with acceptable latency)
- [ ] Covers the right scope (end-to-end, not just component)

**SLO target checklist:**
- [ ] Based on current performance (not aspirational)
- [ ] Informed by user research (what do users tolerate?)
- [ ] Tighter than SLA (buffer for early warning)
- [ ] Achievable with current architecture
- [ ] Different for different criticality features

**Error budget checklist:**
- [ ] Calculated from SLO (formula applied correctly)
- [ ] Tracked in real-time (dashboard + alerts)
- [ ] Policy defined (what happens at 50%, 25%, 0%?)
- [ ] Team agrees on policy (not imposed top-down)
- [ ] Budget is actually spent (not hoarded)

**Measurement checklist:**
- [ ] Multi-layer (server-side + client-side + synthetic)
- [ ] Sufficient granularity (at least hourly, ideally per-request)
- [ ] Handles edge cases (what about 500 errors? timeouts?)
- [ ] Survives component failures (logging doesn't depend on service being up)
- [ ] Privacy-compliant (no PII in metrics)

**Operational checklist:**
- [ ] SLO dashboard accessible to all engineers
- [ ] Burn rate alerts configured (fast, medium, slow windows)
- [ ] Error budget policy documented and followed
- [ ] Incident post-mortems reference SLO impact
- [ ] Quarterly SLO review (adjust targets based on reality)

**Red flags (reassess SLOs):**
- [ ] SLO consistently exceeded by >1% (too loose, or lucky)
- [ ] SLO consistently missed (too tight, or systemic issue)
- [ ] Error budget always exhausted (targets unrealistic)
- [ ] Error budget never spent (targets too loose, or culture problem)
- [ ] Users complain despite meeting SLOs (measuring wrong thing)
