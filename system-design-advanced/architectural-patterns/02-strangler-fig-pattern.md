---
slug: strangler-fig-pattern
title: Strangler Fig Pattern
readTime: 20 min
orderIndex: 2
premium: false
---






# Strangler Fig Pattern (Distributed Systems Edition)

CHALLENGE: You’re migrating a living, breathing monolith… without stopping the business

It’s 11:47 AM. Your checkout service is melting. Marketing just launched a “flash sale.” Support is screaming. The CEO is in a meeting promising “microservices by Q3.”

You have a monolith that:
- contains the only source of truth for orders
- has a fragile release pipeline (weekly “big bang” deploys)
- is coupled to a single relational database schema
- is relied upon by dozens of internal and external clients

The business constraint: **you can’t pause feature work** and you **can’t take downtime**.

You’re asked to “rewrite it as microservices.”

Interactive question (pause and think):

If you start a full rewrite, what’s the most likely outcome?

- A) You deliver a perfect new system in 6 months
- B) You run two systems in parallel, drift, and eventually abandon the rewrite
- C) You freeze features and complete the rewrite faster
- D) You migrate everything with no operational risk

Pause. Think.

Reveal: **B** is the common outcome.

Full rewrites tend to fail because the old system keeps changing, the new system lags behind, and you end up with **two incomplete truths**.

Key insight:
> The Strangler Fig Pattern is a strategy for **incremental replacement**: you build new capabilities around the edges of the old system, **route traffic to the new pieces**, and gradually “strangle” the old implementation until it can be removed.

---

## Section 1 — What is the Strangler Fig Pattern?

Scenario: You inherit a 15-year-old “Order Management System” (OMS). It’s stable-ish, but every change is risky. You want to modernize it into services.

Interactive question (pause and think):

What does the Strangler Fig Pattern actually *replace* first?

- A) The database
- B) The UI
- C) The “edges” (APIs/routes) and a slice of functionality
- D) The entire codebase

Pause and think.

Reveal: **C**.

Analogy: The strangler fig tree
A strangler fig starts by growing around an existing tree. Over time, it forms a lattice that takes over light and nutrients. Eventually, the original tree dies and the fig remains.

In software:
- The **monolith** is the original tree.
- The **new services** are the fig’s growth.
- The **routing layer** is the lattice that gradually shifts load.

Real-world parallel: A delivery service swapping warehouses
Imagine a delivery company with one old warehouse (slow, cramped). You build a new warehouse for “high-volume items” first. You route those items to the new warehouse, while everything else still ships from the old one. Over time, more categories move.

Mental model
The pattern is not “rewrite.” It is:
1. **Intercept** requests at the boundary.
2. **Route** some requests to new implementations.
3. **Observe** correctness and performance.
4. **Expand** coverage.
5. **Retire** the old code paths.

Key insight box:
> **Strangler Fig = controlled traffic migration at the system boundary + incremental functional replacement.**

Challenge questions
- What is the “boundary” in your system (API gateway, load balancer, UI router, message bus)?
- Which slice of functionality can be moved without touching the core data model?

---

## Section 2 — Why distributed systems make Strangler Fig both powerful and dangerous

Scenario: You plan to split the monolith into services. But once you do, you introduce:
- network failures
- partial outages
- inconsistent reads
- distributed tracing needs
- more deployment units

Interactive question (pause and think):

Which is the *biggest* new class of problems you introduce when strangling into microservices?

- A) Slower developer laptops
- B) Partial failure and coordination issues
- C) Fewer logs
- D) Less need for testing

Reveal: **B**.

Analogy: Coffee shop with one barista vs. a team
One barista (monolith) makes every drink. If they’re slow, the whole line slows, but coordination is simple.

A team (microservices) splits tasks: espresso, milk, checkout. Now:
- each station can fail independently
- you need handoffs
- you need a queue
- you need a manager to coordinate

Mental model: The “failure surface area” curve
As you adopt Strangler Fig:
- early: you add a gateway and one new service (small new surface)
- mid: you have multiple services and cross-service calls (surface grows)
- late: you retire monolith paths (surface shrinks if done right)

Key insight box:
> In distributed environments, Strangler Fig is a **traffic-shaping + failure-management** exercise as much as it is a refactoring strategy.

Challenge questions
- What’s your plan for timeouts, retries, and circuit breakers at the routing boundary?
- How will you observe correctness when traffic is split?

---

## Section 3 — Decision game: Which boundary do you strangle from?

Scenario: Your monolith serves:
- a web UI
- mobile clients
- partner APIs
- internal batch jobs

You need a “choke point” to redirect traffic.

Decision game: Which statement is true?

1) “We must start by rewriting the database.”
2) “We can start by routing at the edge (API gateway / reverse proxy) without touching the monolith internals.”
3) “Strangler Fig only works if you own all clients.”
4) “We should migrate everything behind the scenes without any routing changes.”

Pause and think.

Reveal: **2** is generally true.

Explanation
The most common strangling boundary is:
- **API gateway** (HTTP)
- **reverse proxy** (Nginx/Envoy)
- **service mesh ingress**
- **UI routing layer** (BFF)
- **message broker topics** (event-driven boundary)

If you can intercept traffic at a single point, you can gradually shift load.

Common misconception
> “Strangler Fig means you must rewrite the database last.”

Reality: database migration is often the hardest part. Sometimes you migrate data early for a slice, sometimes late; the pattern is flexible but **data is the constraint**.

Key insight box:
> Pick a boundary where you can **route**, **observe**, and **roll back**.

Challenge questions
- Where do requests enter your system today?
- Can you introduce a gateway without breaking clients?

---

## Section 4 — The three canonical phases (with distributed-systems mechanics)

Scenario: You want to move “Order History” out of the monolith first.

Phase 1: Intercept
You add a routing layer that can decide:
- monolith handles `/orders/{id}`
- new service handles `/orders/{id}/history`

[IMAGE: Architecture diagram showing clients -> gateway -> (monolith OR new service) with a routing rule for a subset of endpoints. Include observability sidecars/metrics/tracing.]

Phase 2: Implement a vertical slice
A vertical slice includes:
- API endpoint
- business logic
- data access
- authorization
- observability

Not just “a microservice skeleton.”

Phase 3: Migrate traffic and retire
You gradually increase routing percentage or route by feature.

Interactive question (pause and think):

Which traffic migration strategy is safest for distributed systems?

- A) Switch 100% traffic instantly
- B) Route by user cohort / header flag / tenant
- C) Route randomly without monitoring
- D) Route by time of day only

Reveal: **B**.

Analogy: Restaurant menu rollout
A restaurant introduces a new kitchen station for “vegan dishes.” They don’t switch the whole menu; they route a subset of orders to the new station, then expand.

Key insight box:
> Strangler Fig works best with **controlled cohorts** and **fast rollback**.

Challenge questions
- What’s your “cohort key” (tenant ID, region, user ID hash)?
- How fast can you roll back routing?

---

## Section 5 — Failure scenarios: what breaks when you split traffic?

Scenario: You route `/orders/{id}/history` to a new service. It calls the monolith for “order metadata” because you didn’t migrate that yet.

Now you have a distributed call chain:
Client -> Gateway -> HistorySvc -> Monolith

Failure scenario A: Monolith is slow
HistorySvc times out waiting for monolith.

Pause and think:

Should HistorySvc retry?

- A) Yes, retry aggressively
- B) No, never retry
- C) Retry with backoff + budget + idempotency awareness

Reveal: **C**.

Explanation: Retries amplify load (retry storms). Use:
- timeouts
- bounded retries
- jitter
- circuit breakers
- bulkheads

Failure scenario B: Split-brain business logic
You accidentally implement “refund eligibility” differently in the new service.

Common misconception
> “If endpoints are different, business rules can drift safely.”

Reality: customers experience workflows, not endpoints. Drifts create inconsistent outcomes.

Failure scenario C: Inconsistent data reads
Monolith reads from DB A. New service reads from DB B (or a replica). You route partial traffic.

This can create:
- “I see my order in one view but not another”
- double writes
- missing events

Failure scenario D: Observability blind spots
You can’t tell whether errors come from the gateway, new service, or monolith.

[IMAGE: Trace waterfall view showing request across gateway, service, monolith with latency and error annotations.]

Key insight box:
> Strangling creates **mixed call graphs**. Treat observability and failure handling as first-class features.

Challenge questions
- What are your timeout budgets per hop?
- Where will you enforce circuit breaking—gateway, service, or both?

---

## Section 6 — Routing strategies: path-based, header-based, cohort-based, and shadow traffic

Scenario: You need to move `/payments` next. This is high risk.

Strategy 1: Path-based routing
Simple rules:
- `/v2/payments/**` -> PaymentSvc
- everything else -> monolith

Pros: easy reasoning.
Cons: clients must change paths, or gateway must rewrite.

Strategy 2: Header/flag-based routing
- If `X-Use-New-Payments: true` -> new

Pros: controlled experiments.
Cons: clients (or gateway) must inject flags.

Strategy 3: Cohort/tenant routing
- Tenant A -> new
- Tenant B -> old

Pros: stable user experience per tenant.
Cons: operational complexity if tenants vary in behavior.

Strategy 4: Percentage-based canary
- 1% -> new

Pros: catches unknown unknowns.
Cons: harder to debug user reports (“sometimes it fails”).

Strategy 5: Shadow traffic (dark launch)
Send a copy of requests to new service, but don’t use response.

Pros: validate performance/correctness.
Cons: must avoid side effects.

```yaml
# Implementing Strangler Fig routing rules (Envoy-style)
# - Path-based routing to a new service
# - Header override for forced testing
# - Canary percentage split for gradual migration
routes:
  - match: { prefix: "/v2/payments" }
    route: { cluster: "payments_v2" }
  - match:
      prefix: "/payments"
      headers:
        - name: "x-use-new-payments"
          exact_match: "true"
    route: { cluster: "payments_v2" }
  - match: { prefix: "/payments" }
    route:
      weighted_clusters:
        clusters:
          - name: "payments_v2"
            weight: 1   # 1% canary
          - name: "monolith"
            weight: 99
```

Interactive matching exercise:

Match the routing strategy to the best use case:

| Strategy | Use case |
|---|---|
| Path-based | (A) Validate correctness without impacting users |
| Cohort-based | (B) High-risk migration with easy rollback |
| Shadow traffic | (C) New API version adoption |
| Percentage canary | (D) Avoid “sometimes” behavior for a tenant |

Pause and think.

Reveal:
- Path-based -> (C)
- Cohort-based -> (D)
- Shadow traffic -> (A)
- Percentage canary -> (B)

Key insight box:
> Choose routing based on **debuggability** and **blast radius**, not ideology.

Challenge questions
- Which strategy minimizes customer support confusion for your product?
- Which strategy best fits regulatory constraints (e.g., tenant isolation)?

---

## Section 7 — Data: the real boss fight

Scenario: You moved “Order History API” but the monolith still owns the `orders` table. Your new service needs history data and must not corrupt the monolith.

The core tension
Strangler Fig is easy at the API layer; it’s hard at the **data ownership** layer.

Three data migration patterns (often combined)

1) Shared database (temporary)
New service reads/writes monolith DB schema.

Pros:
- fast
- no data duplication

Cons:
- tight coupling
- schema changes become distributed coordination
- hard to enforce ownership

2) Database-per-service with sync (CDC/events)
Monolith remains system of record initially; new service builds its own store via:
- Change Data Capture (CDC)
- domain events
- outbox pattern

Pros:
- decoupling
- service autonomy

Cons:
- eventual consistency
- replay/backfill complexity

3) Dual-write (avoid if possible)
Both systems write data.

Pros:
- can transition ownership

Cons:
- race conditions
- partial failure
- reconciliation needed

Common misconception
> “Dual-write is fine if we use retries.”

Reality: retries don’t fix **split-brain writes**. You need idempotency keys, transactional outbox, or a single writer.

[IMAGE: Diagram showing monolith DB, outbox table, CDC pipeline (Debezium/Kafka), new service DB, and consumers.]

Interactive question (pause and think)

If you can only pick one principle to keep migrations sane, which is best?

- A) Every service can write any table
- B) Single writer per piece of data (clear ownership)
- C) Use synchronous replication everywhere
- D) Avoid events because they’re complex

Reveal: **B**.

Key insight box:
> Strangler Fig succeeds when you establish **clear data ownership** and use **reliable propagation** (outbox/CDC) during transition.

Challenge questions
- What entity will your first extracted service truly *own*?
- How will you backfill historical data into the new store?

---

## Section 8 — Decision game: Choose your first slice

Scenario: You have these candidates to extract:
1. Authentication
2. Order history read API
3. Payment authorization
4. Inventory reservation

Decision game: Which is the best first slice for Strangler Fig in a distributed environment?

Pick one and justify:
- A) Authentication
- B) Order history read API
- C) Payment authorization
- D) Inventory reservation

Pause and think.

Reveal (typical answer): **B**.

Why “read-mostly” slices are common first moves
- fewer side effects
- easier to validate via shadow traffic
- simpler rollback
- you can tolerate eventual consistency more often

But: if your biggest pain is auth, you might start there—Strangler Fig is context-dependent.

Key insight box:
> Start with a slice that has **low coupling**, **low write complexity**, and **high learning value**.

Challenge questions
- What slice teaches your org the most about operating services?
- What slice has the cleanest boundary and least shared state?

---

## Section 9 — Observability: proving correctness while traffic is split

Scenario: You route 10% of `/orders/{id}/history` to HistorySvc. Customers report “missing items,” but only sometimes.

What you need before serious strangling
- distributed tracing (trace IDs propagated through gateway -> services -> monolith)
- structured logs with correlation IDs
- RED metrics (Rate, Errors, Duration)
- SLOs per route and per dependency

```python
# Implementing correlation IDs + timeouts in a Strangler gateway (Python)
import json, time, uuid
from urllib import request, error

def forward_with_trace(url: str, timeout_s: float = 1.5) -> dict:
    # Generate/propagate a correlation ID for cross-service tracing
    trace_id = uuid.uuid4().hex
    req = request.Request(url, headers={"X-Correlation-Id": trace_id})
    start = time.time()
    try:
        with request.urlopen(req, timeout=timeout_s) as resp:
            body = resp.read().decode("utf-8")
            return {"trace_id": trace_id, "status": resp.status, "ms": int((time.time()-start)*1000), "body": json.loads(body)}
    except error.HTTPError as e:
        return {"trace_id": trace_id, "status": e.code, "ms": int((time.time()-start)*1000), "error": e.read().decode("utf-8")}
    except Exception as e:
        return {"trace_id": trace_id, "status": 0, "ms": int((time.time()-start)*1000), "error": str(e)}

# Usage example
# print(forward_with_trace("http://monolith.local/orders/123/history"))
```

Interactive question (pause and think)

Which metric most quickly reveals that the new service is harming users?

- A) CPU utilization
- B) p95 latency and error rate per route
- C) number of pods
- D) GC pause time

Reveal: **B**.

Mental model: “Split traffic = split truth”
You must answer:
- for the same request, do old and new produce the same result?
- if not, is it acceptable (eventual consistency) or a bug?

Technique: Differential comparison
For read endpoints, you can:
1. serve response from old system
2. shadow request to new system
3. compare results asynchronously

Key insight box:
> Without strong observability, Strangler Fig becomes **guess-and-pray**.

Challenge questions
- How will you detect semantic mismatches (not just 500 errors)?
- What’s your rollback trigger (SLO burn rate, mismatch rate, error budget)?

---

## Section 10 — Consistency and transactions: the “distributed checkout” trap

Scenario: You extracted PaymentSvc. Checkout now calls:
- CartSvc
- InventorySvc
- PaymentSvc
- OrderSvc (still monolith)

One request becomes a saga.

Pause and think

Which statement is true?

1) “We can keep ACID transactions across services by using two-phase commit everywhere.”
2) “We should accept eventual consistency and model workflows as sagas with compensations.”
3) “Distributed transactions are always impossible.”
4) “If we use Kafka, consistency is automatic.”

Reveal: **2**.

Analogy: Restaurant order with multiple stations
If dessert is out of stock after you already cooked the main dish, you don’t rewind time; you offer a substitute or refund. That’s compensation.

Distributed-systems mechanics
- avoid cross-service synchronous chains where possible
- prefer event-driven choreography for long workflows
- use idempotency keys for all side-effecting operations
- define compensations (refund, release inventory)

[IMAGE: Saga diagram showing steps with success path and compensation path.]

Common misconception
> “Strangler Fig is just routing; transactions don’t change.”

Reality: as soon as you split writes, you must redesign workflow consistency.

Key insight box:
> Strangling write paths forces you to choose: **orchestration**, **choreography**, or **keep the write in the monolith longer**.

Challenge questions
- Which operations in your monolith rely on single-DB transactions?
- What compensations make sense for your domain?

---

## Section 11 — Decision game: How do you retire the monolith safely?

Scenario: After months, 80% of traffic is handled by new services. The monolith still contains:
- legacy admin tools
- a few batch jobs
- some “rare” endpoints

Decision game: Which retirement plan is most realistic?

- A) Delete the monolith repo immediately
- B) Keep monolith running forever “just in case”
- C) Identify remaining capabilities, migrate or deprecate them, then remove code paths and data dependencies gradually
- D) Put the monolith behind a VPN and forget it

Reveal: **C**.

The “last 20%” problem
The last pieces are often:
- weird edge cases
- operational tools
- compliance reports
- batch processes tied to the old schema

Retirement checklist
- no production traffic (verify via gateway metrics)
- no hidden consumers (batch jobs, cron, partners)
- data ownership transferred
- runbook updated
- cost and risk assessed

Key insight box:
> Retiring the monolith is a **product + ops** project, not just engineering.

Challenge questions
- What are your “hidden consumers”?
- What’s your plan for legacy batch jobs (rewrite vs. replatform)?

---

## Section 12 — Common misconceptions (rapid-fire)

Scenario: You’re in a design review. You hear these statements.

Misconception 1: “Strangler Fig = microservices”
Strangler Fig is a migration approach. You can strangle into:
- a modular monolith
- a service-oriented architecture
- serverless functions

Misconception 2: “We’ll just add an API gateway and we’re done”
The gateway enables routing, but you still must handle:
- data ownership
- consistency
- observability
- operational maturity

Misconception 3: “We can ignore the monolith once traffic is low”
Low traffic endpoints can still be **high criticality** (admin, refunds, compliance).

Misconception 4: “Event-driven solves coupling automatically”
Events reduce synchronous coupling but introduce:
- schema evolution challenges
- replay semantics
- exactly-once illusions

Key insight box:
> Strangler Fig is a **socio-technical** pattern: it changes org coordination, on-call, and deployment practices.

Challenge questions
- Which misconception is most likely in your org?
- What guardrail (policy/tooling) prevents it?

---

## Section 13 — Trade-offs and when not to use Strangler Fig

Scenario: Leadership asks: “Is Strangler Fig always the best migration approach?”

When Strangler Fig is a strong fit
- you need continuous delivery during migration
- you can route traffic at a boundary
- you can tolerate incremental complexity
- you want learning and risk reduction

When Strangler Fig is risky or expensive
- no clear boundary (many clients directly hit DB)
- extremely tight latency budgets (extra hop unacceptable)
- hard real-time consistency requirements across features
- small system where rewrite cost is low

Comparison table: Strangler Fig vs Big Bang Rewrite vs Modular Monolith Refactor

| Approach | Downtime risk | Parallel run complexity | Learning curve | Data migration difficulty | Typical failure mode |
|---|---:|---:|---:|---:|---|
| Strangler Fig | Low | Medium-High | Medium | High | Never finishing; hybrid spaghetti |
| Big bang rewrite | High | High | High | High | Rewrite abandoned; feature lag |
| Modular monolith refactor | Low-Medium | Low | Medium | Medium | Doesn’t address scaling/org needs |

Interactive question (pause and think):

Which approach minimizes distributed-systems complexity?

- A) Big bang rewrite
- B) Strangler Fig into many services
- C) Modular monolith refactor

Reveal: **C** often minimizes *distributed* complexity, though it may not meet all goals.

Key insight box:
> Strangler Fig trades **migration safety** for **temporary architectural complexity**.

Challenge questions
- Is your target architecture truly required, or is modularization enough?
- What is your maximum tolerable “hybrid period” length?

---

## Section 14 — Real-world usage patterns (what teams actually do)

Scenario: You want to know how this looks in practice.

Pattern A: Gateway + BFF + extracted read services
- Start with a BFF (Backend-for-Frontend)
- Extract read models (CQRS-style)
- Use shadow traffic + diffing

Pattern B: Event outbox + CDC to build new domain stores
- Keep monolith as writer
- Publish events reliably
- New services consume and build their own stores

Pattern C: “Carve out” a high-change domain
- Identify a subdomain with high churn
- Extract it first to reduce monolith change rate

Pattern D: Strangle by message bus (not HTTP)
- Intercept at topic/queue
- Route messages to new consumers
- Gradually stop sending legacy message formats

[IMAGE: Four mini-architectures showing patterns A-D.]

Key insight box:
> Many successful stranglings start with **reads + events** before moving write ownership.

Challenge questions
- Which pattern matches your system’s integration style (HTTP vs async messaging)?
- Where can you introduce outbox/CDC with minimal risk?

---

## Section 15 — Interactive exercise: Design your strangler plan (worksheet)

Scenario: You are the tech lead. You must propose a 90-day strangler plan.

Step 1: Pick a boundary
Choose one:
- API gateway
- UI router/BFF
- message broker
- load balancer

Write down: What is your boundary and why?

Step 2: Pick your first slice
Constraints:
- low write complexity
- high visibility
- limited dependencies

Write down: What slice and what is the success metric?

Step 3: Choose routing strategy
Pick:
- cohort
- canary
- header flag
- path-based
- shadow

Write down: What is the rollback mechanism?

Step 4: Data plan
Pick:
- shared DB temporarily
- CDC/outbox
- new service owns data

Write down: Who is the single writer during transition?

Step 5: Failure plan
Define:
- timeouts
- retries
- circuit breakers
- fallbacks

Write down: What happens when the new service is down?

Key insight box:
> A strangler plan is not a diagram—it’s a **traffic + data + failure** contract.

Challenge questions
- What’s your “definition of done” for a migrated slice?
- What operational runbooks must exist before increasing traffic?

---

## Final synthesis challenge — The Strangler Fig “production readiness” quiz

Scenario: You’re about to route 30% of checkout traffic to a new PaymentSvc.

Answer these in order. Pause and think before revealing.

Q1) Where is your choke point?
- A) In the monolith controllers
- B) At the edge routing layer with metrics and fast rollback

Reveal: **B**.

Q2) What is your rollback time?
- A) Hours (redeploy)
- B) Minutes or seconds (config flip)

Reveal: **B**.

Q3) Who writes “payment authorization” truth during migration?
- A) Both systems
- B) One system, with reliable propagation to the other

Reveal: **B**.

Q4) What’s your plan for partial failure?
- A) Infinite retries
- B) Timeouts + bounded retries + circuit breakers + idempotency

Reveal: **B**.

Q5) How do you know the new service is correct?
- A) Unit tests only
- B) Shadow traffic/diffing + tracing + SLOs + semantic monitoring

Reveal: **B**.

Final key insight box:
> The Strangler Fig Pattern is successful when you treat migration as **traffic engineering + data ownership + failure design**, not just refactoring.

---

## Appendix — Quick reference checklist

- Routing boundary exists (gateway/mesh/BFF/broker)
- Cohort/canary strategy chosen
- Observability: traces, logs, metrics, SLOs
- Timeout/retry/circuit breaker policies
- Data ownership defined (single writer)
- Reliable event propagation (outbox/CDC) if needed
- Backfill and replay plan
- Semantic correctness validation (diffing)
- Runbooks and on-call readiness
- Retirement plan for last 20%
