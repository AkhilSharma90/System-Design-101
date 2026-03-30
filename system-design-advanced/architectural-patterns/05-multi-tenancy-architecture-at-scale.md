---
slug: multi-tenancy-architecture
title: Multi Tenancy Architecture
readTime: 20 min
orderIndex: 5
premium: false
---







# Multi-Tenancy Architecture at Scale

Audience: senior engineers/architects building SaaS platforms and internal multi-tenant platforms.
Goal: build a mental model for multi-tenancy in distributed systems, learn trade-offs, and practice decisions via interactive exercises.

---

## Challenge 0: "We're going enterprise... and everything breaks"

### Scenario
You run a successful SaaS product. Early customers were similar in size and usage. Now an enterprise customer arrives with:
- 50k users
- strict compliance requirements (data residency, encryption, audit)
- "no noisy neighbors" expectations
- SLAs with financial penalties

Your current architecture is "single environment, shared everything." It worked - until it didn't.

### Interactive question (pause and think)
If you change only one thing first to survive the enterprise onboarding, what should it be?

A) Add more database replicas
B) Add per-tenant rate limits and isolation controls
C) Rewrite the app into microservices
D) Move to Kubernetes

Pause and think. Which one is most leverage?

### Progressive reveal (answer)
B) Per-tenant rate limits and isolation controls is often the fastest way to stop one tenant from destabilizing others.

Scaling infrastructure (A/D) helps, but without fairness controls you amplify the blast radius. Rewriting into microservices (C) is usually too slow for the immediate problem.

### Production insight
Rate limiting is necessary but not sufficient. In production you typically need admission control (reject/queue early), per-tenant concurrency caps, and bulkhead isolation (separate pools) for the most expensive paths.

### Real-world analogy: the coffee shop
A coffee shop has one espresso machine (shared resource). If one customer orders 200 cappuccinos for an office, everyone else waits. The first fix isn't buying a new building; it's:
- limiting huge orders,
- batching,
- adding a second barista lane,
- and scheduling.

### Key insight
> Multi-tenancy is not a database schema trick. It's a system of isolation boundaries, fairness, and failure containment across compute, storage, and control planes.

### Challenge question
If you implement per-tenant rate limiting, where do you enforce it first: edge, service, or database? Why?

---

## Section 1: What "multi-tenancy" means in distributed systems

### Scenario
Your product manager says: "We need multi-tenancy." Your compliance officer says: "We need tenant isolation." Your SRE says: "We need blast-radius control." Everyone is right - but talking past each other.

### Interactive question (pause and think)
Which statement is most accurate?

1) Multi-tenancy means multiple customers share the same database tables.
2) Multi-tenancy means multiple customers share the same application instance.
3) Multi-tenancy means multiple customers share some resources with enforceable isolation and identity boundaries.
4) Multi-tenancy is the same as running one cluster per customer.

Pause and think.

### Progressive reveal (answer)
3) is the best definition.
- You can be multi-tenant with separate DBs but shared compute.
- Or shared DB but separate compute.
- Or separate everything (which becomes "single-tenant deployments," but still under a multi-tenant control plane).

### Mental model: "tenants are security + scheduling domains"
Treat a tenant as:
- Identity domain: authN/authZ boundaries, audit scope.
- Scheduling domain: fairness, quotas, rate limits.
- Failure domain: how outages and overload propagate.
- Billing domain: metering, cost attribution.

### Real-world parallel: a food court
A food court is shared seating (shared resource), but each restaurant has its own kitchen (isolation). Some share dishwashing (shared service). The experience depends on which resources are shared and how contention is managed.

### Key insight
> Multi-tenancy choices are primarily about where you draw boundaries for security, performance, and operations.

### Challenge question
Name three resources in your system that become "shared contention points" under multi-tenancy.

---

## Section 2: The three isolation axes (and why people confuse them)

### Scenario
A tenant complains: "Your system is slow." Another tenant says: "Your system leaked data." A third says: "Your outage took us down." These are three different isolation failures.

### Interactive question (pause and think)
Match the complaint to the isolation axis:

A) Security isolation
B) Performance isolation
C) Fault isolation

1) "We saw another tenant's invoice in our UI."
2) "Every Monday at 9am we time out when a big tenant runs reports."
3) "Your background job worker crashed and all tenants' exports stopped."

Pause and think.

### Progressive reveal (answer)
1 -> A, 2 -> B, 3 -> C.

### Common misconception
> "If we have security isolation, we automatically have performance and fault isolation."

No. You can perfectly isolate data access while still sharing a DB connection pool that collapses under one tenant's traffic.

### [IMAGE: Triangle diagram showing Security/Performance/Fault isolation axes, with examples of boundaries (DB schema, queues, pods, clusters) mapped onto each axis.]

### Key insight
> Multi-tenancy is a vector of isolations, not a binary property.

### Challenge question
Which axis is hardest to retrofit late: security, performance, or fault isolation? Explain based on your system.

---

## Section 3: Multi-tenancy patterns (shared -> isolated) and when to use them

### Scenario
You're designing a platform for 5,000 tenants. Tenants range from hobbyists to Fortune 50. You need a strategy that doesn't create 5,000 snowflakes.

### Interactive question (pause and think)
You can choose one of these deployment models:

- Model S: Shared everything (one cluster, one DB)
- Model P: Pooled compute, separate DB per tenant
- Model C: Cell-based (many small stacks; tenants assigned to a cell)
- Model D: Dedicated single-tenant stacks for large customers, pooled for the rest

Which model best supports both long-tail efficiency and enterprise isolation?

Pause and think.

### Progressive reveal (answer)
Model D is common in practice: "hybrid multi-tenancy."
- Long tail: pooled resources for efficiency.
- Top tier: dedicated or semi-dedicated for isolation and compliance.

### Comparison table
| Pattern | What's shared? | Pros | Cons | When it fits |
|---|---|---|---|---|
| Shared DB schema (row-level) | tables, indexes, caches | cheapest, simplest ops | noisy neighbors, harder compliance, hotspot indexes | early stage, homogeneous tenants |
| Shared DB, separate schema | DB instance, but per-tenant schema | better logical isolation | still shared IO + connection limits; catalog bloat | mid-stage, moderate isolation |
| Separate DB per tenant | compute shared, storage isolated | stronger isolation, easier data residency; easier per-tenant PITR | many DBs to manage; migrations complex; connection storms | B2B SaaS with compliance |
| Cell-based | each cell is mini-stack | blast radius limited, scalable ops | capacity planning; tenant placement; cross-cell queries hard | large scale SaaS |
| Dedicated stack | nothing shared | strongest isolation | expensive; operational overhead | top enterprise |

### Real-world analogy: delivery service
- Shared everything: one van delivers for all neighborhoods.
- Cell-based: multiple vans, each assigned to a zone.
- Dedicated: VIP customers get a private courier.

### Key insight
> At scale, you don't pick one pattern - you build a portfolio and a control plane to place tenants onto the right tier.

### Challenge question
What would be your promotion policy from pooled -> dedicated? (e.g., revenue, compliance, usage profile)

---

## Section 4: Decision game - choose your tenant boundary

### Scenario
You operate a service with:
- API traffic (latency-sensitive)
- background analytics jobs (CPU-heavy)
- file uploads (IO-heavy)

Your CTO asks: "Where do we enforce tenant isolation?"

### Decision game: Which statement is true?
Pick two that are true in most systems:

1) If you isolate tenants at the database layer, you don't need isolation at the queue layer.
2) If you isolate tenants at the Kubernetes namespace level, you automatically isolate network and IAM.
3) If you isolate tenants at the request routing layer (API gateway), you can prevent many cascade failures.
4) If you isolate tenants by giving each one a separate cache cluster, you eliminate noisy neighbors.

Pause and think.

### Progressive reveal (answer)
3 is true: early throttling and routing decisions can contain overload.

4 is partially true: separate caches reduce contention, but you can still have shared upstream bottlenecks (DB, workers, shared services). Also cost explodes and operational complexity increases.

1 false: queues can still be dominated by one tenant; async backlogs can starve others.

2 false: namespaces help, but network policies, IAM, and node-level isolation must be configured; many clusters treat namespaces as soft boundaries.

### Key insight
> Isolation is defense-in-depth: edge -> service -> storage -> background systems.

### Challenge question
List your top 3 "shared chokepoints" and where you'd add tenant-aware controls.

---

## Section 5: Tenant identity propagation (the hidden distributed systems problem)

### Scenario
A request enters your system with tenant_id = t-123. It triggers:
- synchronous calls to 5 microservices
- async events to Kafka
- a background job
- a data export to object storage

Six months later, an auditor asks: "Prove that tenant t-123 data never crossed into t-456."

### Interactive question (pause and think)
Where can tenant identity be lost?

A) At service-to-service RPC boundaries
B) In async messaging headers
C) In batch jobs reading from shared storage
D) In logs/metrics/traces
E) All of the above

Pause and think.

### Progressive reveal (answer)
E) All of the above.

### Mental model: "tenant_id is like a baggage tag"
In an airport, your bag changes conveyors, planes, and handlers. If the tag falls off once, the bag can end up anywhere.

In distributed systems, tenant identity must be:
- authenticated at ingress,
- propagated through calls/events,
- validated at each boundary,
- recorded in audit trails.

### Production clarifications
- Prefer signed, non-forgeable tenant context (e.g., JWT claims validated at ingress + internal service identity) over trusting raw headers.
- Treat tenant context as mandatory: "no tenantless work."
- For internal calls, use mTLS + SPIFFE/SPIRE (or equivalent) and authorize service identities to act on tenant scopes.

### [CODE: Go, gRPC interceptor that extracts tenant_id from metadata, injects into context, and enforces presence on outbound calls]

```go
// Tenant identity propagation via gRPC interceptors.
// - Validates tenant_id presence on inbound requests.
// - Injects tenant_id into outbound metadata.
// Production notes:
// - Validate JWT at ingress; internal services should not trust arbitrary headers.
// - Consider signed tenant context tokens for async workflows.

package tenantctx

import (
    "context"
    "errors"

    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
)

type ctxKey string

const tenantKey ctxKey = "tenant_id"

func TenantIDFromContext(ctx context.Context) (string, bool) {
    v := ctx.Value(tenantKey)
    s, ok := v.(string)
    return s, ok && s != ""
}

func WithTenantID(ctx context.Context, tenantID string) context.Context {
    return context.WithValue(ctx, tenantKey, tenantID)
}

func UnaryServerInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        md, _ := metadata.FromIncomingContext(ctx)
        vals := md.Get("x-tenant-id")
        if len(vals) == 0 || vals[0] == "" {
            return nil, errors.New("missing tenant_id")
        }
        ctx = WithTenantID(ctx, vals[0])
        return handler(ctx, req)
    }
}

func UnaryClientInterceptor() grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        tenantID, ok := TenantIDFromContext(ctx)
        if !ok {
            return errors.New("tenantless outbound call")
        }
        md, _ := metadata.FromOutgoingContext(ctx)
        md = md.Copy()
        md.Set("x-tenant-id", tenantID)
        ctx = metadata.NewOutgoingContext(ctx, md)
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

### [CODE: Java, Kafka producer/consumer example adding tenant_id header and validating before processing]

```java
// Kafka tenant header propagation (producer + consumer).
// Production notes:
// - Validate tenant header on consume; route invalid messages to DLQ.
// - Consider schema registry + required header enforcement in middleware.

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.header.Header;
import org.apache.kafka.common.header.internals.RecordHeader;

import java.nio.charset.StandardCharsets;

public final class TenantHeaders {
  public static final String TENANT_HEADER = "tenant_id";

  public static ProducerRecord<String, byte[]> withTenant(String topic, String key, byte[] value, String tenantId) {
    if (tenantId == null || tenantId.isEmpty()) throw new IllegalArgumentException("missing tenant_id");
    ProducerRecord<String, byte[]> rec = new ProducerRecord<>(topic, key, value);
    rec.headers().add(new RecordHeader(TENANT_HEADER, tenantId.getBytes(StandardCharsets.UTF_8)));
    return rec;
  }

  public static String requireTenant(ConsumerRecord<String, byte[]> rec) {
    Header h = rec.headers().lastHeader(TENANT_HEADER);
    if (h == null) throw new IllegalArgumentException("tenantless message");
    String tenantId = new String(h.value(), StandardCharsets.UTF_8);
    if (tenantId.isEmpty()) throw new IllegalArgumentException("tenantless message");
    return tenantId;
  }
}
```

### Common misconception
> "We'll just pass tenant_id as a parameter."

That fails when:
- a developer forgets to pass it,
- a queue consumer processes mixed-tenant batches,
- retries re-enqueue without headers,
- logs omit tenant context.

### Key insight
> Tenant identity is a distributed tracing problem with security consequences.

### Challenge question
How would you enforce "no tenantless work" in background workers?

---

## Section 6: Data architecture choices (row, schema, database, shard, cell)

### Scenario
You're choosing how to store tenant data in Postgres (or a similar relational DB). You also expect to move some workloads to a distributed SQL/NoSQL system later.

### Interactive exercise: matching
Match the storage pattern to the strongest benefit:

Patterns:
- (A) Row-level multi-tenancy (tenant_id column)
- (B) Schema-per-tenant
- (C) Database-per-tenant
- (D) Shard-per-tenant-group (hash-based)
- (E) Cell-based (each cell has its own DB fleet)

Benefits:
1) Simplest cross-tenant analytics
2) Easiest per-tenant backup/restore
3) Best blast-radius containment
4) Best operational scalability at 10k+ tenants
5) Lowest cost at small scale

Pause and think.

### Progressive reveal (one reasonable mapping)
- A -> 5 (cheap early)
- C -> 2 (restore a tenant by restoring a DB)
- E -> 3 (cells contain failure)
- D -> 4 (shard groups scale)
- A (or D) -> 1 (analytics easier when co-located; depends on warehouse strategy)

### [IMAGE: Diagram showing row-level vs schema-per-tenant vs db-per-tenant; plus shard ring and cells, with arrows for query routing.]

### Trade-offs (distributed systems lens)
- Row-level: shared indexes become hotspots; vacuum/compaction affects everyone; connection pool is shared; per-tenant PITR/restore is hard.
- Schema-per-tenant: migration tooling complexity; many schemas stress catalog; still shared IO; permissions sprawl.
- DB-per-tenant: operational overhead; connection storms; but strong isolation and easier residency; can tune per-tenant.
- Shard groups: need routing layer; rebalancing is hard; hot tenants can still dominate a shard.
- Cells: operationally scalable; requires placement service; cross-cell queries are expensive.

### Production insight: "moving tenants" is the real requirement
At scale, the key question is not "where do I start?" but "how do I move a tenant when they outgrow their tier?"
- Row-level -> dedicated DB requires careful migration tooling.
- Cells make it easier to move groups, but cross-cell dependencies become the constraint.

### Key insight
> Data placement is your primary lever for controlling blast radius and operational scalability.

### Challenge question
If a single tenant grows 100x, which patterns let you move that tenant with minimal downtime?

---

## Section 7: Noisy neighbors - the failure mode that keeps returning

### Scenario
Tenant A runs a heavy report at 09:00. Suddenly:
- API p99 latency spikes for everyone
- DB CPU hits 95%
- queue lag grows
- autoscaling adds pods, but DB is the bottleneck

### Interactive question (pause and think)
What's the most likely root cause?

A) Not enough pods
B) Shared bottleneck (DB, cache, queue partitions) with no per-tenant fairness
C) Bad TLS configuration
D) Too many microservices

Pause and think.

### Progressive reveal (answer)
B.

### Mental model: "shared kitchen, unlimited orders"
If all tenants share the same kitchen and there is no policy for how many dishes each can order per minute, one tenant can saturate the grill.

### Practical mitigation toolbox
- Per-tenant rate limits at edge and service
- Per-tenant concurrency limits for expensive endpoints
- Work queues per tenant or per tier (or weighted fair queues)
- DB resource governance: statement timeouts, workload management, separate replicas
- Separate OLTP from OLAP: move reports to warehouse / async materialization
- Admission control: reject/queue work before overload

### Failure scenario to rehearse: retry storms
If clients retry on 5xx/timeouts without jitter/backoff, a single hot tenant can trigger a system-wide retry storm.
- Use bounded retries, exponential backoff + jitter, and idempotency keys.
- Prefer 429 (rate limited) or 503 with Retry-After for overload.

### [CODE: Python, token-bucket rate limiter keyed by tenant_id with burst + sustained limits]

```python
# Per-tenant token-bucket rate limiting (thread-safe).
# Production notes:
# - In multi-process deployments, use a shared store (Redis) or sidecar.
# - Add eviction/TTL for inactive tenants to avoid unbounded memory.

import threading
import time

class TokenBucket:
    def __init__(self, rate_per_sec: float, burst: int):
        self.rate, self.cap = float(rate_per_sec), int(burst)
        self.tokens, self.last = float(burst), time.monotonic()
        self.lock = threading.Lock()

    def allow(self, cost: float = 1.0) -> bool:
        with self.lock:
            now = time.monotonic()
            self.tokens = min(self.cap, self.tokens + (now - self.last) * self.rate)
            self.last = now
            if self.tokens >= cost:
                self.tokens -= cost
                return True
            return False

class TenantLimiter:
    def __init__(self, default_rate=10, default_burst=20, idle_ttl_sec=3600):
        self.default_rate, self.default_burst = default_rate, default_burst
        self.idle_ttl_sec = idle_ttl_sec
        self.buckets = {}  # tenant_id -> (bucket, last_seen)
        self.lock = threading.Lock()

    def allow(self, tenant_id: str) -> bool:
        if not tenant_id:
            raise ValueError("tenant_id required")
        now = time.monotonic()
        with self.lock:
            # opportunistic cleanup
            if len(self.buckets) > 10_000:
                for t, (_, last_seen) in list(self.buckets.items()):
                    if now - last_seen > self.idle_ttl_sec:
                        self.buckets.pop(t, None)

            b, _ = self.buckets.get(tenant_id, (None, None))
            if b is None:
                b = TokenBucket(self.default_rate, self.default_burst)
            self.buckets[tenant_id] = (b, now)

        return b.allow()

# Usage example: limiter=TenantLimiter(); limiter.allow("t-123")
```

### Key insight
> Autoscaling doesn't fix contention on a shared bottleneck; it often makes it worse by increasing pressure on the bottleneck.

### Challenge question
Where would you apply backpressure first to stop a cascade: API gateway, service, queue consumer, or DB?

---

## Section 8: Cell-based architecture (the "zoned delivery" strategy)

### Scenario
You have 20 million daily active users across 8,000 tenants. "Shared everything" is too risky; "dedicated per tenant" is too expensive.

You consider cells: many small, mostly identical stacks. Tenants are assigned to a cell. Each cell has:
- its own service fleet
- its own DB shard(s)
- its own queues
- its own cache

### Interactive question (pause and think)
What problem do cells solve best?

A) Cross-tenant analytics
B) Blast radius and operational scalability
C) Eliminating the need for SRE
D) Making deployments instantaneous

Pause and think.

### Progressive reveal (answer)
B.

### [IMAGE: Cell-based architecture diagram: global control plane + many cells; tenant routing via edge; per-cell DB/queue/cache; failure contained to one cell.]

### Mental model: "delivery zones"
A delivery company splits a city into zones. If one zone has a traffic jam, other zones still deliver. Dispatch (control plane) routes orders to the right zone.

### Distributed systems implications
- Routing: you need a tenant-to-cell mapping service.
  - Network assumption: edge can reach mapping service (or cache) even when a cell is degraded.
  - Consistency: mapping should be strongly consistent for writes (placement changes), but reads can be cached with bounded staleness.
- Migration: moving a tenant between cells requires data copy + cutover.
- Control plane vs data plane: control plane remains global; data plane is per-cell.
- Cross-cell operations: expensive; avoid synchronous cross-cell calls.

### CAP note (practical)
Tenant-to-cell mapping is a coordination problem.
- Under partition between edge and mapping store, you must choose:
  - Availability: serve using cached mapping (bounded staleness) -> risk misrouting during active migrations.
  - Consistency: fail closed when mapping can't be verified -> higher error rate during partitions.
In practice: serve from cache with migration-safe routing.

### Production insight: migration-safe routing
During tenant moves, use a state machine:
- ACTIVE(cellA)
- MIGRATING(read=cellA, write=dual)
- CUTOVER(cellB)
- FINAL(cellB)

Edge routing must understand these states to avoid split-brain writes.

### Failure scenario walkthrough
Cell 12's DB has a bad index migration -> CPU pegged -> API errors for tenants in Cell 12 only.
- With cells: blast radius is limited.
- Without cells: everyone is impacted.

### Key insight
> Cells turn "one big distributed system" into "many smaller distributed systems" with a shared control plane.

### Challenge question
What must the control plane guarantee during an outage of a cell? (Think routing, retries, and tenant mapping.)

---

## Section 9: Control plane vs data plane - and why it matters for multi-tenancy

### Scenario
You add a "tenant management service" that:
- provisions tenants
- assigns them to tiers/cells
- manages config (limits, features)
- triggers migrations

It becomes critical infrastructure.

### Interactive question (pause and think)
Which is the most dangerous design mistake?

A) Control plane uses a different database than the product
B) Control plane outages prevent existing tenants from serving traffic
C) Control plane requires authentication
D) Control plane has an API

Pause and think.

### Progressive reveal (answer)
B.

### Mental model: "the restaurant manager vs the kitchen"
If the manager is absent, the kitchen can still cook for seated customers. But you can't seat new customers or change the menu.

Design principle:
- Data plane should degrade gracefully when control plane is down.
- Cache tenant config at edge/service with TTL.
- Separate "provisioning" from "runtime authorization."

### [CODE: JavaScript, example tenant config document and caching strategy with versioning/ETags]

```javascript
// Versioned tenant config caching with ETag + safe defaults.
// Production notes:
// - Keep control plane off hot path.
// - Use bounded staleness and safe defaults.
// - Consider push updates (watch) to reduce staleness.

const DEFAULTS = { tier: "bronze", limits: { rps: 10, max_concurrent_exports: 1 } };

export async function getTenantConfig(fetchFn, tenantId, cache) {
  if (!tenantId) throw new Error("tenantId required");
  const cached = cache.get(tenantId);
  const headers = cached?.etag ? { "If-None-Match": cached.etag } : {};

  try {
    const res = await fetchFn(`/tenants/${tenantId}/config`, { headers, timeoutMs: 1500 });
    if (res.status === 304 && cached) return cached.value; // bounded staleness
    if (res.status !== 200) throw new Error(`config_fetch_failed:${res.status}`);
    const etag = res.headers.etag;
    const value = { ...DEFAULTS, ...(await res.json()) };
    cache.set(tenantId, { etag, value, expiresAt: Date.now() + 60_000 });
    return value;
  } catch {
    if (cached && cached.expiresAt > Date.now()) return cached.value; // serve stale-but-bounded
    return DEFAULTS; // safe fallback keeps control plane off hot path
  }
}
```

### Key insight
> Keep the control plane off the hot path.

### Challenge question
What tenant metadata must be available on every request, and how do you serve it if the control plane is down?

---

## Section 10: Consistency, caching, and configuration propagation

### Scenario
You roll out a new per-tenant limit: max_concurrent_exports. You update the config store. Some services enforce the new limit immediately; others lag for minutes.

### Interactive question (pause and think)
Which consistency model is most appropriate for tenant config?

A) Strongly consistent everywhere, always
B) Eventually consistent with bounded staleness + safe defaults
C) No caching; always read from config DB
D) Randomized polling

Pause and think.

### Progressive reveal (answer)
B.

### Real-world analogy: menu boards
If the menu price changes, you don't want half the store charging old prices for hours. But you also don't need a distributed transaction across every screen - just bounded propagation and a rule like "when unsure, charge the safer price."

### Techniques
- Versioned config documents; clients cache by version.
- Push-based updates (watch streams) + fallback polling.
- Safe defaults: deny-by-default for risky features; allow-by-default for non-risky.
- Config rollout: canary by tenant.

### Consistency nuance
Some config changes are "safety critical" and may require stronger guarantees:
- disabling a compromised tenant token
- revoking access to a dataset
- rotating encryption keys

For those, prefer:
- short TTLs,
- push invalidation,
- and/or a strongly consistent authorization path (e.g., token introspection) while keeping the rest cached.

### Key insight
> Config propagation is a distributed systems problem; treat it like code deployment with safety rails.

### Challenge question
Which tenant config changes require strong consistency (if any), and why?

---

## Section 11: Background jobs and queues - the multi-tenant "back alley"

### Scenario
Your API layer is tenant-aware and rate-limited. But exports are processed by a shared worker pool reading from a single queue.

Tenant A triggers 1 million export jobs. Tenant B triggers 10. Tenant B's jobs wait hours.

### Interactive question (pause and think)
What's the best fix?

A) Add more workers
B) Create per-tenant queues
C) Use a single queue but implement weighted fair scheduling
D) Disable exports

Pause and think.

### Progressive reveal (answer)
Often C is best at scale: weighted fair scheduling (WFQ) or per-tenant concurrency caps.
- Per-tenant queues (B) can explode in count.
- More workers (A) can worsen DB contention.

### [IMAGE: Queue scheduling diagram showing FIFO vs per-tenant fair queue vs priority tiers.]

### [CODE: Python, weighted fair scheduler keyed by tenant tier with per-tenant concurrency limits]

```python
# Weighted fair scheduling with per-tenant concurrency caps.
# NOTE: This is a teaching example; production schedulers must handle:
# - dynamic tenant sets
# - starvation prevention
# - persistence (crash-safe)
# - visibility timeouts / leases

import threading
from collections import defaultdict, deque

class FairScheduler:
    def __init__(self, weights: dict[str, int], max_inflight_per_tenant=2):
        self.weights = weights
        self.max_inflight = max_inflight_per_tenant
        self.qs = defaultdict(deque)        # tenant_id -> deque[job]
        self.inflight = defaultdict(int)    # tenant_id -> count
        self.cv = threading.Condition()

    def submit(self, tenant_id: str, job: dict):
        if not tenant_id:
            raise ValueError("tenant_id required")
        with self.cv:
            self.qs[tenant_id].append(job)
            self.cv.notify()

    def next_job(self):
        with self.cv:
            while True:
                # Simple priority by weight; production: use deficit round robin / WFQ.
                for tenant_id, _w in sorted(self.weights.items(), key=lambda x: -x[1]):
                    if self.inflight[tenant_id] < self.max_inflight and self.qs[tenant_id]:
                        self.inflight[tenant_id] += 1
                        return tenant_id, self.qs[tenant_id].popleft()
                self.cv.wait(timeout=0.5)

    def done(self, tenant_id: str):
        with self.cv:
            self.inflight[tenant_id] = max(0, self.inflight[tenant_id] - 1)
            self.cv.notify()
```

### Failure scenarios to cover
- Poison messages: one tenant's bad payload causes infinite retries -> global backlog.
  - Use DLQ, retry budgets, and per-tenant circuit breakers.
- Downstream dependency outage: object storage down -> exports retry -> queue grows.
  - Use backoff, max retry time, and shed load per tenant.

### Key insight
> Multi-tenant fairness must exist in the async plane, not only in the request plane.

### Challenge question
What metric would you alert on to detect "tenant starvation" in background processing?

---

## Section 12: Failure scenarios you must rehearse (and how multi-tenancy changes them)

### Scenario
You run a game day. You inject faults. You realize your incident response playbooks assume a single-tenant world.

### Interactive exercise: "What fails next?"
For each scenario, predict the second-order failure.

1) One tenant causes DB CPU saturation.
2) Control plane config store is down.
3) A cell loses connectivity to object storage.
4) A tenant's encryption key in KMS is disabled.

Pause and think.

### Progressive reveal (possible answers)
1) Connection pool starvation -> timeouts across services -> retry storms -> further DB load.

2) New tenant provisioning fails; feature flags can't change. If config is on hot path, everyone fails.

3) Upload/download retries accumulate -> queue backlog grows -> workers saturate network -> cascading latency.

4) Tenant-specific operations fail (good), but if your code treats it as a global KMS outage, you may trip global circuit breakers.

### Production insight: selective degradation
Multi-tenancy requires representing "partial outage" explicitly:
- circuit breakers keyed by (tenant, dependency) not just dependency
- error budgets/SLOs per tier and per tenant (at least for top tenants)

### Key insight
> Multi-tenancy adds "selective failure": one tenant should fail while others keep working. Your system must represent that explicitly.

### Challenge question
In your error budgets and SLOs, do you measure per-tenant availability or global availability? Which should drive paging?

---

## Section 13: Observability - per-tenant metrics without blowing up cardinality

### Scenario
You add tenant_id as a Prometheus label. Suddenly Prometheus OOMs, dashboards slow, and alerts become noisy.

### Interactive question (pause and think)
Which is the best approach?

A) Label every metric with tenant_id
B) Never record tenant_id anywhere
C) Use tiered observability: per-tenant for sampled/high-value tenants + aggregated metrics for all
D) Store per-tenant metrics only in logs

Pause and think.

### Progressive reveal (answer)
C.

### Mental model: "receipts vs accounting ledger"
A restaurant keeps detailed receipts for audits and disputes, but the daily ledger is aggregated. You don't want every receipt line item in the main dashboard.

### Practical patterns
- High-cardinality in traces/logs, not in time-series metrics.
- Per-tenant dashboards generated on demand for top tenants.
- Tenant tier labels (gold/silver/bronze) for broad analysis.
- Cardinality budgets: enforce in instrumentation reviews.

### [CODE: OpenTelemetry, adding tenant_id to trace attributes; metrics use tenant_tier only]

```javascript
import { trace, SpanStatusCode } from "@opentelemetry/api";

export async function handleRequest(req, res, meter) {
  const tenantId = req.headers["x-tenant-id"];
  const tenantTier = req.headers["x-tenant-tier"] || "bronze";
  const tracer = trace.getTracer("mt-svc");
  const counter = meter.createCounter("http_requests_total");

  return tracer.startActiveSpan("http.request", async (span) => {
    try {
      if (!tenantId) throw new Error("missing tenant_id");
      span.setAttribute("tenant.id", tenantId); // high-cardinality: traces only
      span.setAttribute("tenant.tier", tenantTier);
      counter.add(1, { tenant_tier: tenantTier, route: req.url }); // low-cardinality metrics
      res.writeHead(200, { "content-type": "application/json" });
      res.end(JSON.stringify({ ok: true }));
      span.setStatus({ code: SpanStatusCode.OK });
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(err) });
      res.writeHead(400);
      res.end("bad request");
    } finally {
      span.end();
    }
  });
}
```

### Key insight
> Per-tenant observability is essential, but you must choose the right telemetry channel to avoid cardinality collapse.

### Challenge question
Where would you store per-tenant latency histograms: metrics, logs, or traces - and why?

---

## Section 14: Security boundaries - beyond "WHERE tenant_id = ?"

### Scenario
A developer writes a query without tenant filter. A bug slips through code review. It returns data across tenants.

### Interactive question (pause and think)
Which defense is strongest?

A) Developer training
B) Unit tests
C) Database-enforced row-level security (RLS) / authorization policies
D) A linter rule

Pause and think.

### Progressive reveal (answer)
C is strongest because it enforces at the lowest layer.

### [IMAGE: Defense-in-depth stack: ingress auth -> service authZ -> DB RLS -> encryption keys -> audit logs.]

### [CODE: SQL, Postgres RLS policy enforcing tenant_id = current_setting('app.tenant_id')]

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_invoices
ON invoices
USING (tenant_id = current_setting('app.tenant_id', true));

CREATE POLICY tenant_isolation_invoices_write
ON invoices
FOR INSERT
WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

### [CODE: Python, safe session setup for Postgres RLS]

```python
import os
import psycopg


def fetch_invoices(tenant_id: str):
    if not tenant_id:
        raise ValueError("tenant_id required")
    dsn = os.environ.get("DATABASE_URL")
    if not dsn:
        raise RuntimeError("DATABASE_URL not set")

    with psycopg.connect(dsn, autocommit=False) as conn:
        with conn.cursor() as cur:
            cur.execute("BEGIN")
            # LOCAL scopes it to this transaction.
            cur.execute("SELECT set_config('app.tenant_id', %s, true)", (tenant_id,))
            cur.execute("SELECT id, total_cents FROM invoices ORDER BY id DESC LIMIT 50")
            rows = cur.fetchall()
            conn.commit()
            return rows
```

### Additional layers
- Service-level authZ checks
- Signed tenant context tokens
- Separate encryption keys per tenant (envelope encryption)
- Audit logs partitioned by tenant

### Common misconception
> "RLS will solve all isolation issues."

RLS helps security isolation, but not performance isolation. Poor queries still melt the shared DB.

### Key insight
> Security isolation should be enforced at multiple layers, with at least one "hard boundary" that developers can't accidentally bypass.

### Challenge question
If you use RLS, how do you ensure background jobs set the correct tenant context before querying?

---

## Section 15: Tenant lifecycle operations - migrations, backups, restores, deletes

### Scenario
A tenant requests:
- data export (GDPR)
- partial restore (accidental deletion)
- full deletion (right to be forgotten)

At scale, these become platform features.

### Interactive question (pause and think)
Which architecture makes tenant restore easiest?

A) Row-level in shared tables
B) DB-per-tenant
C) Cell-based with shared DB
D) Any architecture is equally easy

Pause and think.

### Progressive reveal (answer)
B is usually easiest for restore. You can snapshot/restore a tenant DB.

Row-level restore in shared tables often requires:
- point-in-time recovery + selective extraction
- replaying changes
- complex tooling

### Production insight: deletion proof
"Right to be forgotten" is not just deleting rows.
- You need a deletion ledger (who/when/what),
- proof that backups expire or are scrubbed,
- and a clear policy for derived data (indexes, caches, analytics).

### [IMAGE: Lifecycle operations diagram: provision -> migrate -> backup -> restore -> delete; showing differences for row-level vs db-per-tenant.]

### Key insight
> Operational workflows (restore/delete/migrate) are first-class architecture drivers in multi-tenant systems.

### Challenge question
How long does it take you to delete a tenant completely today? What's your proof of deletion?

---

## Section 16: Placement and tiering - "who goes where?"

### Scenario
You have tiers:
- Bronze: pooled
- Silver: pooled + stricter limits
- Gold: dedicated DB
- Platinum: dedicated cell

You need an automated placement policy.

### Decision game
Which placement rule is safest?

1) Place tenants randomly to spread load.
2) Place tenants by geography only.
3) Place tenants by predicted load and isolate "spiky" tenants.
4) Place all big tenants together so you can monitor them.

Pause and think.

### Progressive reveal (answer)
3.

Random placement spreads average load but doesn't manage correlated spikes. Putting all big tenants together creates a "hot cell."

### [CODE: JavaScript, tenant placement algorithm using constraints + scoring]

```javascript
export function placeTenant(tenant, cells) {
  if (!tenant?.id) throw new Error("tenant.id required");
  const eligible = cells.filter((c) =>
    c.region === tenant.region &&
    (!tenant.euOnly || c.euOnly === true) &&
    c.freeCpu > 0.2 &&
    c.freeDbConn > 50
  );
  if (eligible.length === 0) throw new Error("no_eligible_cells");

  // Score: prefer headroom and avoid concentrating spiky tenants.
  const scored = eligible
    .map((c) => {
      const headroom = 0.6 * c.freeCpu + 0.4 * (c.freeDbConn / c.dbConnCap);
      const riskPenalty = (c.spikyTenants / Math.max(1, c.tenantCount)) * 0.5;
      const score = headroom - riskPenalty - c.utilization * 0.2;
      return { cellId: c.id, score };
    })
    .sort((a, b) => b.score - a.score);

  return scored[0].cellId; // production: tie-breaker + audit log + placement idempotency
}
```

### Key insight
> Tenant placement is like bin packing with failure domains. The objective isn't just utilization - it's risk management.

### Challenge question
What signals would you use to classify a tenant as "spiky"?

---

## Section 17: Cross-tenant features (analytics, search) without breaking isolation

### Scenario
Product wants:
- global search across all tenants (for internal support)
- benchmarking ("how do you compare to peers?")
- fraud detection across tenants

### Interactive question (pause and think)
Which approach best preserves isolation?

A) Query the production OLTP DB across tenants
B) Build a separate analytics pipeline with strict access controls and anonymization
C) Copy all data into a shared cache for fast search
D) Disable cross-tenant features

Pause and think.

### Progressive reveal (answer)
B.

### Real-world analogy: kitchen vs tasting lab
You don't run experiments on the main kitchen line during dinner rush. You use a lab with controlled access.

### Production insight
For internal support search, consider:
- a separate index with support-only access,
- strict audit logging,
- and tenant-scoped query enforcement.

### Key insight
> Cross-tenant features should be built on separate data planes (warehouse, search index) with explicit governance.

### Challenge question
How would you prove to customers that cross-tenant benchmarking can't reveal their raw data?

---

## Section 18: Cost attribution and "who pays for the blast radius?"

### Scenario
Finance asks: "Why did infra costs spike 40%?" SRE says: "One tenant ran massive workloads." Sales says: "Don't throttle them - they pay a lot."

### Interactive question (pause and think)
What do you need to make this conversation rational?

A) Better graphs
B) Per-tenant cost attribution (showback/chargeback)
C) A bigger cluster
D) More meetings

Pause and think.

### Progressive reveal (answer)
B.

### [IMAGE: Cost attribution flow: request tagged with tenant -> metrics/logs -> cost model -> per-tenant bill/showback.]

### Production insight
Cost attribution requires consistent tagging across:
- requests (tenant context)
- async jobs
- storage objects
- DB usage (queries, IOPS)

If you can't attribute DB cost, you'll under-incentivize the biggest bottleneck.

### [CODE: Python, example aggregation of usage events by tenant]

```python
import json
from multiprocessing import Pool
from collections import defaultdict


def _accumulate(lines):
    out = defaultdict(lambda: {"requests": 0, "bytes": 0})
    for line in lines:
        try:
            e = json.loads(line)
            t = e["tenant_id"]
            out[t]["requests"] += int(e.get("requests", 0))
            out[t]["bytes"] += int(e.get("bytes", 0))
        except (KeyError, ValueError, json.JSONDecodeError):
            continue  # production: count bad events + ship to quarantine
    return out


def aggregate_usage(log_lines, workers=4):
    chunks = [log_lines[i::workers] for i in range(workers)]
    totals = defaultdict(lambda: {"requests": 0, "bytes": 0})
    with Pool(processes=workers) as p:
        for partial in p.map(_accumulate, chunks):
            for t, v in partial.items():
                totals[t]["requests"] += v["requests"]
                totals[t]["bytes"] += v["bytes"]
    return totals
```

### Key insight
> Multi-tenancy without per-tenant metering turns performance incidents into political incidents.

### Challenge question
What's your unit of billing: requests, seats, GB stored, CPU-seconds, or something else? How does it map to real resource contention?

---

## Section 19: Migration strategies - evolving multi-tenancy without downtime

### Scenario
You started with row-level multi-tenancy. Now you need to move a top tenant to a dedicated DB or a different cell.

### Interactive question (pause and think)
Which migration plan is safest?

A) Stop the world, dump/restore, restart
B) Dual-write + backfill + verify + cutover with idempotent consumers
C) Copy data once and hope nothing changes
D) Ask the tenant to stop using the product

Pause and think.

### Progressive reveal (answer)
B.

### [IMAGE: Migration timeline diagram: backfill -> dual write -> consistency check -> cutover -> cleanup.]

### Production insight: correctness model
During dual-write you are effectively running a distributed system with two sources of truth.
- Define which store is authoritative for reads.
- Ensure writes are idempotent.
- Use reconciliation jobs and explicit cutover.

### [CODE: JavaScript, idempotent event consumer with tenant-aware dedupe keys]

```javascript
import crypto from "node:crypto";

export async function consumeEvent(event, store, applyFn) {
  const tenantId = event?.tenant_id;
  const eventId = event?.event_id;
  if (!tenantId || !eventId) throw new Error("invalid_event_missing_ids");

  // Dedupe key must include tenant to prevent cross-tenant collisions.
  const key = crypto.createHash("sha256").update(`${tenantId}:${eventId}`).digest("hex");

  const inserted = await store.putIfAbsent(key, { seenAt: Date.now() }, 7 * 24 * 3600);
  if (!inserted) return { ok: true, deduped: true };

  await applyFn(event); // must be deterministic and safe to retry
  return { ok: true, deduped: false };
}
```

### Key insight
> Tenant migrations are distributed data migrations. Treat them like you treat region migrations: staged, observable, reversible.

### Challenge question
What invariant would you monitor during migration to detect divergence early?

---

## Section 20: Testing multi-tenancy - proving isolation under chaos

### Scenario
You have unit tests and integration tests. But you've never proven that Tenant A cannot starve Tenant B under load.

### Interactive exercise
Design a test in your head:
- Two tenants: A (heavy) and B (light)
- Run load: A sends 1000 rps, B sends 10 rps
- Success criteria: B's p99 latency stays under X, error rate under Y

Pause and think: what components must be included to make this test meaningful?

### Progressive reveal (answer)
A meaningful test must include:
- real rate limiting / admission control
- real queues and workers (if async)
- real DB contention (or a faithful emulator)
- realistic retry behavior

### [CODE: k6, load test script generating traffic for two tenants and asserting SLOs]

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  scenarios: {
    heavy: { executor: "constant-arrival-rate", rate: 1000, timeUnit: "1s", duration: "30s", preAllocatedVUs: 50 },
    light: { executor: "constant-arrival-rate", rate: 10, timeUnit: "1s", duration: "30s", preAllocatedVUs: 5 },
  },
  thresholds: {
    "http_req_failed{tenant:light}": ["rate<0.01"],
    "http_req_duration{tenant:light}": ["p(99)<250"],
  },
};

export default function () {
  const tenant = __SCENARIO__ === "light" ? "light" : "heavy";
  const res = http.get("https://api.example.com/v1/ping", {
    headers: { "x-tenant-id": tenant },
    tags: { tenant },
  });
  check(res, { "status 2xx": (r) => r.status >= 200 && r.status < 300 });
  sleep(0.01);
}
```

### Key insight
> Multi-tenant correctness includes fairness and isolation properties, not just functional correctness.

### Challenge question
What is your "tenant isolation SLO," and how would you continuously validate it?

---

## Section 21: Final synthesis challenge - design a multi-tenant platform under constraints

### Scenario (capstone)
You're building "AcmeDocs," a document SaaS.
- 12,000 tenants
- 95% small tenants, 4% mid-market, 1% enterprise
- Workloads: API CRUD, full-text search, batch exports, analytics dashboards
- Compliance: some tenants require EU-only data residency
- Availability: 99.9% overall, but enterprise wants 99.95%

### Your task
Design a multi-tenant architecture and explain:
1) Tenant placement (pooled/cell/dedicated)
2) Data model (row/schema/db/cell) and migration path
3) Isolation controls (rate limits, queues, DB governance)
4) Failure containment (blast radius, circuit breakers, selective degradation)
5) Observability and cost attribution (without cardinality explosion)

### Interactive prompts (pause and think)
- Where do you put full-text search: per-cell, shared global, or per-tenant?
- How do you handle a tenant that suddenly becomes 50x larger?
- What breaks when the control plane is down?

Pause and think. Sketch it.

### Progressive reveal (one reference design)
- Architecture: cell-based pooled tier for most tenants; dedicated cell or dedicated DB for enterprise.
- Routing: edge routes by tenant-to-cell mapping cached with TTL.
- Data:
  - pooled cells use shard groups (tenant groups per DB cluster)
  - enterprise uses DB-per-tenant or dedicated shard
  - migrations via dual-write/backfill
- Isolation:
  - edge rate limits per tenant and per tier
  - service-level concurrency limits for heavy endpoints
  - background WFQ scheduler with per-tenant caps
  - separate OLTP from OLAP (warehouse)
- Failure containment:
  - cell outages isolate impact
  - selective circuit breakers per tenant/cell
  - control plane off hot path
- Observability:
  - tenant_id in traces/logs; tier labels in metrics
  - per-tenant dashboards for top tenants
- Cost:
  - per-tenant metering events for storage, requests, exports

### Production trade-off: search placement
- Per-cell search: best isolation and blast radius; harder global support search.
- Shared global search: operationally simpler; becomes a shared bottleneck and a security boundary.
- Per-tenant search: strongest isolation; expensive and operationally heavy.

### Key insight box
> Scaling multi-tenancy is scaling governance: placement, identity propagation, fairness, and lifecycle operations - implemented as a control plane over many data planes.

### Final challenge questions
1) What's the smallest change you can make next week to reduce blast radius by 10x?
2) What's the hardest part of your design to operate at 10,000 tenants?
3) If you had to pick only one: better performance isolation or easier tenant migrations - what do you choose and why?

---

## Appendix: Quick reference checklists

### Tenant isolation checklist
- [ ] Tenant identity authenticated at ingress
- [ ] Tenant identity propagated (RPC, events, jobs)
- [ ] Hard boundary for data access (RLS/policies or separate DB)
- [ ] Per-tenant rate limits + concurrency limits
- [ ] Fair scheduling in queues/workers
- [ ] Per-tenant cost attribution
- [ ] Per-tenant incident visibility (without cardinality blowup)

### "Game day" failure drills
- [ ] Hot tenant load spike
- [ ] Cell DB overload
- [ ] Control plane outage
- [ ] KMS key disabled for one tenant
- [ ] Partial region outage (for residency)
- [ ] Mapping store partition (edge can't reach control plane)
- [ ] Retry storm simulation (clients + internal)

### Design review questions
- Where is tenant mapping stored and cached?
- What is the blast radius of a bad migration?
- How do you move a tenant between tiers/cells?
- How do you do tenant delete/restore?
- What is your explicit consistency model for tenant mapping and config?

---

End of reviewed article.
