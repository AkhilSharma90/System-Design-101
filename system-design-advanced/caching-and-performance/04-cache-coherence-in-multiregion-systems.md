---
slug: cache-coherence-in-multi-region-system
title: Cache Coherence in Multi-Region System
readTime: 20 min
orderIndex: 4
premium: false
---




# Cache Coherence in Multi-Region Systems
> Audience: engineers building/operating distributed systems across regions.
>
> Goal: build a practical mental model for keeping caches “coherent enough” when your data and users span continents.

---

##  CHALLENGE: “Why did the user see the old price… in only one region?”

You run a global e-commerce platform:

- Region A (us-east) serves North America.
- Region B (eu-west) serves Europe.
- Region C (ap-southeast) serves Asia.

You store product data in a multi-region database. To keep latency low, each region has a local cache (Redis/Memcached/in-process) in front of the database.

A user updates a product price from $19.99 -> $24.99 using an admin tool in us-east.

Minutes later:

- Users in us-east see $24.99.
- Users in eu-west still see $19.99.
- Users in ap-southeast see a mix depending on which edge node they hit.

### Pause and think
Which statement best explains what happened?

A) The database replication is broken.

B) The caches are coherent within a region, but not coherent across regions.

C) The cache TTL is too small.

D) TCP packet loss caused stale reads.

Progressive reveal -> Answer:

B is the likely culprit.

- Your database may be correct and replicated.
- Your caches may be independently serving stale entries because cache invalidation/propagation across regions is slow, lossy, or not coordinated.

Key insight:

In multi-region systems, “cache coherence” is less about perfect global agreement and more about controlling staleness, handling failures, and choosing trade-offs intentionally.

---

##  What “Cache Coherence” Means Outside a CPU

### Scenario
In CPUs, cache coherence means cores don’t see contradictory values for the same memory location (MESI, etc.). In distributed systems, the word gets overloaded.

### Interactive question (pause and think)
When someone says “our cache is coherent,” what do they most likely mean?

1) Every cache node always has the latest value.

2) Reads never return stale data.

3) There is a defined bound or mechanism for staleness (time/version/invalidation).

4) The cache has a high hit rate.

Progressive reveal -> Answer:

3 is the practical distributed meaning.

### Mental model
Think of multi-region cache coherence as a promise about staleness:

- Time-bounded: “At worst, data is 30 seconds old.”
- Version-bounded: “You won’t see versions older than v-2.”
- Session-bounded: “After you write, you read your own write.”
- Event-bounded: “After invalidation event is delivered, you won’t see old values.”

### Real-world analogy
Imagine a coffee chain with stores in three cities. The menu price changes.

- HQ updates the official price list (database).
- Each store has a printed menu (cache).
- Coherence is about how quickly and reliably stores replace old menus.

If some stores keep old menus, customers see old prices.

Key insight:

In distributed systems, coherence is about propagation and agreement under latency, partitions, and failures.

### Challenge question
If you can’t guarantee instantaneous coherence globally, what other guarantees can you offer that are still valuable to users?

---

##  CHALLENGE: The Multi-Region Reality—Latency, Partitions, and “Physics”

### Scenario
You want eu-west to see the new price immediately after it’s updated in us-east.

### Pause and think
What’s the minimum round-trip time (RTT) between us-east and eu-west in the real world?

- Often 60–100ms+ depending on network paths.

Now add:

- replication lag,
- cache invalidation delivery,
- queueing,
- retries,
- failovers.

### Explanation
Multi-region coherence is constrained by:

- Speed of light (latency floor)
- Network variance (tail latency)
- Partial failures (packet loss, partitions)
- Independent failure domains (a region can be down)

### Common Misconception
“Cache coherence is just an engineering problem; we can make it perfect with enough effort.”

Reality: you can improve it, but you can’t eliminate the fundamental trade-offs:

- Stronger coherence usually means higher write latency and/or lower availability.
- Weaker coherence improves availability/latency but increases staleness.

Key insight:

Coherence is a spectrum. Your job is to choose the right point based on product requirements and failure tolerance.

### Challenge question
Would you rather:

- pay 200ms extra on every write to keep caches globally coherent, or
- allow up to 5 seconds of staleness in remote regions?

What user experience does each choice create?

---

## INVESTIGATE: What Exactly Is “Coherence” in a Multi-Region Cache?

### Scenario
You have a key: `product:1234:price`.

- us-east cache has version 17 ($24.99)
- eu-west cache still has version 16 ($19.99)

### Think about it
Is this a “bug”? Or an expected outcome?

Progressive reveal -> Answer:

It depends on your consistency contract.

If your contract is:

- “eventual consistency with TTL 60s” -> expected.
- “read-your-writes for admin users globally” -> bug.
- “monotonic reads per session” -> maybe bug depending on session routing.

### Mental model: Coherence dimensions
In distributed caches, coherence often decomposes into:

1. Visibility latency: how long until an update is visible everywhere?
2. Ordering: do updates arrive in the same order in every region?
3. Atomicity: do readers see partial updates (multi-key writes)?
4. Scope: per-key vs per-transaction vs per-object graph.
5. Client guarantees: read-your-writes, monotonic reads, causal consistency.

### Matching exercise
Match the guarantee to the symptom:

| Guarantee | Symptom if missing |
|---|---|
| Read-your-writes | User updates profile, then immediately sees old profile |
| Monotonic reads | User sees new value, then later sees older value |
| Causal consistency | User sees comment reply before seeing original comment |
| Strong consistency | Two regions disagree on current value |

Pause and think, then check.

Answers:
- Read-your-writes -> old profile right after write
- Monotonic reads -> newer then older
- Causal consistency -> reply before original
- Strong consistency -> regions disagree

Key insight:

“Cache coherence” is not one thing. It’s a bundle of properties about when and in what order updates become visible.

### Challenge question
Which of these properties matters most for:

- product prices?
- shopping cart contents?
- bank balances?

---

##  PIPELINE: The Basic Patterns—How Multi-Region Caches Stay (Mostly) Coherent

We’ll start with the common patterns, then stress them with failures.

### Pattern 1: TTL-based caching (time-based coherence)

Scenario: each region caches `product:1234` for 60 seconds.

Pause and think: what’s the maximum staleness a user might see?

Progressive reveal -> Answer:

Up to TTL (plus propagation delays). If TTL is 60s, worst-case staleness is often ~60–70s.

**Clarification (production):** TTL bounds *how long a particular cached entry can live*, not necessarily end-to-end staleness. If refills read from a lagging replica, you can keep re-caching old data beyond TTL.

Analogy: a restaurant prints a daily menu each morning (TTL=24h). If the chef changes a dish at noon, the printed menus won’t reflect it until tomorrow.

Key insight: TTL caching is simple and robust, but coherence is probabilistic and bounded only by TTL *if* refills are fresh.

Challenge question: If you cut TTL from 60s to 5s, what happens to hit rate, database load, and staleness?

---

### Pattern 2: Write-through cache (cache updated on write path)

Scenario: when you update price, you also update cache entries.

Pause and think: does write-through solve multi-region coherence?

Progressive reveal -> Answer:

Not by itself.

- It ensures the local region’s cache is updated.
- Remote regions still need the update (via replication, invalidation, or shared cache).

Analogy: HQ updates the menu and immediately updates the menu in the HQ city. Other cities still need delivery.

Key insight: write-through helps correctness locally, but cross-region coherence requires distribution of cache updates/invalidation.

**Failure mode to call out:** if the cache update succeeds but the DB write fails, you’ve created a “dirty cache” (cache ahead of truth). If DB succeeds but cache update fails, you get a local stale cache.

**Production guidance:** treat cache writes as best-effort unless you can tolerate write unavailability. If you must keep cache and DB in sync, you’re building a transactional system—plan for retries, idempotency, and compensating actions.

Challenge question: what happens if the cache update succeeds but the database write fails (or vice versa)?

---

### Pattern 3: Cache-aside + explicit invalidation

Scenario:
- On write: (1) write database (2) publish invalidation event (3) each region deletes cache key
- On read: if cache miss -> read DB -> populate cache

Decision game: which ordering is safer?

A) Invalidate cache -> write DB

B) Write DB -> invalidate cache

Pause and think.

Progressive reveal -> Answer:

Usually B.

- If you invalidate first and then fail the DB write, you may cause unnecessary cache churn.
- If you write DB first and then invalidate, you risk a short window where stale cache still exists, but the DB is correct.

However, B is not automatically correct—you must handle the race where a reader repopulates stale data between write and invalidation.

**Critical race (often missed):** a reader can miss the cache, read a *stale replica*, and repopulate stale data even after the write committed elsewhere.

Key insight: invalidation is powerful but subtle: you must reason about races and delivery guarantees.

Challenge question: how would you prevent a reader from repopulating stale data right after a write?

---

### Pattern 4: Shared global cache (single logical cache across regions)

Scenario: all regions read/write the same cache cluster (or a globally replicated cache).

Pause and think: what do you gain and lose?

Progressive reveal -> Answer:

- Gain: fewer coherence problems (single source of cached truth).
- Lose: cross-region latency on cache hits, higher blast radius, harder availability.

Analogy: instead of each city having printed menus, everyone calls HQ for the current menu.

Key insight: a global cache can improve coherence but often defeats the purpose of regional caching (low latency, fault isolation).

**CAP note:** a “global cache” is still a distributed system. Under partitions you choose between serving stale/partial data (AP-ish) or failing/adding latency (CP-ish).

Challenge question: would you use a global cache for feature flags, authentication tokens, product catalog? Why?

---

##   The Race Conditions That Break Your Intuition

Scenario: “Write DB then invalidate cache” still serves stale.

Timeline:

1. Cache in eu-west has value V0.
2. Writer in us-east writes DB -> V1.
3. Before invalidation reaches eu-west, a reader in eu-west cache-misses (or key expires) and reads DB.

But what does it read?

- If DB replication lag means eu-west DB replica still has V0, the reader repopulates cache with V0.
- Even if DB is global, the reader may hit a stale read path (e.g., follower reads, read-only transactions, bounded staleness).

Pause and think: which component is the real source of staleness?

A) Cache only

B) Database replication/read consistency

C) Invalidation messaging

D) All of the above

Progressive reveal -> Answer:

D.

Mental model: coherence is a pipeline:

Write -> DB commit -> replication visibility -> invalidation delivery -> cache miss/refill behavior

Any stage can delay visibility.

[IMAGE: A pipeline diagram across regions showing stages: write in region A -> DB primary -> replication to region B/C -> invalidation bus -> regional caches; annotate where delay/partition can occur and how stale values re-enter via refill.]

Key insight: you don’t have “a cache coherence problem.” You have an end-to-end propagation problem.

Challenge question: where would you instrument to measure propagation delay end-to-end?

---

##   Common Misconceptions (and Why They Bite)

Misconception 1: “If we invalidate, we’re coherent.”

Reality: invalidation is only as good as its delivery and your ability to prevent stale refill.

- Lost invalidation message -> stale until TTL (or longer if stale refills happen).
- Delayed invalidation -> stale until delivery.
- Reordered invalidations -> can delete a newer value unless you use versions.

Misconception 2: “TTL guarantees staleness <= TTL.”

Reality: TTL bounds how long a given cached entry lives, not how long stale values can persist system-wide.

Stale can persist longer when:

- stale values are refilled from stale replicas,
- clocks are skewed (absolute expiration),
- background refresh keeps extending TTL (“refresh-ahead”),
- negative caching keeps “not found” around after creation.

Misconception 3: “We can just use Redis replication across regions.”

Reality: cross-region replication is subject to lag, failover split-brain, divergent writes, and operational complexity. Also, many Redis replication modes are asynchronous; you can observe stale reads after acknowledged writes.

Misconception 4: “Strong DB consistency means cache is consistent.”

Reality: the cache is an independent system with its own semantics.

Key insight: a coherent cache requires explicit design—it doesn’t inherit the database’s consistency.

Challenge question: which misconception is most likely to show up during a region failover?

---

##   Coherence Strategies by Consistency Goal

Let’s map common product requirements to cache strategies.

Comparison table:

| Requirement | Typical guarantee | Cache strategy | Trade-offs |
|---|---|---|---|
| Product catalog | Eventual, bounded by TTL | TTL + async invalidation | Stale ok; cheap |
| Shopping cart | Read-your-writes per user | Sticky sessions + write-through + per-user cache | Failover complexity |
| Inventory | Stronger (avoid oversell) | Avoid caching or use lease/version checks | Higher latency/load |
| Feature flags | Fast propagation | Push-based updates, versioned config | Complexity, fanout |
| Auth tokens | Correctness critical | Short TTL, introspection fallback | Latency on misses |

Decision game: you need read-your-writes for user profile updates globally.

Pick a design:

A) TTL 5 minutes

B) Invalidate all regions via pub/sub

C) Route the user to the write region for a while (session affinity)

D) Store per-user “minimum version” in a strongly consistent system

Pause and think.

Progressive reveal -> Answer:

Often C or D, sometimes combined with B.

- B alone can be slow/unreliable under partitions.
- C gives a strong user-centric guarantee without global coordination.
- D enforces a version floor but adds complexity.

Key insight: many real systems aim for client-centric consistency rather than global strong consistency.

Challenge question: if you use session affinity (C), what happens when the region fails?

---

##  Versioning—Your Best Friend Against Stale Refills

Scenario: you want to prevent stale values from being reintroduced into cache after an update.

Idea: store a version (or timestamp, or LSN) alongside the value.

- DB row has `version` incremented on each update.
- Cache stores `{value, version}`.
- Invalidation carries `version`.

Pause and think: if eu-west cache has version 16 and receives invalidation for version 17, what should it do?

A) Delete key unconditionally

B) Delete only if cached version <= 17

C) Delete only if cached version == 17

Progressive reveal -> Answer:

B.

- If cache already has version 18 (due to reordering), you don’t want to delete it.

Analogy: a store receives a “menu update v17” notice. If it already has menu v18, it ignores v17.

Key insight: monotonic versions protect you from message reordering and stale refills.

```python
# Implementing versioned cache entries with CAS update + conditional invalidation
from __future__ import annotations
import threading
from dataclasses import dataclass

@dataclass(frozen=True)
class Entry:
    value: object
    version: int

class VersionedCache:
    def __init__(self):
        self._lock = threading.Lock()
        self._data: dict[str, Entry] = {}

    def cas_set(self, key: str, value: object, version: int) -> bool:
        """Set only if version is newer (monotonic). Safe under reordering."""
        if version < 0:
            raise ValueError("version must be non-negative")
        with self._lock:
            cur = self._data.get(key)
            if cur and cur.version > version:
                return False  # reject stale write
            self._data[key] = Entry(value=value, version=version)
            return True

    def invalidate_upto(self, key: str, version: int) -> bool:
        """Delete only if cached version <= invalidation version."""
        if version < 0:
            raise ValueError("version must be non-negative")
        with self._lock:
            cur = self._data.get(key)
            if not cur or cur.version > version:
                return False
            del self._data[key]
            return True

# Usage example: cache.invalidate_upto("product:1234:price", 17)
```

**Production note:** in real caches you’ll want atomic compare-and-set primitives (e.g., Redis `WATCH`/`MULTI`, Lua scripts, Memcached CAS tokens) rather than a process-local lock.

Challenge question: where do versions come from (DB commit LSN, application counter, hybrid logical clock)? What breaks if versions are not monotonic?

---

##  Leases—Time-Bounded Exclusivity for Cached Data

Scenario: you want a cache entry to be “authoritative” for a short time, and you want to avoid thundering herds and stale overwrites.

Approach: cache leases

- A cache node grants a lease to a client to populate/update a key.
- While lease is held, others either wait or serve old value.
- Lease expires after a short time.

Pause and think: what failure does a lease primarily address?

A) Lost invalidations

B) Concurrent refill causing inconsistent overwrites

C) DB replication lag

D) Clock skew

Progressive reveal -> Answer:

B.

Analogy: only one delivery driver is assigned to pick up a package; others don’t show up and fight over it.

Key insight: leases help coordinate who is allowed to refresh a key, reducing races and stampedes.

**Multi-region reality:** global leases require cross-region coordination (latency + availability hit). Most systems use *regional* leases and accept cross-region staleness, or use ownership (single-writer per key) to avoid global contention.

Challenge question: in multi-region systems, can a lease be global without adding latency? If not, what’s the compromise?

---

##  Decision Game—Invalidate vs Update vs Bypass

Scenario: a write occurs: price changes.

You must choose how caches react.

1) Invalidate key everywhere.
2) Update key everywhere with new value.
3) Bypass cache for some period (serve from DB).

Pause and think: which is best for each case?

| Data type | Best reaction (1/2/3) |
|---|---|
| Hot key with huge read volume | ? |
| Rarely read key | ? |
| Highly sensitive correctness (balance) | ? |

Progressive reveal -> Suggested answers:

- Hot key: 2 (update) can prevent stampedes after invalidation.
- Rarely read: 1 (invalidate) is simplest.
- Balance: 3 (bypass) or avoid caching; or cache with strict validation.

Key insight: invalidation is not always the cheapest. Sometimes pushing updates is better—if you can do it safely.

**Partition risk (important):** pushing updates (2) during partitions can create divergent cache values across regions. If you do push updates, you still need versioning and a conflict rule.

Challenge question: what’s the risk of pushing updates (2) across regions during partitions?

---

##   Failure Scenarios (The Real Curriculum)

### Failure 1: Invalidation bus partition

Scenario: us-east publishes invalidations to Kafka/PubSub. eu-west consumer is partitioned from the bus for 10 minutes.

Pause and think: what happens to eu-west cache?

- It keeps serving stale until TTL.
- If TTL is long, staleness is long.

Mitigations:

- Shorter TTL for critical keys.
- Catch-up mechanism: on reconnect, replay invalidations by offset.
- Version floor: store latest version in a durable store and require cache version >= floor.

**Operational insight:** consumer lag is not just a Kafka metric; it’s a *correctness* metric for caches. Alert on lag by keyspace criticality.

Key insight: messaging reliability determines coherence reliability.

Challenge question: if invalidations are replayed, how do you handle out-of-order delivery relative to local cache updates?

---

### Failure 2: Region failover + split brain

Scenario: active-active writes.

- us-east and eu-west both accept writes.
- Network partition occurs.
- Both regions update `product:1234:price`.

Pause and think: which outcome is most dangerous?

A) Both caches show different values temporarily

B) Writes diverge and later conflict resolution picks one

C) Conflict resolution picks different winners in different services

Progressive reveal -> Answer:

C is catastrophic: inconsistent resolution across services causes long-lived incoherence.

Explanation: in active-active, coherence depends on a consistent conflict resolution strategy:

- last-write-wins (LWW) with synchronized clocks (dangerous)
- version vectors
- CRDTs (when data type permits)
- single-writer per key (sharding ownership)

Key insight: cache coherence cannot exceed the coherence of your write model.

Challenge question: would you allow active-active writes for prices? If yes, what conflict rule is acceptable?

---

### Failure 3: Stale reads from DB replicas

Scenario: eu-west reads from a local replica that is 30 seconds behind.

Even if cache invalidation works perfectly, refills can reintroduce stale data.

Mitigations:
- Read-your-writes routing to primary or quorum.
- Version checks: if replica version < required, retry against primary.
- Bypass cache for a window after write.

[IMAGE: Sequence diagram showing stale refill: cache miss -> replica stale read -> cache set stale -> invalidation arrives late; and an improved flow with version check/retry.]

Key insight: coherence requires the source of truth used for refills to be sufficiently fresh.

Challenge question: how do you detect replica staleness automatically?

---

### Failure 4: Clock skew breaks TTL and LWW

Scenario: you use timestamps for cache expiration, last-write-wins conflict resolution, invalidation ordering. Clock skew between regions is 500ms–5s.

Pause and think: which system property is most at risk?

A) Only performance

B) Only correctness

C) Both correctness and performance

Progressive reveal -> Answer:

C.

Mitigations:
- Avoid wall-clock for ordering; use logical versions/LSNs.
- Use HLC (Hybrid Logical Clocks) if you need causality-ish ordering.
- Keep TTL semantics relative, not absolute, where possible.

Key insight: time is a liar. Versions are better.

Challenge question: where must you use wall-clock time anyway (e.g., user-visible expiration)? How do you make it safe?

---

##   Client-Centric Coherence—Make the User Happy First

Scenario: a user updates their shipping address. They expect to see the update immediately—for them—even if the rest of the world catches up later.

Approach: session guarantees

- Read-your-writes: after a successful write, subsequent reads by same user reflect it.
- Monotonic reads: user never goes backward.
- Writes-follow-reads: user’s writes are ordered after what they read.

How to implement (typical techniques):

- Sticky sessions: route user to same region for a period.
- Session tokens with version: include last-seen version in client token; server ensures reads are at least that fresh.
- Per-user cache partition: store user-specific data in a cache that is updated synchronously.

Decision game: which technique is most robust during regional outages?

A) Sticky sessions

B) Session token with version + fallback to primary

C) Pure TTL

Progressive reveal -> Answer:

B.

- Sticky sessions break when region fails.
- Version tokens allow rerouting while preserving freshness requirements.

Key insight: global coherence is expensive. User-centric coherence often delivers most value at a fraction of the cost.

Challenge question: what’s the operational cost of version tokens (token size, invalidation on logout, privacy)?

---

##   Invalidation Delivery—At-Most-Once, At-Least-Once, Exactly-Once (and Why You Still Lose)

Scenario: you publish invalidations: `{key, version}`.

Interactive quiz: match delivery semantics to failure mode:

| Delivery | What can happen? |
|---|---|
| At-most-once | ? |
| At-least-once | ? |
| Exactly-once | ? |

Pause and think.

Answers:

- At-most-once -> lost invalidation -> stale persists until TTL (or longer via stale refill).
- At-least-once -> duplicates -> harmless if idempotent; harmful if not.
- Exactly-once -> hard/expensive; still can’t prevent reordering without additional metadata.

Mental model:

For invalidations, you want:

- Idempotence (duplicates safe)
- Monotonicity (ordering safe via versions)
- Catch-up (replay after downtime)

Exactly-once is often a distraction.

Key insight: prefer at-least-once + idempotent handlers + version checks.

Challenge question: if your invalidation stream is partitioned by key, what does that buy you? What does it not buy you?

---

##  Write Propagation Architectures

Architecture A: Central event bus + regional consumers

- Writers publish to a global bus.
- Each region consumes and invalidates/updates.

Pros:
- Simple mental model.
- Replayable.

Cons:
- Bus becomes critical dependency.
- Cross-region bandwidth and latency.

Architecture B: Multi-region gossip

- Regions exchange invalidations peer-to-peer.

Pros:
- No single central bus.

Cons:
- Harder to reason about convergence and ordering.

Architecture C: Database change data capture (CDC)

- Derive invalidations from DB log/stream.

Pros:
- Ties cache coherence to actual committed writes.

Cons:
- CDC lag; operational complexity.

[IMAGE: Comparison diagram of A/B/C showing flow of writes, invalidations, and where ordering/version metadata comes from.]

Key insight: CDC-based invalidation reduces “phantom invalidations” (invalidate without commit) and aligns cache state with DB truth.

**Production insight:** CDC is great for *eventual* coherence. For read-your-writes, you still need a fast path (session routing, version tokens, or synchronous read from primary) because CDC lag is non-zero.

Challenge question: what happens if CDC is delayed but clients demand read-your-writes?

---

##  Multi-Key and Transactional Coherence (Where Dreams Go to Die)

Scenario: checkout updates:

- `inventory:sku123` decremented
- `order:987` created
- `user:alice:cart` cleared

You cache these keys.

Pause and think: if invalidations happen per key, what can a reader observe?

A) Always a consistent transaction snapshot

B) A mix of old and new across keys

C) Only stale, never new

Progressive reveal -> Answer:

B.

Explanation: caches are typically per-key. Transactions are multi-key.

If you need transactional consistency at read time, you have options:

- avoid caching transactional reads,
- cache a materialized view of the transaction result,
- use snapshot/version reads from DB and include snapshot version in cache keys,
- use read repair with validation.

Common Misconception: “We can make the cache transactional by invalidating all touched keys.”

Reality: you can reduce inconsistency windows, but you can’t guarantee atomic visibility without:

- atomic multi-key cache operations (rare at scale), or
- gating reads on a transaction version.

Key insight: if you need transactional reads, the cache must be designed around snapshots, not keys.

Challenge question: which is cheaper—caching a transaction snapshot per user, or trying to make per-key caches transactional?

---

##  Practical Techniques That Actually Work

Technique 1: “Soft TTL” + background refresh (stale-while-revalidate)

- Serve cached value even after TTL for a short grace period.
- Trigger async refresh.

Pros: smooths latency spikes.

Cons: increases staleness unless combined with versions/invalidation.

Technique 2: Two-level caches (L1 in-process, L2 Redis)

- L1 per instance (fast)
- L2 regional shared (still fast)

Coherence issues multiply:

- invalidation must hit both layers
- L1 can reintroduce staleness even if L2 is fresh

**Production tip:** put very short TTLs on L1 (seconds) and rely on L2 for most hits; otherwise L1 becomes a “staleness amplifier.”

Technique 3: Negative caching

- Cache “not found” results.

Risk: after creation, remote regions may still serve “not found.”

Mitigation:
- very short TTL for negative entries,
- versioned existence markers.

Technique 4: Probabilistic early expiration

- Randomize TTL to reduce stampedes.

Key insight: performance techniques often worsen coherence unless paired with versioning and good invalidation.

Challenge question: which of these techniques would you avoid for security-sensitive data (permissions)? Why?

---

##  Coherence vs Availability—Pick Your Poison (CAP in Cache Clothing)

Scenario: during a partition, eu-west cannot reach us-east.

You must decide what eu-west does on cache miss:

A) Serve stale value from cache.

B) Fail the request (or degrade) because you can’t guarantee freshness.

C) Route to another region (higher latency).

Pause and think: which choice corresponds to which priority?

- (A) prioritizes _______
- (B) prioritizes _______
- (C) prioritizes _______

Progressive reveal -> Answer:

- (A) prioritizes availability + latency over freshness.
- (B) prioritizes consistency/freshness over availability.
- (C) prioritizes freshness while trying to keep availability, paying latency.

Key insight: in multi-region caching, you are repeatedly making micro-CAP decisions.

**Network assumption (explicit):** assume asynchronous networks with unbounded delay during partitions; you cannot distinguish “slow” from “partitioned” perfectly.

Challenge question: for each endpoint in your API, would you choose A, B, or C during partitions?

---

##   Observability—How to Know Your Cache Is Lying

Scenario: you suspect coherence issues but can’t prove them.

What to measure:

1. Staleness distribution
   - difference between cached version/time and source-of-truth version/time.

2. Invalidation lag
   - time from DB commit -> invalidation processed in each region.

3. Miss refill source
   - which replica/region served the refill.

4. Reintroduced staleness rate
   - cache set events where version decreases.

[IMAGE: Dashboard mock showing invalidation lag percentiles per region, staleness histogram, and “stale refill” counter.]

Interactive exercise: design a metric name and labels for “stale refill detected.”

Pause and think.

Possible answer:

- `cache_stale_refill_total{region, cache_layer, keyspace, source_replica}`

Key insight: without measuring version regressions and invalidation lag, you’re debugging with vibes.

Challenge question: what is the SLO for coherence? “99% of reads within 2 seconds of latest write” — how would you compute it?

---

##   Implementation Sketch—Versioned Cache-Aside with Invalidation

Below is a simplified pattern that handles:

- at-least-once invalidations,
- reordering,
- stale refills,

…assuming you have monotonic versions.

### Correctness note (important)
The original sketch used raw TCP `data` events and `JSON.parse(buf)`. In Node.js, TCP is a byte stream: a single `data` event can contain partial JSON or multiple JSON messages. Production code must frame messages (newline-delimited JSON, length-prefix, etc.) and buffer accordingly.

```javascript
// Versioned invalidation handler over TCP sockets (Node.js)
// Concept: at-least-once invalidations + reordering-safe conditional delete.
// Uses newline-delimited JSON (NDJSON) framing.

const net = require('net');

const cache = new Map(); // key -> { value, version }

function invalidateUpto(key, version) {
  if (typeof key !== 'string' || key.length === 0) throw new Error('bad key');
  if (!Number.isInteger(version) || version < 0) throw new Error('bad version');
  const cur = cache.get(key);
  if (!cur || cur.version > version) return false; // ignore stale invalidation
  cache.delete(key);
  return true;
}

net
  .createServer((socket) => {
    socket.setEncoding('utf8');
    let buf = '';

    socket.on('data', (chunk) => {
      buf += chunk;
      while (true) {
        const idx = buf.indexOf('\n');
        if (idx === -1) break;
        const line = buf.slice(0, idx);
        buf = buf.slice(idx + 1);
        if (line.trim().length === 0) continue;

        try {
          const msg = JSON.parse(line); // { key, version }
          const ok = invalidateUpto(msg.key, msg.version);
          socket.write(JSON.stringify({ ok }) + '\n');
        } catch (e) {
          socket.write(JSON.stringify({ error: e.message }) + '\n');
        }
      }
    });

    socket.on('error', () => socket.destroy());
  })
  .listen(7070, '127.0.0.1');

// Usage: send {"key":"product:1234:price","version":17}\n to 127.0.0.1:7070
```

Pause and think: where is the hardest unsolved problem in this sketch?

Progressive reveal -> Answer:

- Obtaining a monotonic version that all regions agree on.
- Ensuring refill reads are fresh enough (replica too stale).

Key insight: the cache is only as coherent as your ability to define “newer” and to fetch “fresh.”

Challenge question: if your database can’t provide a global monotonic version, what alternatives do you have?

---

##  Real-World Usage Patterns (What Big Systems Often Do)

Pattern: CDN + regional cache + DB

- Static-ish objects (catalog, images) -> CDN with TTL and purges.
- Dynamic user data -> regional cache with session guarantees.
- Financial/critical -> bypass caches or validate on read.

Pattern: “Control plane vs data plane”

- Control plane (configs, flags) uses push propagation and versioning.
- Data plane (user content) uses eventual + client-centric.

Pattern: Key ownership (single-writer per key)

- Assign each key range to a home region.
- Writes go to owner; others read replicas.

Pros: simplifies conflict resolution.

Cons: cross-region write latency for some users.

Key insight: many successful multi-region systems avoid global strong coherence by designing ownership and user routing.

Challenge question: if you introduce key ownership, how do you handle a region outage of the owner?

---

## Choose Your Coherence Contract (Interactive Worksheet)

You’re designing caching for three endpoints:

1. `GET /product/{id}`
2. `GET /cart` and `POST /cart/items`
3. `GET /balance`

Step 1: Pick a coherence contract
For each endpoint, choose one:

- Eventual + TTL
- Read-your-writes
- Monotonic reads
- Strong consistency (no stale)

Pause and think.

Step 2: Pick a strategy
Choose a strategy per endpoint:

- TTL only
- Invalidation bus
- Versioned invalidation
- Sticky sessions
- Version tokens + fallback
- No cache

Step 3: Pick failure behavior
During partition:

- Serve stale
- Fail closed
- Route cross-region

Key insight: coherence is a product requirement expressed as an engineering contract.

Challenge question: write down one explicit sentence per endpoint: “We guarantee X under Y conditions.” If you can’t, your cache will surprise you in production.

---

##  The “Global Price Change” Incident Postmortem Game

Scenario: you are on-call. A global price change rollout caused:

- us-east correct immediately
- eu-west stale for 12 minutes
- ap-southeast oscillating values

You have:

- TTL=10 minutes on product cache
- invalidations via Kafka
- cache-aside refill from local DB replica
- active-passive DB replication (primary in us-east)

Your tasks (pause and think before reading answers):

1) Identify three plausible root causes.

2) For each root cause, propose one instrumentation signal that would confirm/deny it.

3) Propose two design changes that reduce probability of recurrence.

Progressive reveal -> Possible answers:

1) Plausible root causes:

- Kafka consumer lag or partition in eu-west -> invalidations not processed.
- Local DB replica lag in eu-west -> cache refills reintroduce stale values.
- Out-of-order invalidations or missing versioning -> ap-southeast deletes newer entries or refills stale.

2) Instrumentation signals:

- Consumer lag metrics per region: `kafka_consumer_lag{region, topic}`
- Replica freshness metric: `db_replica_lag_seconds{region, replica}`
- Cache version regression counter: `cache_version_regression_total{region}`

3) Design changes:

- Add versioned invalidation and conditional delete/update.
- For price changes, switch to push update (update caches with new value) or temporarily bypass cache.
- Reduce TTL or add soft TTL with background refresh but coupled with version checks.
- For critical admin flows, enforce read-from-primary for a short window.

**Trade-off callout:** enforcing read-from-primary improves freshness but increases cross-region latency and can overload the primary during bursts. Consider scoped bypass (only for keys/users touched recently) and rate-limit.

Key insight: multi-region cache coherence is an end-to-end system property. Fixes usually span messaging, read consistency, and cache semantics.

Final challenge question: if you could only implement one improvement this quarter, which would you pick and why?

---

## The Mental Model to Keep

- Caches are distributed replicas with weaker semantics.
- Coherence is about bounded staleness and client experience, not perfection.
- Versioning + idempotent invalidation + fresh-enough refill sources are your core tools.

[IMAGE: Summary visual: a “coherence triangle” with corners: Freshness, Availability, Latency; show typical strategies plotted inside.]
