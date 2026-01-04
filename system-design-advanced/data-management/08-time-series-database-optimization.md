---
slug: time-series-database-optimization
title: Time Series Database Optimization
readTime: 20 min
orderIndex: 8
premium: false
---




# Time-Series Database Optimization (Distributed Systems Edition)

> **Audience**: engineers who already ship distributed systems and now need their time-series stack to stay fast, cheap, and correct under load.

---

## [CHALLENGE] The “Black Friday Telemetry Storm”

It’s 11:58 PM. Your on-call phone is quiet. Two minutes later, your dashboards freeze.

- Your fleet of services emits metrics every 10s.
- A new feature doubles cardinality (oops).
- A regional network flap triggers retries.
- Your time-series database (TSDB) starts timing out on writes.
- Query latency spikes right when incident response needs graphs.

**Scenario**: You run a distributed TSDB (or a single-node TSDB behind a distributed ingestion tier). You must keep:

- **Ingest**: sustained high write throughput
- **Query**: low-latency reads for recent data + reasonable performance for long-range analytics
- **Retention**: predictable storage costs
- **Correctness**: no silent data loss, and clear semantics when failures happen

### [PAUSE AND THINK]
If you could change only one thing *right now* to survive the storm, which would it be?

1) Increase replication factor
2) Increase shard count
3) Reduce series cardinality
4) Add a cache in front of queries

Don’t answer yet—keep it in mind. We’ll return to it after building the mental model.

---

## [MENTAL MODEL] A TSDB is a delivery service for timestamps

Imagine a city-wide delivery service:

- **Packages** = samples (timestamp, value)
- **Address** = series identity (metric name + labels/tags)
- **Neighborhoods** = shards/partitions
- **Warehouses** = nodes
- **Delivery routes** = ingestion path + compaction
- **Customer requests** = queries over time ranges

A TSDB must do two opposing jobs:

- **Accept packages fast** (write-optimized)
- **Answer “what happened between 9:00–9:05?” fast** (read-optimized)

It achieves this by **buffering** and **organizing** data over time.

### [KEY INSIGHT]
> **Most TSDB optimizations are about controlling amplification**:
> - **Write amplification** (how many bytes written per byte ingested)
> - **Read amplification** (how many bytes scanned per byte returned)
> - **Space amplification** (how many bytes stored per byte of raw samples)
> - **Network amplification** (how many nodes touched per query/write)

---

## [SECTION] The ingestion pipeline (where performance is born or dies)

### [CHALLENGE] “Why are my writes timing out?”

Your ingestion path typically includes:

1. **Client batching**
2. **Load balancer / ingest gateway**
3. **Routing** to a shard owner
4. **Write-ahead log (WAL)**
5. **In-memory buffer (memtable/head)**
6. **Flush to immutable files** (segments/blocks)
7. **Compaction** (merge, dedupe, downsample)

### [PAUSE AND THINK]
Where do you expect the first bottleneck under a sudden ingest spike?

- A) Network bandwidth
- B) CPU (compression/encoding)
- C) Disk IOPS (WAL fsync)
- D) Lock contention / memory pressure

Hold your answer.

### Explanation (coffee shop analogy)
Think of a coffee shop:

- The **cashier** = ingestion gateway
- The **order ticket printer** = WAL
- The **counter** holding drinks to be picked up = in-memory buffer
- The **fridge storage** = on-disk immutable blocks
- The **barista reorganizing supplies** = compaction

If the ticket printer jams (WAL fsync), the whole line stops.

### Real-world parallel
Many TSDBs (Prometheus TSDB, InfluxDB, ClickHouse MergeTree, Cassandra-based systems, M3DB, VictoriaMetrics) rely on a WAL to survive crashes. The WAL is frequently a **latency floor**.

### [KEY INSIGHT]
> If your WAL is synchronous per request, your *p99 write latency* is often bounded by **fsync latency** and **queueing**.

### [DECISION GAME] Which statement is true?

1) “If I increase replication, ingestion gets faster because more nodes share the load.”
2) “If I batch writes, I can reduce WAL overhead per sample.”
3) “If I compress harder, writes always get faster because fewer bytes hit disk.”
4) “If I add a query cache, write timeouts disappear.”

**Pause and think.**

**Answer reveal**: **2** is generally true. Batching amortizes per-request overhead (WAL fsync, RPC overhead, index updates). (1) can increase coordination cost. (3) may increase CPU and hurt tail latency. (4) doesn’t address ingest bottlenecks.

### Challenge question
If you can only change one client-side behavior to help ingestion, what is it?

- **Answer**: batch and backoff (with jitter), and avoid retry storms.

---

## [CHALLENGE] Cardinality—the silent budget killer

A time series is identified by metric + labels/tags. **Cardinality** is how many distinct series exist.

**Symptom**: Everything looks fine at 50k series. At 5M series, your TSDB collapses:

- index memory explodes
- compaction becomes endless
- queries that touch “all series” become O(N)

### [PAUSE AND THINK]
Which metric is more dangerous?

A) 10 metrics x 1,000,000 series each
B) 1,000 metrics x 10,000 series each

**Think**: both are 10M series total, but query patterns and index layout matter.

### Explanation (restaurant menu analogy)
A restaurant can handle:

- a **big menu** (many metrics) if each dish is ordered by few people
- or **few dishes** (few metrics) if many people order them

But if every customer customizes their dish differently (high label cardinality), the kitchen (index) breaks.

### Real-world parallel
- Kubernetes labels like `pod`, `container`, `request_id`, `user_id` can explode cardinality.
- Logs-to-metrics pipelines often accidentally turn unique IDs into labels.

### [KEY INSIGHT]
> **Cardinality is not just storage**. It’s **index size**, **memory residency**, **compaction fanout**, and **query fanout**.

### [COMMON MISCONCEPTION]
> “Compression will save me from high cardinality.”

Compression helps sample payloads. High cardinality kills you in **metadata and indexes**, which often compress poorly and must be kept hot.

### [MATCHING EXERCISE]
Match the label to “safe-ish” vs “dangerous”:

| Label example | Safe-ish | Dangerous |
|---|---:|---:|
| `region=us-east-1` | [ ] | [ ] |
| `instance=10.2.3.4:9100` | [ ] | [ ] |
| `request_id=...` | [ ] | [ ] |
| `status_code=500` | [ ] | [ ] |
| `user_id=...` | [ ] | [ ] |

**Pause and think.**

**Answer reveal**:
- Safe-ish: `region`, `status_code`
- Often dangerous: `request_id`, `user_id`
- `instance` is acceptable in infra metrics but can be expensive at huge scale (still bounded by fleet size)

### Challenge question
Name one label you should almost never use in metrics.

---

## [MENTAL MODEL] Partitioning = choosing the “neighborhood map”

Distributed TSDBs must decide:

- how to **shard** series across nodes
- how to place data for a **time range**
- how to handle **rebalancing** when nodes join/leave

### [CHALLENGE] You must pick a shard key
Common approaches:

1) **Hash by series key** (metric+labels)
2) **Time-based partitioning** (by day/week)
3) **Hybrid** (hash by series + time blocks)

### [PAUSE AND THINK]
If you shard purely by time (e.g., daily partitions), what happens to write load?

- A) evenly distributed
- B) hotspots on the “current day” partition
- C) no effect

**Answer reveal**: **B**. Everyone writes “now”. Time-only sharding concentrates writes.

### Explanation (delivery service analogy)
If you assign deliveries by **day** instead of by **address**, then *today’s warehouse* gets every package in the city.

### Real-world parallel
- Columnar stores (ClickHouse) partition by time but still distribute by another key (e.g., `ORDER BY (metric, tags, time)` + sharding key).
- Systems like Cortex/Mimir shard by series and store blocks by time.

### [KEY INSIGHT]
> **Good sharding spreads writes and bounds query fanout**. You rarely get both for free.

### [DECISION GAME] Which sharding choice best fits the workload?

Workload: high ingest, most queries are “last 15 minutes for a single service”, occasional “30 days across all services”.

1) shard by time only
2) shard by series only
3) hybrid: series hash + time blocks

**Pause and think.**

**Answer reveal**: **3**. Series-hash spreads ingest; time blocks enable retention, compaction, and cold storage.

### Challenge question
What query pattern becomes expensive when sharding by series hash?

- **Answer**: global aggregates across many series (touches many shards).

---

## [SECTION] Replication, quorum, and “what does success mean?”

### [CHALLENGE] A node dies mid-ingest
You replicate data for durability and availability. But replication raises questions:

- Do you require **quorum acks** for writes?
- What happens during partitions?
- How do you handle duplicates and out-of-order samples?

### [PAUSE AND THINK]
If your TSDB returns HTTP 200 for a write, what do you want that to guarantee?

- A) at least one node has it
- B) a quorum has it
- C) it’s on disk in multiple nodes
- D) it will appear in queries immediately

There’s no universally correct answer—only trade-offs.

### Explanation (restaurant order analogy)
You place an order:

- **Order accepted** != food cooked
- **Food cooked** != delivered to your table
- **Delivered** != you ate it

Write acknowledgment levels map to stages of completion.

### Real-world parallel
- Dynamo-style systems: `W` and `R` quorums.
- Kafka-like ingestion: ack levels and durability.
- Cortex/Mimir: distributor -> ingesters with replication; queries hit ingesters + store.

### [KEY INSIGHT]
> Decide your **write success contract** explicitly. Otherwise, you’ll discover it during an incident.

### [COMMON MISCONCEPTION]
> “Replication factor 3 means I can lose 2 nodes without losing data.”

Only if:
- writes reached all 3
- data was durable (WAL fsync)
- anti-entropy repaired gaps
- you didn’t acknowledge early

### [DECISION GAME] Which statement is true?

1) With `RF=3` and `W=1`, you can lose data on a single-node failure.
2) With `RF=3` and `W=2`, you can never lose data.
3) With `RF=1`, you can still be highly available if you add caches.

**Pause and think.**

**Answer reveal**: **1** is true. `W=1` acknowledges before replication completes.

### Challenge question
What’s the trade-off between `W=quorum` and `W=1` for metrics?

---

## [SECTION] Query optimization: Make the common path scream

### [CHALLENGE] “My dashboards are slow, but only sometimes”

Dashboard queries are usually:

- recent time ranges (last 5m/15m/1h)
- high fanout aggregations (sum/avg over many series)
- repeated every few seconds

### [PAUSE AND THINK]
Which is typically the biggest query accelerator?

A) better compression
B) pre-aggregation / downsampling
C) adding more replicas
D) increasing retention

**Answer reveal**: **B**. Downsampling and rollups reduce scan volume.

### Explanation (coffee shop prep analogy)
If customers constantly order “large drip coffee”, you pre-brew a big batch. That’s downsampling/rollups.

### Real-world parallel
- Prometheus recording rules
- TimescaleDB continuous aggregates
- InfluxDB tasks
- Mimir/Cortex ruler

### [KEY INSIGHT]
> **Most dashboards don’t need raw 10s resolution for 30-day ranges.**

[IMAGE: A diagram showing query path for “recent” (hits ingesters/mem + cache) vs “historical” (hits object storage blocks + index) with fanout across shards and a query-frontend cache.]

### Challenge question
What is the risk of downsampling for incident response?

- **Answer**: it can hide short spikes; you need raw data for short windows.

---

## [SECTION] Indexes: The part you don’t see until it hurts

### [CHALLENGE] Index fits in RAM… until it doesn’t

TSDBs maintain an index mapping:

- series key -> series ID
- series ID -> chunks/blocks containing samples
- label -> series sets (inverted index)

When the index spills to disk, query latency becomes unpredictable.

### [PAUSE AND THINK]
If you have to choose, would you rather keep:

- A) raw samples in cache
- B) index structures in cache

**Answer reveal**: **B**. Without the index, you can’t find the samples efficiently.

### Explanation (library analogy)
Raw samples are books. The index is the card catalog. Without the catalog, finding books is slow even if books are nearby.

### Real-world parallel
- Prometheus: head block index and postings
- ClickHouse: primary key and skip indexes
- Cassandra: partition index vs clustering

### [KEY INSIGHT]
> **Index locality matters more than raw throughput** for interactive queries.

### [COMMON MISCONCEPTION]
> “SSD solves index problems.”

SSDs reduce pain, but the cost is still high: random reads + cache misses + CPU overhead. RAM-resident indexes still win for p99.

### Challenge question
Name one optimization that reduces index size.

---

## [SECTION] Compaction: The janitor that can block the hallway

### [CHALLENGE] Compaction is eating my cluster
Compaction merges small immutable files into larger ones to:

- reduce read amplification
- deduplicate overlapping samples
- enforce retention
- improve compression

But compaction consumes:

- CPU
- disk bandwidth
- temporary disk space

### [PAUSE AND THINK]
What happens if compaction falls behind for 24 hours?

- A) nothing; it will catch up
- B) query latency increases and disk usage spikes
- C) ingest stops immediately

**Answer reveal**: **B** (usually). Many small files increase seeks and metadata overhead; storage grows due to duplicates and poor compression.

### Explanation (restaurant cleanup analogy)
If no one clears tables, the restaurant can still take orders—until every surface is covered and service collapses.

### Real-world parallel
- LSM-tree compaction in RocksDB-based TSDBs
- MergeTree merges in ClickHouse
- Prometheus block compaction and remote storage

### [KEY INSIGHT]
> **Compaction debt** is real operational debt. Track it like error budget.

[IMAGE: Timeline showing ingestion producing many small segments, compaction merging into larger blocks, and how backlog increases read amplification.]

### Challenge question
What’s one safe lever to reduce compaction pressure without losing data?

---

## [SECTION] Failure scenarios: When distributed reality shows up

### [CHALLENGE] Network partition during peak ingest

You have:

- ingest gateways in region A
- storage nodes in regions A and B
- replication across AZs

A partition isolates half of the storage nodes.

### [PAUSE AND THINK]
What do you prefer during the partition?

- A) accept writes in A even if replication to B fails (AP-ish)
- B) reject writes unless quorum is met (CP-ish)

### Explanation (delivery analogy)
Do you accept packages when your second warehouse is unreachable?

- If yes: you might have to reconcile later.
- If no: you avoid divergence but drop/deny packages now.

### Real-world parallel
Metrics are often treated as **AP**: better to accept and be “eventually consistent” than to drop all telemetry.

### [KEY INSIGHT]
> For metrics, **availability often dominates consistency**, but you must engineer **reconciliation** (dedupe, backfill, anti-entropy).

### [COMMON MISCONCEPTION]
> “Metrics are non-critical, so we can drop them.”

During incidents, metrics become critical. Dropping them precisely when failure happens is the worst time.

### Challenge question
What mechanism can you use to survive partitions without losing data?

- Hint: buffering and replay.

---

## [SECTION] Backpressure and overload control: Stop the retry storm

### [CHALLENGE] Retries make it worse
When writes fail, clients retry. If they retry immediately, they amplify load.

### [PAUSE AND THINK]
Which retry policy is safer?

1) immediate retry with fixed delay
2) exponential backoff with jitter + max retry budget

**Answer reveal**: **2**.

### Explanation (traffic jam analogy)
If everyone honks and accelerates into a jam, it gets worse. Backoff is spacing cars out.

### Real-world parallel
- Prometheus remote_write has queueing and backoff knobs
- OpenTelemetry collectors buffer and retry

### [KEY INSIGHT]
> **Backpressure is a feature**. If your TSDB never says “slow down,” it will fail catastrophically.

[CODE: YAML, Prometheus remote_write queue settings demonstrating batching, backoff, and capacity tuning]

```yaml
# Implementing remote_write batching + backoff (Prometheus)
remote_write:
  - url: https://mimir.example.com/api/v1/push
    remote_timeout: 30s
    queue_config:
      capacity: 20000            # buffer to absorb short spikes (watch memory)
      max_samples_per_send: 5000 # batching amortizes WAL/RPC overhead
      batch_send_deadline: 5s    # flush even if batch isn't full
      min_backoff: 200ms         # start gentle
      max_backoff: 10s           # cap retry storm
      max_shards: 200            # parallelism; too high can overload TSDB
    metadata_config:
      send: false                # reduce cardinality/metadata churn if not needed
```

### Challenge question
What is the risk of buffering too much at the edge?

---

## [SECTION] Data modeling for TSDBs: Design for queries you actually run

### [CHALLENGE] Your schema is your query plan
In TSDBs, “schema” often means:

- metric naming
- label set design
- rollups
- retention tiers
- whether you store exemplars/traces/logs links

### [PAUSE AND THINK]
You need per-endpoint latency metrics. Should `path` be a label?

- A) always
- B) never
- C) only if it’s normalized (templated) and bounded

**Answer reveal**: **C**.

### Explanation (menu customization analogy)
If every customer can invent a new dish name, the kitchen collapses. If choices are from a fixed menu, it’s manageable.

### Real-world parallel
Use route templates like `/users/:id` not `/users/12345`.

### [KEY INSIGHT]
> **Bounded label values** are the difference between observability and self-inflicted DDoS.

### Challenge question
Name a label you would normalize before storing.

---

## [SECTION] Distributed query execution: Fanout is the tax you always pay

### [CHALLENGE] “Why does a simple sum() touch 200 nodes?”

In distributed TSDBs, a query often executes as:

1. **Frontend** receives query
2. Splits by time range or shard
3. Dispatches to storage nodes
4. Partial aggregations occur near data
5. Results merge at frontend

### [PAUSE AND THINK]
Where should aggregation happen?

- A) always at the frontend
- B) as close to data as possible

**Answer reveal**: **B**. Push down aggregation reduces network and merge cost.

### Explanation (catering analogy)
If each restaurant branch totals its day’s sales locally, HQ only merges totals—not every receipt.

### Real-world parallel
- Mimir query-frontend + queriers
- Druid brokers + historical nodes
- ClickHouse distributed queries

### [KEY INSIGHT]
> **Query pushdown** converts network amplification into CPU work near data.

[IMAGE: A distributed query plan diagram showing pushdown aggregation on shard nodes, then merge step.]

### Challenge question
What’s the danger of too much pushdown?

---

## [SECTION] Consistency, deduplication, and out-of-order samples

### [CHALLENGE] Two writers, one series
You have HA scrapers or multiple agents writing the same metric series.

### [PAUSE AND THINK]
If two writers send the same timestamp with different values, what should happen?

- A) last-write-wins
- B) reject
- C) store both

It depends on your system’s semantics.

### Explanation (duplicate delivery analogy)
Two couriers deliver the same package. Do you keep both? Return one? Decide based on business rules.

### Real-world parallel
- Prometheus HA pairs with external labels; remote storage may dedupe
- Some TSDBs treat identical timestamps as overwrite or keep-first

### [KEY INSIGHT]
> Define dedupe behavior for **same timestamp**, **out-of-order**, and **late arriving** data.

### [COMMON MISCONCEPTION]
> “Out-of-order is just a minor annoyance.”

Out-of-order can break compression, compaction assumptions, and query correctness.

### Challenge question
What ingestion-side technique reduces out-of-order risk?

---

## [SECTION] Storage tiers: Hot, warm, cold (and object storage reality)

### [CHALLENGE] 400 TB/month and growing
You can’t keep everything on fast disks.

Common architecture:

- **Hot**: recent data on SSD + RAM indexes
- **Warm**: compacted blocks on cheaper disks
- **Cold**: object storage (S3/GCS) with index gateways

### [PAUSE AND THINK]
What gets worse when you move data to object storage?

- A) latency
- B) throughput
- C) failure modes
- D) all of the above

**Answer reveal**: **D**.

### Explanation (warehouse analogy)
Hot storage is your local warehouse. Cold storage is a remote warehouse with slower trucks and occasional delays.

### Real-world parallel
Cortex/Mimir/Thanos store blocks in object storage; query path includes store gateways and caches.

### [KEY INSIGHT]
> Object storage is cheap and durable, but it shifts complexity into **caching, indexing, and consistency handling**.

[IMAGE: Tiered storage diagram with hot ingesters, compactors, object storage, store-gateway, query-frontend cache.]

### Challenge question
Why do many systems keep a “recent window” in ingesters even after shipping blocks?

---

## [SECTION] Caching strategies: Cache what’s expensive to recompute

### [CHALLENGE] Cache everything? Not possible.
Caches in TSDB architectures:

- **Query result cache** (range queries)
- **Chunk cache** (compressed blocks)
- **Index/postings cache**
- **Metadata cache** (label names/values)

### [PAUSE AND THINK]
Which cache is most sensitive to cardinality?

- A) chunk cache
- B) index/postings cache

**Answer reveal**: **B**.

### Explanation (library analogy)
If the catalog grows, caching the catalog pages matters more than caching books.

### Real-world parallel
Thanos store gateway uses index cache; Mimir uses multiple caches.

### [KEY INSIGHT]
> Cache the *join points*: label -> series sets, series -> chunks.

### Challenge question
What cache invalidation strategy works for immutable blocks?

---

## [SECTION] Observability of the TSDB itself: Measure the measurer

### [CHALLENGE] You can’t optimize what you can’t see

Track:

- ingest rate (samples/s)
- WAL fsync latency
- compaction backlog
- index size and cache hit ratio
- query fanout and bytes scanned
- tail latencies (p95/p99)
- dropped samples / rejected writes

### [PAUSE AND THINK]
Which metric is the earliest warning sign of cardinality explosion?

- A) disk usage
- B) index memory
- C) CPU

**Answer reveal**: **B** is often earliest.

### Explanation (restaurant analogy)
Kitchen prep space (RAM) fills before the pantry (disk).

### [KEY INSIGHT]
> Watch **series count**, **active series**, and **index memory** like you watch CPU.

### Challenge question
What SLO would you set for dashboard queries during incidents?

---

## [SECTION] Optimization playbook: Levers and trade-offs

### [CHALLENGE] Pick the right lever under pressure

Below is a comparison table of common optimizations.

| Lever | Helps | Hurts / Risk | Best when |
|---|---|---|---|
| Reduce cardinality | index/memory, query fanout, compaction | loss of granularity | labels are unbounded |
| Client batching | WAL overhead, RPC | larger loss on crash if buffers not flushed | high ingest |
| Increase shards | parallelism | overhead, rebalancing churn | CPU-bound or single shard hot |
| Increase replication | availability | write latency, cost | need HA and can tolerate slower writes |
| Downsampling/rollups | long-range queries | hides spikes | dashboards + billing |
| Tiered storage | cost | complexity, cold query latency | high retention |
| Caching | query latency | consistency, memory | repeated queries |
| Pushdown aggregation | network | CPU hotspots | high fanout queries |

### [PAUSE AND THINK]
Which lever is most likely to fix **write timeouts** quickly?

- A) caching
- B) batching + backpressure
- C) downsampling

**Answer reveal**: **B**.

### [KEY INSIGHT]
> Optimize the path that’s failing: ingest failures rarely yield to query-side tricks.

### Challenge question
If you see WAL fsync p99 spike, what are two immediate mitigations?

---

## [SECTION] Interactive incident simulation: You are the TSDB on-call

### [CHALLENGE SCENARIO]
At 00:03:

- write errors begin
- CPU is 60%
- disk util is 95%
- WAL fsync p99 jumps from 3ms -> 80ms
- series count climbs rapidly

### [PAUSE AND THINK]
Pick the **best first action**:

1) add more queriers
2) throttle ingestion + enable backoff at clients
3) increase query cache size
4) extend retention

**Answer reveal**: **2**. You’re disk-bound on WAL; stop the storm first.

### Explanation (firefighting analogy)
Don’t rearrange furniture while the kitchen is on fire. Reduce inflow, then fix root cause.

### [KEY INSIGHT]
> In distributed systems, **stability** beats **throughput** during incidents.

### Challenge question
After stabilizing, what’s your second action to address root cause?

---

## [SECTION] Progressive optimization: From single node to multi-region

### [CHALLENGE] Your architecture evolves

**Phase 1**: Single-node TSDB
- optimize WAL, disk, retention

**Phase 2**: Sharded ingestion + replicated storage
- consistent hashing, rebalancing, quorum

**Phase 3**: Object storage + compactor + query frontend
- caching, index gateways, eventual consistency

**Phase 4**: Multi-region
- write locality, cross-region replication, failover semantics

### [PAUSE AND THINK]
In multi-region, where should writes go?

- A) always to a single home region
- B) to nearest region, replicate asynchronously
- C) to all regions synchronously

**Answer reveal**: usually **B**, unless strict consistency is required (rare for metrics).

### Explanation (delivery hub analogy)
Ship from nearest hub, then transfer to central warehouse later.

### [KEY INSIGHT]
> Multi-region TSDBs are mostly about **latency and failure domains**, not raw throughput.

### Challenge question
What’s the hardest part of multi-region queries?

---

## [SECTION] Common misconceptions (rapid-fire)

### [COMMON MISCONCEPTION 1] “TSDB optimization is mostly about compression.”
**Reality**: compression helps, but **cardinality, indexing, and compaction** dominate many failure modes.

### [COMMON MISCONCEPTION 2] “More shards always improves performance.”
**Reality**: shards increase parallelism but also overhead: coordination, metadata, rebalancing, query fanout.

### [COMMON MISCONCEPTION 3] “Object storage is infinitely scalable, so we’re done.”
**Reality**: object storage changes the bottleneck to **indexing, caching, and request rate limits**.

### [COMMON MISCONCEPTION 4] “Metrics are eventually consistent so correctness doesn’t matter.”
**Reality**: correctness matters for alerting, SLOs, billing, and incident response. You need explicit semantics.

### Challenge question
Which misconception have you personally seen cause an outage?

---

## [SECTION] Practical exercises (with progressive reveal)

### [EXERCISE 1] Spot the cardinality bomb
You see a metric:

```
http_requests_total{service="api", path="/users/12345", status="200"} 1
```

[PAUSE AND THINK] What’s the problem?

**Answer reveal**: `path` contains unbounded IDs. Normalize to route templates.

### [KEY INSIGHT]
> A single unbounded label can create millions of series.

---

### [EXERCISE 2] Pick the right rollup
You need:

- dashboards for last 6 hours at 10s resolution
- dashboards for last 30 days at 5m resolution

[PAUSE AND THINK] How many rollup levels do you store?

**Answer reveal**: at least two tiers (raw 10s for short window, 5m rollup for long window), possibly 1h rollup for 1y.

### [KEY INSIGHT]
> Rollups are a query-time optimization and a cost-control strategy.

---

### [EXERCISE 3] Quorum semantics
You run RF=3 across AZs. You choose `W=1` for ingestion.

[PAUSE AND THINK] What failure can lose acknowledged writes?

**Answer reveal**: the single node that acked crashes before replicating or before durable fsync.

### [KEY INSIGHT]
> Acknowledgment level defines durability, not replication factor.

---

## [SECTION] Code markers: Where code clarifies key tuning knobs

[CODE: YAML, Prometheus remote_write queue settings demonstrating batching, backoff, and capacity tuning]

```yaml
# Implementing remote_write batching + backoff (Prometheus)
remote_write:
  - url: https://mimir.example.com/api/v1/push
    remote_timeout: 30s
    queue_config:
      capacity: 20000            # buffer to absorb short spikes (watch memory)
      max_samples_per_send: 5000 # batching amortizes WAL/RPC overhead
      batch_send_deadline: 5s    # flush even if batch isn't full
      min_backoff: 200ms         # start gentle
      max_backoff: 10s           # cap retry storm
      max_shards: 200            # parallelism; too high can overload TSDB
    metadata_config:
      send: false                # reduce cardinality/metadata churn if not needed
```

[CODE: SQL, TimescaleDB continuous aggregate creation and refresh policy for downsampling]

```sql
-- Implementing downsampling with TimescaleDB continuous aggregates
-- Requires: CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE MATERIALIZED VIEW IF NOT EXISTS cpu_5m
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('5 minutes', time) AS bucket,
  host,
  avg(usage) AS avg_usage,
  max(usage) AS max_usage
FROM cpu
GROUP BY bucket, host;

SELECT add_continuous_aggregate_policy('cpu_5m',
  start_offset => INTERVAL '2 hours',
  end_offset   => INTERVAL '5 minutes',
  schedule_interval => INTERVAL '1 minute');
```

[CODE: SQL, ClickHouse table definition using MergeTree with PARTITION BY toYYYYMM(time) and ORDER BY (metric, tags..., time) plus TTL for retention]

```sql
-- Implementing time partitioning + locality with ClickHouse MergeTree
CREATE TABLE IF NOT EXISTS metrics
(
  time DateTime,
  metric LowCardinality(String),
  tags Map(LowCardinality(String), String),
  value Float64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(time)
ORDER BY (metric, tags, time)
TTL time + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

[CODE: Go, example of client-side batching and exponential backoff with jitter for remote write]

```python
# Implementing client-side batching + exponential backoff with jitter (remote_write-like)
import random, time, json, socket

def send_batch(host: str, port: int, batch: list[dict], timeout_s: float = 2.0) -> None:
    payload = (json.dumps(batch) + "\n").encode("utf-8")
    with socket.create_connection((host, port), timeout=timeout_s) as s:
        s.sendall(payload)

def push_with_backoff(host: str, port: int, samples: list[dict], batch_size: int = 500):
    backoff, max_backoff = 0.2, 5.0
    for i in range(0, len(samples), batch_size):
        batch = samples[i:i+batch_size]
        for attempt in range(8):
            try:
                send_batch(host, port, batch)
                break  # success
            except (OSError, TimeoutError) as e:
                if attempt == 7:
                    raise RuntimeError(f"remote write failed after retries: {e}")
                sleep = min(max_backoff, backoff * (2 ** attempt))
                time.sleep(sleep * (0.5 + random.random()))  # jitter avoids sync storms

# Usage example: push_with_backoff("127.0.0.1", 9009, samples)
```

[CODE: Python, simulating cardinality explosion by generating label combinations and estimating series count]

```python
# Implementing cardinality estimation by generating label combinations
from itertools import product

def estimate_series_count(label_values: dict[str, list[str]], hard_cap: int = 5_000_000) -> int:
    # Multiply cardinalities without materializing all combinations (safe for huge spaces)
    count = 1
    for k, vals in label_values.items():
        if not vals:
            raise ValueError(f"label '{k}' has no values")
        count *= len(vals)
        if count > hard_cap:
            return hard_cap
    return count

def demo():
    labels = {
        "region": ["us-east-1", "us-west-2"],
        "status": ["200", "500", "503"],
        "user_id": [str(i) for i in range(1_000_000)],  # cardinality bomb
    }
    print("estimated series:", estimate_series_count(labels))

# Usage example: demo()
```

[CODE: Bash, using tsdb tooling/PromQL to estimate active series and top label cardinalities]

```bash
#!/usr/bin/env bash
# Implementing active-series + top-cardinality label discovery via Prometheus HTTP API
set -euo pipefail
PROM_URL="${PROM_URL:-http://localhost:9090}"

query() {
  local q="$1"
  curl -fsS --get "$PROM_URL/api/v1/query" --data-urlencode "query=$q" | jq -r '.data.result'
}

# Active series (approx): count of series in head (Prometheus-specific metric)
query 'prometheus_tsdb_head_series'

# Top label pairs by series count (find cardinality bombs)
query 'topk(10, count by (label_name) (prometheus_tsdb_head_series_created_total))' || true

# Usage: PROM_URL=http://prom:9090 ./cardinality_audit.sh
```

---

## [FINAL SYNTHESIS CHALLENGE] Design an optimized distributed TSDB for a delivery marketplace

You’re building telemetry for a delivery marketplace (drivers, restaurants, customers). Requirements:

- 2M active devices
- metrics every 5s
- 99.9% ingest availability
- dashboards: last 15m at 5s resolution, last 90d at 5m
- multi-region (2 regions) active-active
- cost constraint: object storage for long retention

### [DECISION GAME] Multi-part

**Part 1: Labeling**
Which label is most dangerous?

A) `city`
B) `driver_id`
C) `vehicle_type`
D) `status_code`

**Pause and think.**

**Answer**: **B** (unbounded/high cardinality).

---

**Part 2: Sharding**
Choose a sharding approach:

1) time-only
2) series-hash only
3) series-hash + time blocks

**Pause and think.**

**Answer**: **3**.

---

**Part 3: Multi-region writes**
Choose write strategy:

1) write to one region, async replicate
2) write to nearest region, async replicate
3) sync write to both regions

**Pause and think.**

**Answer**: usually **2** for metrics.

---

**Part 4: Query path**
You need fast dashboards during incidents. Pick two:

- A) query-frontend with result cache
- B) pushdown aggregation
- C) disable compaction
- D) store only downsampled data

**Pause and think.**

**Answer**: **A + B**.

---

### [YOUR DESIGN WRITE-UP] Do it for real
Write a 10-bullet design:

- ingestion path
- backpressure strategy
- sharding key
- replication + ack semantics
- dedupe/out-of-order policy
- compaction strategy
- rollups/downsampling
- storage tiers
- caching layers
- failure handling (partition + node loss)

### [KEY INSIGHT]
> Optimization is not one trick—it’s aligning **data model, partitioning, and failure semantics** with your actual query and ingest patterns.

---

## [CLOSING] Return to the opening question

At the start, you had one change to survive the telemetry storm:

1) Increase replication factor
2) Increase shard count
3) Reduce series cardinality
4) Add a cache in front of queries

**Answer**: most often **3** (reduce cardinality) is the highest-leverage long-term fix. But during an active incident, the fastest stabilizers are usually **batching/backpressure** and **ingest throttling**. Replication and caches help, but they won’t save a cluster drowning in unbounded series.

---

### Appendix: Quick checklist (printable)

- [ ] Track active series and top-cardinality labels
- [ ] Batch writes and enforce backoff with jitter
- [ ] Set explicit write acknowledgment semantics
- [ ] Keep indexes hot; watch cache hit ratios
- [ ] Monitor compaction debt and WAL fsync latency
- [ ] Use rollups for long-range queries
- [ ] Plan for partitions: buffer + replay + dedupe
- [ ] Use tiered storage with immutable blocks + caches
- [ ] Test failure scenarios (node loss, AZ loss, object store slowdown)
- [ ] Document your “success means…” contracts for writes and reads
