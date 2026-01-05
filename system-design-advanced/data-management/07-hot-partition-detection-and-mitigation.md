---
slug: hot-partition-detection-and-mitigation
title: Hot Partition Detection and Mitigation
readTime: 20 min
orderIndex: 7
premium: false
---





## Hot Partition Detection and Mitigation

> Audience: engineers building/operating distributed databases, stream processors, caches, and event-driven systems.
>
> Goal: learn to detect, diagnose, and mitigate hot partitions without trading one failure mode for another.

---

## [CHALLENGE] The "One Counter Kills the Cluster" Incident

Its 10:17 AM. Your on-call phone rings.

- p99 latency jumped from 35 ms -> 2.8 s.
- CPU on one storage node is pegged at 100%.
- The rest of the cluster looks bored.
- Autoscaling added nodes, but nothing improved.

A product manager messages: "We launched a promotion banner. Traffic is up, but why is one node dying?"

### Pause and think
If the system is "distributed," why would a single node become the bottleneck?

- A) Bad luck: random skew
- B) Cache miss storm
- C) Hot partition / hot key
- D) Network packet loss

> Dont answer yet - keep reading. Well come back with evidence.

Real-world analogy (coffee shop chain):
Imagine a coffee shop chain with 20 baristas across 5 counters. A new promotion says: "Free espresso if you ask for 'THE SPECIAL'." Suddenly, everyone queues at counter #3 because thats the only counter that knows the special phrase. The chain is "distributed," but the workload isnt.

Key insight:
> Hot partitions happen when routing concentrates load on a subset of shards/partitions/keys, turning a distributed system into a single-threaded system.

Production note:
- Autoscaling often fails here because the bottleneck is not cluster capacity; its placement + routing. Unless the hot partition can move/split or the hot key can be sharded/cached, new nodes remain idle.

---

## [HANDSHAKE] What Exactly Is a Hot Partition?

### Scenario
You have a partitioned dataset or stream:
- a distributed DB table sharded by user_id
- a Kafka topic partitioned by account_id
- a cache keyed by session_id

Everything is fine until one key/partition starts receiving a disproportionate share of:
- reads
- writes
- updates
- compactions
- locks
- replication traffic

### Interactive question (pause and think)
Which of the following is a hot partition?

1) One partition has 10x the QPS of others.
2) One partition has 10x the data size of others.
3) One partition has 10x the compaction I/O of others.
4) One partition has 10x the lock contention of others.

Pause and think.

Progressive reveal (answer)
All of them can be "hot," depending on what resource is saturated.

- Hot by traffic (QPS/throughput)
- Hot by size (storage/compaction/rebalancing time)
- Hot by contention (locks, MVCC conflicts)
- Hot by replication (leader/follower imbalance)

Restaurant analogy:
A restaurant has 10 kitchens (shards). If 70% of orders are "Burger #1," and only Kitchen 7 makes it, you get a hot kitchen. If Kitchen 7 also stores all frozen buns (data size), its hot in multiple dimensions.

Key insight:
> "Hot partition" is not just a metric; its a resource dominance pattern: one partition dominates CPU, I/O, memory, network, or coordination.

Clarification:
- "Hot partition" is often used loosely to mean "hot shard leader." In leader-based systems (Kafka, Raft groups, many DBs), the leader is the write bottleneck and often the read bottleneck too (depending on read routing).

---

## [WARNING] Why Hot Partitions Are So Dangerous in Distributed Systems

### Scenario
Your system has:
- partition leaders
- replication
- background maintenance (compaction, GC)
- load balancers

A hot partition doesnt just slow itself down; it triggers feedback loops.

### Think about it
What happens when a partition leader is overloaded?

- clients retry -> more load
- replicas fall behind -> read repair or leader elections
- timeouts -> circuit breakers open -> traffic shifts -> new hot spots
- compaction lags -> read amplification increases -> even more CPU

Progressive reveal
Hot partitions are dangerous because they create nonlinear degradation:

1. Queueing effects: latency grows superlinearly as utilization approaches 100% (classic M/M/1 intuition: as rho->1, wait time explodes).
2. Retry storms: timeouts cause clients to amplify traffic.
3. Replica divergence: follower lag forces leader to do extra work (catch-up, read repair, snapshotting).
4. Maintenance starvation: compaction/checkpointing falls behind.
5. Failover cascades: overloaded leader fails -> new leader inherits hot partition -> repeats.

Delivery service analogy:
If one neighborhood suddenly orders 10x more deliveries, one depot gets slammed. Drivers start missing deadlines, customers reorder ("retry"), depot backlog grows, and eventually the depot shuts down - forcing the next closest depot to take over and also collapse.

Key insight:
> Hot partitions are "distributed" failure modes with centralized symptoms.

Production insight:
- Many outages are caused by control-plane reactions (retries, rebalances, elections) rather than the initial skew.
- During incidents, prefer mitigations that reduce work (shed, cache, coalesce) over those that move work (rebalance) unless youre sure the hotspot is placement-only.

---

## [MENTAL MODEL] How Partitioning Creates Hot Spots

### Mental model
Partitioning is a function:

> partition = f(key)

Common choices:
- hash partitioning: hash(key) % N
- range partitioning: key in [a, b)
- consistent hashing / ring
- application routing tables

Hot partitions happen when:
- key distribution is skewed
- time-based locality concentrates writes
- "celebrity keys" dominate
- partition counts are too low
- leadership is imbalanced

### Interactive exercise: Match the cause to the symptom

Match each cause (left) to the likely symptom (right):

| Cause | Symptom |
|---|---|
| A) Range partitioning by timestamp | 1) Latest partition hottest, oldest cold |
| B) Hash by user_id, but one user is a bot | 2) One key dominates QPS |
| C) Partition leader placement uneven | 3) One node has many leaders |
| D) Compaction debt on one shard | 4) I/O spikes, read amp increases |

Pause and think.

Answer
A->1, B->2, C->3, D->4.

Coffee shop analogy:
If you route customers by last-name range (A-F, G-L, ...), and a celebrity "Kardashian" shows up with 1,000 fans, the G-L counter melts.

Key insight:
> Hot partitions are often "correct behavior" of the partition function under skewed inputs.

Network assumption (explicit):
- Assume an asynchronous network: variable latency, occasional packet loss, partitions possible. This matters because retries, timeouts, and leader elections interact with hotspots.

---

## [GAME] Decision Game - Is This a Hot Partition or Something Else?

### Scenario
You observe:
- One node high CPU
- p99 latency high
- cluster-wide QPS stable

Which statement is most likely true?

1) "Its a network issue; hot partitions dont affect only one node."
2) "Its a GC pause; hot partitions would show high QPS everywhere."
3) "Its a hot partition; one shard is overloaded while others idle."
4) "Its a slow disk; hot partitions require skewed keys, not hardware."

Pause and think.

Progressive reveal
3 is most likely.

But you must prove it. Hardware issues can mimic hot partitions (and vice versa). The distinguishing feature is correlation with partition/key-level metrics.

Key insight:
> Treat "hot partition" as a hypothesis that must be validated with per-partition telemetry.

Production checklist to disambiguate:
- If CPU is high but partition QPS is not skewed: suspect GC, kernel issues, noisy neighbor, or a single expensive query pattern.
- If disk latency spikes correlate with one shards compaction: suspect storage engine maintenance.
- If network retransmits correlate with one node: suspect NIC/ToR issues.

---

## [HANDSHAKE] Observability Prerequisites - You Cant Fix What You Cant See

### Scenario
Your monitoring shows node-level CPU and disk, but nothing about partitions.

You attempt mitigation:
- add nodes
- increase thread pools
- raise timeouts

Nothing works.

### Pause and think
Why doesnt adding nodes help?

Progressive reveal
Because the hot partition is still mapped to the same leader/owner. Without re-partitioning or rebalancing, youve only added idle capacity.

### What you need (minimum viable hot-partition observability)
Per partition/shard (and ideally per key for top offenders):
- QPS (reads/writes)
- bytes in/out
- latency histograms
- queue depth / in-flight requests
- CPU time attributed (or service time)
- lock wait time / conflict rate
- compaction backlog / LSM levels / SST counts
- replication lag (leader->follower)
- error rates / timeouts

#### [IMAGE] Partition heatmap dashboard (ASCII mock)

```
Time ->

Partition   QPS      p99 Lat   CPU time   Repl lag   Compaction
---------  -------  ---------  ---------  ---------  ----------
0          ......   .......    .......    .......    .......
...
17         #######  ########   ########   #######    ######
...
255        ......   .......    .......    .......    .......

Legend: . low  # high
Drill-down: partition 17 -> top keys, request types, leader node, follower lag
```

Key insight:
> Node metrics tell you where pain is. Partition metrics tell you why.

Production insight:
- Be careful with high-cardinality labels (e.g., key=...). Prefer:
  - top-K sampling exported separately
  - sketches (Count-Min, SpaceSaving)
  - tracing logs sampled, analyzed offline

---

## [INVESTIGATE] Detection - From "Somethings Slow" to "Partition 17 Is Hot"

### Scenario
Your cluster has 256 partitions. You suspect partition 17 is hot.

### Detection patterns (ranked by usefulness)
1. Heatmaps: partition vs time colored by QPS/latency/CPU.
2. Top-N partitions: list partitions by request rate or service time.
3. Skew indices: quantify imbalance (Gini coefficient, p95/p50 ratio across partitions).
4. Queueing indicators: per-partition queue depth, saturation.
5. Key sampling: top keys within hot partition.

### Interactive exercise: Choose a detection metric
You have to pick one metric to page on for hot partitions. Which is best?

- A) Node CPU > 90%
- B) Partition QPS ratio (max / median) > 10
- C) Cluster-wide QPS > baseline + 30%
- D) Disk utilization > 80%

Pause and think.

Progressive reveal
B is the most directly targeted.

Node CPU (A) is too indirect. Cluster QPS (C) misses skew. Disk utilization (D) is slow-moving and noisy.

Key insight:
> Hot partition detection is fundamentally about relative imbalance, not absolute load.

Alerting production note:
- Page on skew + impact (e.g., skew>10x AND p99>threshold AND queue depth rising) to avoid false positives during normal diurnal patterns.

---

## [PUZZLE] Quantifying Skew (So You Dont Argue in Slack)

### Scenario
Two engineers disagree:
- Engineer 1: "Partition 17 is hot."
- Engineer 2: "Its just normal variance."

You need a number.

### Options to quantify skew
1) Max/Median ratio
- Simple, intuitive
- Sensitive to outliers

2) p95/p50 across partitions
- More robust than max

3) Gini coefficient
- Measures inequality across partitions
- Great for dashboards and alerts

4) Entropy / KL divergence
- Useful if you know expected distribution

### Mini exercise (pause and think)
If you have 10 partitions and QPS distribution:

[10, 9, 11, 10, 10, 9, 10, 10, 10, 100]

What is max/median?

Pause and compute.

Answer
Median is ~10, max is 100 -> 10x skew.

```python
# Implementing skew metrics (max/median, p95/p50, Gini) for hot-partition detection
from __future__ import annotations

from typing import Iterable, Dict


def skew_metrics(qps: Iterable[float]) -> Dict[str, float]:
    vals = sorted(float(x) for x in qps)
    if not vals or any(x < 0 for x in vals):
        raise ValueError("qps must be a non-empty list of non-negative numbers")

    n = len(vals)
    median = vals[n // 2] if n % 2 else (vals[n // 2 - 1] + vals[n // 2]) / 2
    p50 = median

    # Use a simple nearest-rank style index for p95.
    # For alerting, consistency matters more than the exact quantile estimator.
    p95_idx = min(n - 1, int(round(0.95 * (n - 1))))
    p95 = vals[p95_idx]

    max_v = vals[-1]

    # Gini coefficient (0=perfectly even, 1=all load on one partition)
    total = sum(vals)
    if total == 0:
        return {"max_median": 0.0, "p95_p50": 0.0, "gini": 0.0}

    cum = 0.0
    for i, x in enumerate(vals, 1):
        cum += i * x
    gini = (2 * cum) / (n * total) - (n + 1) / n

    return {
        "max_median": max_v / max(median, 1e-9),
        "p95_p50": p95 / max(p50, 1e-9),
        "gini": gini,
    }


# Usage example
if __name__ == "__main__":
    print(skew_metrics([10, 9, 11, 10, 10, 9, 10, 10, 10, 100]))
```

Key insight:
> Use a skew metric to move from "feels hot" to "measurably hot."

Production insight:
- Track skew separately for reads, writes, and bytes. A partition can be "hot by bytes" (large scans) without being "hot by QPS."

---

## [WARNING] Common Misconceptions (Hot Partitions Edition)

### Misconception 1: "Hash partitioning prevents hot partitions."
Reality: Hashing spreads random keys well, but it cannot fix skewed popularity (celebrity keys). If 40% of traffic targets one key, hashing faithfully routes 40% to one partition.

### Misconception 2: "Just add more partitions."
Reality: More partitions can reduce average load per partition, but the hottest key still maps to exactly one partition. You may reduce collateral damage, but not the root cause.

### Misconception 3: "Caching always fixes hot keys."
Reality: Caches help read-heavy hot keys, but:
- write-heavy hot keys still hurt
- cache stampedes can amplify load
- cache invalidation can create bursts

### Misconception 4: "Rebalancing solves it."
Reality: Rebalancing moves partitions across nodes, but if the partition is intrinsically hot, youre just moving the hotspot around.

Key insight:
> Hot partitions are usually data-model problems or routing problems, not simply capacity problems.

---

## [CHALLENGE] Root Causes - The Usual Suspects

### Scenario
You found partition 17 is hot. Now you need to know why.

### Root cause categories
1) Hot key (single key dominates)
- counters (global like count)
- "latest" feed key
- feature flag key
- session store for one mega-session

2) Hot range (range partitioning)
- time-based partitions (todays data)
- monotonically increasing IDs

3) Leadership hotspot
- too many leaders on one node
- leader election pinned

4) Background work hotspot
- compaction backlog
- repair/rebuild
- checkpointing

5) Client routing bug
- all clients pinned to one partition due to bad hashing

### Interactive question
Which is the most common hot key in real systems?

- A) user:12345
- B) global_counter
- C) config:rollout_percentage
- D) All of the above

Pause and think.

Answer
D.

Stadium analogy:
In a stadium with many entrances, if everyone insists on using "Gate A" because its the famous one, Gate A becomes the hot partition.

Key insight:
> Hot keys are often created by product features that introduce global coordination.

---

## [INVESTIGATE] Diagnosing Hot Keys (Inside a Hot Partition)

### Scenario
Partition 17 has 10x QPS. Is it one key or many?

### Techniques
1) Heavy hitter detection
- Count-Min Sketch
- SpaceSaving algorithm
- Top-K sampling

2) Distributed tracing with key tags
- careful: high-cardinality tags can blow up your metrics backend

3) Request logs sampling
- sample 1/1000 requests, compute key frequency offline

4) Storage engine introspection
- per-key lock stats
- per-range read/write bytes

#### [IMAGE] Hot partition internal distribution (ASCII)

```
Case A: single hot key dominates

Partition 17 key share:
  key X  ########################## 90%
  others .......................... 10%

Case B: many moderately skewed keys

Partition 17 key share:
  key A  ####### 20%
  key B  ######  18%
  key C  #####   15%
  others ######## 47%
```

### Decision game
If the hot partition is caused by one key, which mitigation is usually most effective?

- A) Move the partition to a bigger node
- B) Increase replication factor
- C) Split the key (key salting / sharded counters)
- D) Increase client timeouts

Pause and think.

Answer
C.

Key insight:
> If the hotspot is a single key, you need key-level sharding, not partition-level shuffling.

---

## [HANDSHAKE] Mitigation Strategy Map (Pick the Right Lever)

### Scenario
You have 30 minutes to stabilize production, then a week to fix properly.

### Two-phase approach
1) Immediate stabilization (minutes-hours)
- shed load
- rate limit
- cache
- move leadership
- isolate noisy tenants

2) Structural fix (days-weeks)
- redesign partition key
- split ranges
- introduce write aggregation
- change data model

### Comparison table: Mitigation techniques

| Technique | Works best for | Pros | Cons / Trade-offs |
|---|---|---|---|
| Rate limiting | sudden bursts / noisy neighbors | fast, safe | drops requests, needs fairness |
| Request hedging | tail latency from rare slow replicas | reduces p99 | can amplify load if misused |
| Caching | read-hot keys | huge win | stampedes, staleness |
| Key salting | single hot key | spreads writes | complicates reads/queries |
| Range splitting | hot ranges | localizes growth | operational complexity |
| Leader rebalancing | leader hotspots | quick relief | moves problem if intrinsic |
| Adaptive partitioning | evolving skew | automatic | complex, can thrash |
| Write coalescing | counters/aggregates | reduces write QPS | increases staleness |

Key insight:
> The mitigation must match the hotspots shape: key, range, leader, or maintenance.

---

## [FIREHOSE] Immediate Tactics - Stop the Bleeding

### Scenario
Partition 17 leader is at 100% CPU. p99 is exploding. Customers are timing out.

### Tactic 1: Backpressure and load shedding

Interactive question:
Where should you apply backpressure first?

- A) At the storage node
- B) At the client
- C) At the load balancer / API gateway
- D) Everywhere simultaneously

Pause and think.

Progressive reveal
Start at C (edge) if possible, because it prevents unnecessary work from entering the system. Then propagate B (client) via retry budgets. Storage-node backpressure (A) is necessary but often too late.

Restaurant analogy:
Stop people at the door before they occupy tables and overwhelm the kitchen.

Key insight:
> Backpressure is most effective when applied upstream.

Production insight:
- Prefer load shedding with explicit errors (429/503 with Retry-After) over silent timeouts; timeouts trigger retries and amplify load.

---

### Tactic 2: Retry budgets (anti-retry storm)

If clients retry blindly, they amplify hotspots.

- Use exponential backoff + jitter
- Cap retries
- Prefer deadline propagation
- Use retry budgets: a fixed percentage of traffic allowed to be retries

```python
# Implementing a retry budget with deadlines + jittered exponential backoff (storage client)
import random
import socket
import time


def call_with_retry_budget(
    host: str,
    port: int,
    payload: bytes,
    *,
    deadline_s: float = 1.0,
    max_retries: int = 3,
    retry_budget: float = 0.05,
) -> bytes:
    if not (0.0 <= retry_budget <= 1.0):
        raise ValueError("retry_budget must be in [0,1]")
    if deadline_s <= 0:
        raise ValueError("deadline_s must be > 0")

    start = time.monotonic()
    attempts = 0
    retries_used = 0

    while True:
        attempts += 1
        remaining = deadline_s - (time.monotonic() - start)
        if remaining <= 0:
            raise TimeoutError("deadline exceeded")

        try:
            with socket.create_connection((host, port), timeout=remaining) as s:
                s.sendall(payload)
                return s.recv(65536)
        except (OSError, socket.timeout) as e:
            if attempts > 1:
                retries_used += 1

            # Budget: allow only a small fraction of attempts to be retries.
            # This is a toy example; production systems usually implement budgets
            # as a token bucket shared across requests.
            allowed_retries = max(1, int((max_retries + 1) * retry_budget))

            if attempts > max_retries + 1 or retries_used > allowed_retries:
                raise ConnectionError(f"request failed after {attempts} attempts") from e

            # Exponential backoff with full jitter; never sleep past deadline.
            base = min(0.05 * (2 ** (attempts - 1)), 0.5)
            sleep_s = min(random.random() * base, max(0.0, remaining))
            time.sleep(sleep_s)


# Usage example
# resp = call_with_retry_budget("127.0.0.1", 9000, b"GET key\n", deadline_s=0.8)
```

Key insight:
> A hot partition often becomes catastrophic only after retries turn it into a traffic multiplier.

Production insight:
- Implement retry budgets per client / per endpoint / per partition when possible.
- Ensure idempotency (idempotency keys, conditional writes) so retries dont corrupt state.

---

### Tactic 3: Hot key caching + request coalescing

For read-heavy hot keys:
- cache at edge or service layer
- use singleflight / request coalescing to prevent stampedes

```javascript
// Implementing request coalescing (singleflight-style) around a hot-key cache miss
// Node.js: coalesce concurrent fetches so only one hits storage.
const inflight = new Map(); // key -> Promise

async function getWithSingleflight(key, cache, fetchFromStore) {
  const cached = cache.get(key);
  if (cached !== undefined) return cached;

  if (inflight.has(key)) return inflight.get(key);

  const p = (async () => {
    try {
      const value = await fetchFromStore(key); // may throw
      cache.set(key, value);
      return value;
    } finally {
      inflight.delete(key); // ensure cleanup on success/failure
    }
  })();

  inflight.set(key, p);
  return p;
}

// Usage example
// const value = await getWithSingleflight("feature_flags:global", cache, store.get);
```

Trade-offs:
- staleness
- cache invalidation complexity

Key insight:
> Caching helps when the hot resource is read compute, not write serialization.

Failure scenario:
- If the backing store is down, singleflight can turn into a "shared failure." Consider:
  - negative caching with short TTL
  - serving stale (SWR)
  - circuit breakers to avoid hammering the store

---

### Tactic 4: Move leadership (when leadership is the hotspot)

If one node hosts too many leaders:
- rebalance leaders
- ensure rack/zone spread
- avoid "preferred leader" pinning

Failure scenario:
Leader moves can cause:
- temporary unavailability
- follower catch-up load
- client metadata cache invalidation

Key insight:
> Leader rebalancing fixes placement hotspots, not intrinsic key hotspots.

Production insight:
- Use load-aware leader placement (CPU/service-time/queue depth) rather than round-robin.
- Rate-limit leader moves; treat them like deploys.

---

## [INVESTIGATE] Structural Fixes - Designing for Skew

### Scenario
Your product has a "global like counter" updated on every page view. Thats a write-hot key.

### Option 1: Sharded counters (key salting)

Instead of:
- likes:global -> single key

Use:
- likes:global:shard_i for i in [0..k)
- reads aggregate across shards

Pause and think
Whats the main trade-off?

- A) Higher write latency
- B) Higher read complexity and eventual consistency
- C) More disk usage only
- D) Requires strong consistency across shards

Answer
B.

```python
# Implementing sharded counters with periodic aggregation (write spread, cheap reads)
import sqlite3
import time


def incr_sharded(conn: sqlite3.Connection, name: str, shard: int, delta: int = 1) -> None:
    if shard < 0:
        raise ValueError("shard must be >= 0")
    conn.execute(
        "INSERT INTO counters(name, shard, value) VALUES(?,?,?) "
        "ON CONFLICT(name, shard) DO UPDATE SET value=value+excluded.value",
        (name, shard, delta),
    )


def refresh_total(conn: sqlite3.Connection, name: str) -> None:
    total = conn.execute(
        "SELECT COALESCE(SUM(value),0) FROM counters WHERE name=?",
        (name,),
    ).fetchone()[0]

    conn.execute(
        "INSERT INTO counter_totals(name, value, updated_at) VALUES(?,?,?) "
        "ON CONFLICT(name) DO UPDATE SET value=excluded.value, updated_at=excluded.updated_at",
        (name, int(total), int(time.time())),
    )


# Usage example (schema assumed):
# counters(name TEXT, shard INT, value INT, PRIMARY KEY(name, shard));
# counter_totals(name TEXT PRIMARY KEY, value INT, updated_at INT)
```

Analogy:
Instead of one cashier handling all tips, each cashier keeps a jar; at closing time, you sum them.

Key insight:
> Sharding a hot key turns serialization into parallelism, at the cost of aggregation complexity.

Consistency model clarification:
- Sharded counters typically provide eventual consistency for reads of the total.
- If you need strict monotonic reads, youll pay with coordination (e.g., single-writer, consensus, or transactional aggregation), which can reintroduce hotspots.

---

### Option 2: Write buffering and coalescing

For counters, metrics, and "last seen" fields:
- buffer writes in memory
- flush periodically
- coalesce multiple updates

Failure scenario:
- buffer loss on crash
- duplicates on retry

Mitigate with:
- WAL for buffer
- idempotent updates

Key insight:
> Coalescing reduces write QPS by changing semantics from "every event" to "latest/aggregate."

Performance note:
- Coalescing reduces write amplification and lock contention, but increases tail latency for "freshness." Make freshness an explicit product requirement.

---

### Option 3: Range partitioning with split/merge (hot ranges)

If partitioning by time or increasing IDs:
- newest range is hottest

Mitigations:
- pre-split future ranges
- split hot ranges dynamically
- merge cold ranges

#### [IMAGE] Range partition timeline (ASCII)

```
Time ->

Old ranges (cold)                 Newest range (hot)
[Jan] [Feb] [Mar] [Apr] [May] ... [Today]
                                  |
                                  +-- split --> [Today-A] [Today-B]
                                               |
                                               +-- split --> [Today-A1] [Today-A2] ...

Background: merge cold ranges to reduce metadata overhead.
```

Key insight:
> Range splits convert a hot range into multiple smaller ranges, but require routing updates and can cause rebalancing churn.

Failure scenario:
- Splits/merges are metadata operations; if metadata is stored in a consensus system, the metadata store can become the new hot partition.

---

### Option 4: Adaptive partitioning (automatic skew handling)

Systems may:
- detect hotspots
- create sub-partitions
- migrate keys

Trade-offs:
- complexity
- oscillation (thrashing)
- metadata overhead

Think about it:
How do you avoid thrashing?

Progressive reveal
Use hysteresis:
- only split when skew persists for N minutes
- only merge when cold persists
- rate-limit moves

Key insight:
> Automation needs stability controls (hysteresis, budgets) to avoid becoming its own outage.

Production insight:
- Also add blast-radius controls:
  - max concurrent moves
  - max bandwidth for rebalancing
  - "do not move" list for currently-hot partitions during incidents

---

## [GAME] Decision Game - Pick a Mitigation Under Constraints

### Scenario
You run a multi-tenant event ingestion pipeline.

- Partition key: tenant_id
- One tenant just onboarded a batch job and now produces 60% of traffic.
- You must keep other tenants healthy.

Which mitigation is best?

A) Increase partitions and keep keying by tenant_id
B) Add per-tenant rate limits and isolate heavy tenant into dedicated partitions
C) Reduce replication factor cluster-wide
D) Disable retries for everyone

Pause and think.

Answer
B.

Why:
- preserves fairness
- isolates noisy neighbor
- doesnt punish all tenants

Key insight:
> Hot partitions are often a fairness problem disguised as a performance problem.

---

## [WARNING] Failure Scenarios You Must Plan For

### Scenario 1: Hot partition triggers leader failover loop

Sequence:
1) Partition leader overloaded
2) health check fails
3) leader election
4) new leader takes over but is also overloaded
5) repeat

Mitigations:
- health checks based on correct signals (avoid CPU-only)
- election dampening
- load-aware leader selection

Production insight:
- Prefer health checks based on ability to serve (queue depth, request timeouts, internal saturation) rather than raw CPU.

### Scenario 2: Rebalancing amplifies hotspot

Rebalancing moves a hot shard:
- triggers data copy
- saturates network/disk
- increases latency for everyone

Mitigations:
- throttle rebalancing
- isolate rebalancing traffic
- avoid moving hottest shards during incident

### Scenario 3: Cache stampede on a hot read key

Mitigations:
- request coalescing
- stale-while-revalidate
- probabilistic early refresh

### Scenario 4: Hot partition + tail latency = retry storm

Mitigations:
- retry budgets
- idempotency keys
- hedged requests with caps

Key insight:
> Hot partitions rarely kill you alone; they kill you via control-plane reactions (retries, elections, rebalancing).

---

## [INVESTIGATE] Distributed Coordination Angle - Hot Partitions in Consensus Systems

### Scenario
You store metadata in a consensus-backed store (Raft/Paxos) and notice one Raft group is overloaded.

### Why it happens
- One Raft group holds "directory" metadata
- Many clients read/write that group
- Leader becomes bottleneck because:
  - it serializes log appends
  - it handles read leases
  - it replicates to followers

Pause and think
Which is true about a hot Raft group?

1) Followers can absorb leader write load automatically.
2) Adding more followers increases write throughput linearly.
3) The leader is a natural hotspot for writes.
4) Raft eliminates hotspots by design.

Answer
3.

### Mitigations (consensus-specific)
- shard metadata across multiple Raft groups
- avoid global monotonic counters in one group
- use leases/read-only queries on followers (if safe)
- batch proposals

#### [IMAGE] Multiple Raft groups with one hot group (ASCII)

```
Clients
  |\
  | \____ many requests
  v
[Raft Group A]   [Raft Group B]   [Raft Group C]
   Leader*           Leader            Leader
   ^^^^^^^
   HOT: handles most writes + (often) linearizable reads

*Followers replicate but do not remove leader serialization for writes.
```

Key insight:
> Consensus groups are partitions too; a "hot partition" can be a hot Raft group.

CAP/consistency note:
- Linearizable writes require leader coordination; under network partitions you must choose:
  - CP: reject/unavailable rather than risk split-brain
  - AP: accept writes locally and reconcile later (not Rafts model)
- Many "hot partition" mitigations (caching, coalescing, async aggregation) intentionally relax consistency to preserve availability and latency.

---

## [HANDSHAKE] Stream Processing Angle - Hot Partitions in Kafka/Flink/Spark

### Scenario
Kafka topic has 48 partitions. Consumer group lags, but only on partitions 3 and 17.

### Why it happens
- producer keys skewed
- "unknown" key defaulted to constant
- partitioner bug

Interactive question
If you increase consumer parallelism, will lag disappear?

Pause and think.

Progressive reveal
Only if you can increase partition parallelism or change key distribution. If one partition is hot, one consumer still owns it (within a consumer group), so lag persists.

### Mitigations
- change partition key (include more entropy)
- use sticky partitioning carefully
- split topics / add partitions (with caveats)
- implement two-stage aggregation: pre-aggregate by random shard, then final aggregate

```javascript
// Implementing a salted partitioner (KafkaJS-style) to spread hot tenants while keeping per-tenant ordering per shard
// Concept: partitionKey = `${tenantId}:${shard}` where shard is stable per message key.
const crypto = require('crypto');

function stableShard(tenantId, messageKey, shardCount) {
  if (!Number.isInteger(shardCount) || shardCount <= 0) throw new Error('invalid shardCount');
  const h = crypto.createHash('sha256').update(`${tenantId}|${messageKey}`).digest();
  return h.readUInt32BE(0) % shardCount;
}

function choosePartition({ tenantId, messageKey, partitions, shardCount = 8 }) {
  if (!tenantId || !messageKey) throw new Error('tenantId and messageKey required');
  if (!Number.isInteger(partitions) || partitions <= 0) throw new Error('invalid partitions');

  const shard = stableShard(tenantId, messageKey, shardCount);
  const saltedKey = `${tenantId}:${shard}`;
  const h = crypto.createHash('md5').update(saltedKey).digest();
  return h.readUInt32BE(0) % partitions;
}

// Usage example
// const p = choosePartition({ tenantId: 't1', messageKey: 'order-123', partitions: 48 });
```

Key insight:
> In streams, hot partitions show up as consumer lag concentration per partition.

Ordering clarification:
- Salting changes ordering guarantees. You can preserve ordering within a sub-stream (e.g., per order_id) but not necessarily for all events of a tenant unless you keep them on one partition.

---

## [PUZZLE] Trade-offs - Consistency, Ordering, and the Cost of Spreading Load

### Scenario
You want to "just salt the key." But your product requires:
- per-user ordering
- read-your-writes
- uniqueness constraints

Pause and think
Which property is hardest to preserve while mitigating hot keys?

- A) Availability
- B) Ordering
- C) Latency
- D) Storage cost

Answer
Often B (ordering).

### Comparison table: Mitigation vs semantics

| Mitigation | Ordering | Strong consistency | Complexity |
|---|---:|---:|---:|
| Key salting | No (unless careful) | Harder | Medium |
| Two-stage aggregation | Per shard yes, global no | Harder | High |
| Leader rebalancing | Yes | Yes | Low |
| Caching | Reads yes | No (stale) | Medium |
| Range splitting | Within range yes | Yes | Medium |

Key insight:
> Hot partition mitigation is often a negotiation between throughput and semantic guarantees.

CAP framing (practical):
- Under partitions, you cant have both strict consistency and availability. Many mitigations intentionally move you toward AP for reads (cache/SWR) while keeping CP for writes (leader/consensus), or vice versa depending on product needs.

---

## [WARNING] Operational Playbook - Step-by-Step Incident Response

### Scenario
Its happening again. You have 15 minutes to restore SLOs.

### Step 0: Confirm symptom pattern
- p95/p99 latency up
- one node or one partition saturated

### Step 1: Identify the hot partition
- top partitions by service time
- heatmap

### Step 2: Determine hotspot type

Decision tree (pause and think):
- If one partition hot, but many keys inside it are evenly used -> likely range or tenant skew.
- If one key dominates -> hot key.
- If node hot due to many leaders -> leader placement.
- If I/O hot with compaction backlog -> maintenance hotspot.

Progressive reveal
Use the above to pick mitigation.

### Step 3: Apply safe immediate mitigations
- rate limit or shed load
- disable aggressive retries
- enable caching/coalescing
- move leaders (if safe)

### Step 4: Post-incident: structural fix
- change partitioning
- implement key sharding
- add fairness controls

Key insight:
> Incidents are won by classification speed: identify the hotspot type quickly.

Production insight:
- During the incident, freeze "helpful automation" that can amplify churn:
  - aggressive rebalancers
  - auto-leader-movers
  - auto-scaling that triggers re-sharding without guardrails

---

## [INVESTIGATE] Designing Alerts That Dont Cry Wolf

### Scenario
You add an alert "max/median partition QPS > 5" and get paged constantly during normal diurnal patterns.

### Better alert design
- alert on sustained skew (e.g., >10x for 10 minutes)
- combine skew with saturation (queue depth/latency)
- separate read vs write skew
- add "top key share" if available

### Matching exercise
Match alert type to what it catches:

| Alert | Catches |
|---|---|
| A) max/median QPS skew | 1) uneven traffic distribution |
| B) partition queue depth | 2) saturation / backpressure |
| C) replication lag on one partition | 3) leader overload / network |
| D) compaction backlog | 4) LSM maintenance debt |

Pause and think.

Answer
A->1, B->2, C->3, D->4.

Key insight:
> Alerts should detect imbalance + impact, not just imbalance.

---

## [HANDSHAKE] Real-World Patterns (What Big Systems Actually Do)

### Pattern 1: "Celebrity user" isolation
- detect top users/tenants
- route them to dedicated shards

### Pattern 2: Multi-level partitioning
- first by tenant
- then by hashed sub-key

### Pattern 3: Hierarchical aggregation
- edge aggregation -> regional -> global

### Pattern 4: Load-aware placement
- place hot shards on stronger nodes
- keep headroom

### Pattern 5: Explicit fairness
- per-tenant quotas
- weighted fair queueing

Key insight:
> At scale, hot partition mitigation becomes traffic engineering + fairness, not only data modeling.

---

## [PUZZLE] Worked Example - Hot Partition in a Sharded KV Store

### Scenario
You operate a sharded KV store with:
- 128 shards
- shard leader per shard
- replication factor 3
- hash partitioning by key

A new feature introduces key feature_flags:global read on every request and updated frequently.

### Step-by-step diagnosis
1) Heatmap shows shard 42 has 25x read QPS.
2) Within shard 42, top key is feature_flags:global with 90% of reads.
3) CPU on shard 42 leader is saturated; followers are fine.

Pause and think
Whats the best fix?

- A) Increase shard count to 1024
- B) Cache feature_flags:global in application with TTL + singleflight
- C) Move shard 42 to a bigger node
- D) Increase replication factor to 5

Progressive reveal
B is the best first fix.

Why:
- It targets the read-hot key directly.
- It avoids changing semantics much (TTL acceptable).

Longer-term you might also:
- move feature flags to a separate system optimized for fanout
- push updates via pub/sub

```typescript
// Implementing TTL cache + singleflight + stale-while-revalidate for hot feature flags
type Fetcher<T> = () => Promise<T>;

export class SWRCache<T> {
  private value?: T;
  private expiresAt = 0;
  private inflight?: Promise<T>;

  constructor(private ttlMs: number, private swrMs: number) {
    if (ttlMs <= 0 || swrMs < 0) throw new Error("invalid ttl/swr");
  }

  async get(fetcher: Fetcher<T>): Promise<T> {
    const now = Date.now();
    if (this.value !== undefined && now < this.expiresAt) return this.value;

    // Serve stale within SWR window while refreshing in background.
    if (this.value !== undefined && now < this.expiresAt + this.swrMs) {
      void this.refresh(fetcher);
      return this.value;
    }
    return this.refresh(fetcher);
  }

  private refresh(fetcher: Fetcher<T>): Promise<T> {
    if (this.inflight) return this.inflight;
    this.inflight = (async () => {
      try {
        const v = await fetcher();
        this.value = v;
        this.expiresAt = Date.now() + this.ttlMs;
        return v;
      } finally {
        this.inflight = undefined;
      }
    })();
    return this.inflight;
  }
}

// Usage example
// const cache = new SWRCache<Flags>(1000, 5000);
// const flags = await cache.get(() => store.getFlags());
```

Key insight:
> Read-hot global config is best handled by fanout distribution, not by hammering storage.

---

## [WARNING] When Mitigation Backfires (And How to Notice)

### Scenario
You deployed key salting for a hot counter. Writes spread out, but now reads are slow and expensive.

### Failure modes
- fan-in reads across many shards increase latency
- partial failures cause undercount
- inconsistent reads cause UI jitter

Pause and think
How do you make reads cheaper?

Progressive reveal
Introduce a materialized aggregate:
- background job periodically sums shards into likes:global:total
- reads hit total
- writes continue to shards

Trade-offs:
- eventual consistency
- more moving parts

Key insight:
> Many mitigations shift cost from write path to read path (or vice versa). Measure both.

Production insight:
- For fan-in reads, consider:
  - parallel reads with timeouts + partial aggregation
  - caching the aggregate
  - storing per-shard deltas and using a streaming aggregator

---

## [CHALLENGE] Final Synthesis - The Hot Partition Gauntlet

### Scenario
Youre designing a new multi-region system:
- user activity events
- near-real-time counters
- per-tenant dashboards

You must choose:
- partition keys
- alerting
- mitigation strategies

### Synthesis challenge (pause and think)
Given requirements:
- per-tenant ordering for events
- dashboards tolerate 30s staleness
- one tenant may be 100x larger than median

Design a strategy:
1) What is your primary partition key?
2) How do you prevent one tenant from melting the cluster?
3) How do you handle global counters?
4) What do you alert on?

Pause and write your answers.

Progressive reveal (one strong solution)
1) Partition by tenant_id, but sub-partition within tenant for scalable processing:
   - partition_key = (tenant_id, shard = hash(event_id) % k(tenant_id))
   - keep ordering where needed by using per-stream keys for ordered subsets.

2) Enforce per-tenant quotas and isolate heavy tenants:
   - dedicated partitions or dedicated topic
   - weighted fair queueing

3) Global counters:
   - sharded counters + periodic aggregation
   - or stream-based aggregation pipeline (edge -> regional -> global)

4) Alerting:
   - skew (max/median) + saturation (queue depth/latency)
   - consumer lag concentration per partition
   - top key share within partition (if measurable)

Key insight:
> The best hot-partition strategy is proactive: design for skew, enforce fairness, and build observability that pinpoints hotspots quickly.

Multi-region note:
- Cross-region replication can turn a hot partition into a WAN hotspot (egress, replication lag). Consider:
  - local writes + async replication for dashboards (AP-ish reads)
  - per-region aggregation then global rollup

---

## [GOODBYE] Closing Challenge Questions

1) In your current system, what are the top 5 keys/tenants by traffic? Do you know?
2) Do you have per-partition queue depth and latency histograms?
3) What happens when a leader is overloaded - do clients retry responsibly?
4) Which mitigation would you deploy in 15 minutes vs 2 weeks?

---

### Appendix: Quick Checklist

- [ ] Partition-level metrics exist and are cheap enough to keep
- [ ] Skew metric (max/median, p95/p50, Gini) tracked over time
- [ ] Top keys for hot partitions discoverable (sampling/sketch)
- [ ] Retry budgets and deadline propagation implemented
- [ ] Rate limiting / fairness controls per tenant
- [ ] Safe leader rebalancing and throttled rebalancing
- [ ] Structural mitigations: salting, splitting, aggregation, caching
