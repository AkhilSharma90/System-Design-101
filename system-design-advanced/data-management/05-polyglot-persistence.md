---
slug: polyglot-persistence
title: Polyglot Persistence
readTime: 20 min
orderIndex: 5
premium: false
---






# Polyglot Persistence (in Distributed Systems): Choosing the Right “Kitchen” for Every Dish

> Audience: advanced distributed-systems engineers, architects, and senior developers
>
> Promise: by the end, you’ll be able to design a polyglot persistence architecture that survives partitions, handles multi-store transactions, and stays operable under real failure conditions.

---

## [CHALLENGE] Your app is a busy food court, not a single restaurant

Scenario: You’re building a global commerce platform.

- Product catalog needs flexible attributes (size charts, localized copy, variant options).
- Orders must be strongly consistent (no double-shipping).
- Search must be fast and typo-tolerant.
- Recommendations need graph traversals.
- Analytics needs columnar scans over billions of events.

Your team proposes: “Let’s use one database for everything.”

Pause and think: If you had to run a food court, would you force every vendor to cook every cuisine with one oven?

### [INTERACTIVE] Question
Which statement is more realistic?

A) One database can be optimal for all workloads, and operational simplicity always wins.

B) Different data shapes and access patterns benefit from specialized storage engines, but integration becomes the hard part.

Pause. Decide.

Answer: **B**.

### Analogy
A food court uses:
- A pizza oven (high throughput, predictable).
- A sushi bar (precision, freshness).
- A dessert stand (fast assembly).

Each station is optimized for a type of work. But the customer experience depends on coordination: shared seating, payment, cleanup.

### Key insight
> Polyglot persistence is not “use many databases.” It’s intentionally matching storage technologies to specific data models and workload constraints, then managing the distributed-systems consequences.

---

## [INVESTIGATE] What Polyglot Persistence actually means (and what it doesn’t)

### [CHALLENGE] Define it precisely
You hear “polyglot persistence” used to mean:
- microservices each with their own DB
- using Redis + Postgres
- event sourcing
- CQRS

Some of these overlap; some are orthogonal.

### [INTERACTIVE] Question (pause and think)
Is polyglot persistence primarily:

1) A data modeling strategy
2) A database vendor strategy
3) A distributed systems integration strategy

Pause. Pick the best.

Answer: **1 and 3**.

- It’s data modeling/workload driven (document vs relational vs graph vs column vs key-value).
- And it’s integration driven (consistency boundaries, replication, failure handling).

### [MISCONCEPTION]
> “Polyglot persistence means we’ll store the same data in multiple databases for safety.”

Not necessarily. That’s redundancy (which you might do), but polyglot persistence is about fit-for-purpose.

### Mental model
Think of each datastore as a contract:
- **Data model**: what shape is natural?
- **Query model**: what questions are cheap?
- **Consistency model**: what anomalies can happen?
- **Operational model**: how does it scale? how does it fail?

### Challenge question
Name one workload where a relational DB is suboptimal even if it can technically do the job.

(Keep it in mind; you’ll revisit later.)

---

## [CHALLENGE] The “one DB” trap in distributed environments

### Scenario
You pick a single relational database for everything. Then:
- Your search queries become slow `LIKE` queries.
- Your recommendation graph joins become multi-hop self-joins.
- Your analytics becomes expensive OLTP + OLAP contention.

You scale vertically. Then you shard. Then you discover cross-shard joins and cross-region latency.

### [INTERACTIVE] Question
Which is the most common failure mode of the “one DB for everything” approach at scale?

A) It can’t store enough data

B) It becomes a bottleneck for one workload and forces the entire system to compromise

C) It’s impossible to run in Kubernetes

Pause and think.

Answer: **B**.

### Restaurant analogy
If your burger kitchen also has to bake cakes and brew coffee, the burger line slows down because the oven is busy with pastries.

In distributed systems, shared resources become contention points:
- buffer pools
- I/O
- locks
- replication bandwidth
- failover complexity

### Key insight
> A single datastore often forces a global compromise on indexing, schema evolution, transaction isolation, and scaling strategy.

### Production insight
The first sign you’re past a DB’s “comfort zone” is usually **tail latency and operational coupling**, not disk usage:
- p99/p999 spikes during background maintenance (vacuum/compaction, checkpointing, rebalancing)
- lock contention and long-running transactions
- replication lag and failover time increasing
- “one workload” (search/analytics) forcing global config changes

### Challenge question
If you must keep one DB, what’s the first sign that you’re past its “comfort zone”? (Hint: it’s not disk usage.)

---

## [INVESTIGATE] Workload -> Datastore mapping (the “menu”)

### [CHALLENGE] Build your storage food court
You’re designing the platform. You can choose multiple datastores.

But you must justify each with:
- data model fit
- query patterns
- consistency needs
- scaling and failure behavior

### Mental model: “Three axes of fit”
1) **Shape**: relational rows vs documents vs edges vs columns vs key-value
2) **Questions**: point lookups vs aggregations vs full-text vs traversals
3) **Guarantees**: strong vs eventual, transactional vs best-effort

### Comparison table: common datastores in polyglot persistence

| Need / Workload | Typical store | Strength | Distributed-systems gotcha |
|---|---|---|---|
| Orders, payments, inventory | Relational (Postgres/MySQL) or NewSQL (CockroachDB, Spanner) | ACID transactions, constraints | Cross-region latency; distributed transactions cost; write amplification |
| Catalog with flexible attributes | Document (MongoDB) | schema flexibility, nested docs | multi-document transactions add coordination; write conflicts; secondary index costs |
| Session cache, counters, rate limits | Key-value (Redis) | low latency, simple ops | persistence tradeoffs; failover consistency; split-brain risk without quorum |
| Search, autocomplete | Search index (Elasticsearch/OpenSearch) | inverted index, relevance | refresh lag; eventual consistency vs SoR; mapping conflicts; backpressure |
| Recommendations, social graph | Graph DB (Neo4j) or graph layer | traversals, relationships | sharding graphs is hard; hot partitions; cross-shard traversals expensive |
| Analytics, reporting | Columnar warehouse (BigQuery/Snowflake/ClickHouse) | scans + compression | ingestion pipelines; data freshness; late-arriving events; cost controls |
| Time-series metrics | TSDB (Prometheus, InfluxDB, VictoriaMetrics) | time-window queries | high cardinality; retention policies; downsampling |
| Event log / integration | Kafka/Pulsar | durable ordered log | ordering is per-partition; “exactly-once” is nuanced; consumer lag and reprocessing |

### [DECISION GAME] Which statement is true?

1) “Search indices should be treated as the system of record.”
2) “Search indices are usually derived views and can be rebuilt.”

Pause and choose.

Answer: **2**.

Search indexes are typically materialized views derived from a source of truth (often a relational or document store).

### [MISCONCEPTION]
> “If we add Elasticsearch, we can stop worrying about database indexes.”

Elasticsearch solves a different problem (text relevance + inverted index). It doesn’t replace transactional indexing for OLTP.

### Challenge question
Pick one workload above and write down the single most important distributed-systems gotcha you’d expect.

---

## [CHALLENGE] Polyglot persistence forces you to draw boundaries

### Scenario
You adopt:
- Postgres for orders
- MongoDB for catalog
- Redis for sessions
- Elasticsearch for search
- Kafka for events

Now you must decide:
- what is the source of truth for each entity?
- what is derived?
- what is cached?
- what is rebuildable?

### [INTERACTIVE] Question
For each store, label it as one of:

- **SoR** (System of Record)
- **Derived View**
- **Cache**
- **Integration Log**

Pause and think:
- Postgres orders: ?
- Elasticsearch product search: ?
- Redis session tokens: ?
- Kafka events: ?

Answer (typical):
- Postgres orders: **SoR**
- Elasticsearch search: **Derived View**
- Redis sessions: **Cache** (or ephemeral state store)
- Kafka events: **Integration Log** (and sometimes SoR in event-sourced systems)

### Analogy
Think of:
- SoR = the kitchen’s official order ticket
- Derived View = the display screen showing “Order #42 in progress”
- Cache = the barista remembering your regular order
- Integration Log = the delivery dispatch log

### Key insight
> Polyglot persistence is fundamentally about explicitly declaring what can be rebuilt and what cannot.

### Challenge question
If Elasticsearch is a derived view, what’s your plan when it loses data or becomes inconsistent?

---

## [INVESTIGATE] Distributed data duplication—friend, foe, or inevitability?

### [CHALLENGE] You will duplicate data—now do it intentionally
In polyglot persistence, duplication happens because:
- read models differ from write models
- indexes and caches copy data
- services need local reads
- cross-store joins are too slow

Duplication is not “bad.” Undisciplined duplication is.

### [INTERACTIVE] Question
Which duplication is safer?

A) Copying data into a derived view that can be rebuilt from an immutable event log

B) Copying data manually via ad-hoc scripts and hoping it stays in sync

Answer: **A**.

### Mental model: “Rebuildability ladder”
From safest to scariest:

1) Rebuildable projections from an immutable log
2) Rebuildable indexes from a SoR snapshot + change stream
3) Caches with TTL + fallback
4) Dual-writes with no reconciliation

### [MISCONCEPTION]
> “Eventual consistency means it will eventually be consistent.”

Only if:
- updates are delivered reliably
- consumers are idempotent
- ordering assumptions are correct
- you have reconciliation for missed events

### Clarification: eventual consistency vs convergence
- **Eventual consistency** is a *property of a replicated system* under certain assumptions (no new updates, reliable delivery, etc.).
- **Eventual convergence** is what you usually need operationally: *all replicas/derived views end up with the same value* (or a defined conflict-resolved value).

In polyglot systems, you often don’t have “replicas” so much as **projections**. Your real requirement is typically:
- **eventual convergence of projections** to the SoR (or to the event log), plus
- **bounded staleness** (an SLO like “search reflects catalog changes within 2 minutes”).

### Challenge question
What’s the difference between “eventual consistency” and “eventual convergence”? Which one do you actually need?

---

## [CHALLENGE] The hardest part—cross-store consistency

### Scenario
A customer places an order.

You must:
1) Write order to Postgres (transactional)
2) Decrement inventory (maybe also Postgres)
3) Emit `OrderCreated` event to Kafka
4) Update Elasticsearch order history index
5) Update analytics warehouse

You want: no lost orders, no double decrements, no missing events.

### [DECISION GAME] Which statement is true?

1) “We can wrap all of these in a distributed transaction (2PC) and be done.”

2) “Distributed transactions across heterogeneous systems are possible but often operationally expensive and can reduce availability.”

Pause and choose.

Answer: **2**.

### CAP / partition reality check
Assume:
- networks can partition
- nodes can pause (GC, noisy neighbors)
- clocks drift

During a partition, you must choose between:
- **Consistency** (reject/timeout some operations) and
- **Availability** (accept operations that may later conflict)

Polyglot persistence makes this explicit because different stores make different CAP trade-offs.

### Analogy
Trying to make five separate restaurants commit to a single shared bill atomically is possible… if everyone stays online, responsive, and agrees on a protocol. If one kitchen’s printer jams, everyone waits.

### Key insight
> Polyglot persistence pushes you toward sagas, outbox/inbox, idempotency, and reconciliation rather than global ACID.

### Challenge question
List two reasons why 2PC can hurt availability in a partition.

---

## [INVESTIGATE] Patterns for polyglot persistence (with failure behavior)

### [CHALLENGE] Choose a consistency strategy you can operate
Here are the big patterns. You’ll pick based on failure tolerance and correctness needs.

---

## [PATTERN] Transactional Outbox (SoR -> Event Log)

Goal: Ensure that if you commit to the SoR, you also reliably publish an event.

How it works:
- In the same DB transaction as your business write, insert an outbox row.
- A background relay reads outbox rows and publishes to Kafka.
- Mark outbox rows as sent.

[IMAGE: Sequence diagram showing service writing to Postgres (business table + outbox table in same transaction), then an outbox relay publishing to Kafka, then consumers updating Elasticsearch/warehouse. Include failure points and retries.]

### [INTERACTIVE] Question (pause and think)
What happens if the service crashes right after committing the transaction but before publishing to Kafka?

Answer: The outbox row is committed, so the relay will publish later. No lost event.

### [MISCONCEPTION]
> “Outbox guarantees exactly-once delivery.”

Outbox helps prevent lost messages, but you still need:
- idempotent consumers
- deduplication keys
- careful handling of retries

### Production-grade clarifications (important)
1) **Polling vs push**: polling is simplest but adds DB load and latency. Alternatives:
   - `LISTEN/NOTIFY` (Postgres) to wake the relay (still keep polling as a safety net)
   - logical decoding / CDC (different pattern)
2) **Ordering**: outbox ordering is only meaningful within a single DB instance/partition. If you need per-aggregate ordering, include `aggregate_id` and a monotonic `aggregate_version`.
3) **Backpressure**: if Kafka is down, outbox grows. Plan:
   - retention/partitioning on outbox table
   - alert on “oldest unsent age”
   - throttle producers or shed non-critical writes
4) **Relay crash safety**: marking `sent_at` after publish is correct for at-least-once, but you must tolerate duplicates.

[CODE: SQL + pseudocode, transactional outbox]

```python
# Transactional Outbox (Postgres) + relay loop
# Notes:
# - At-least-once publish: duplicates are possible.
# - Use a stable event_id for deduplication downstream.
# - Consider batching and retry/backoff.

import json
import socket
import time
import uuid
from contextlib import closing

import psycopg2
import psycopg2.extras


def create_order_and_outbox(dsn: str, order_id: str, payload: dict) -> None:
    """Single DB transaction: business write + outbox insert."""
    event_id = str(uuid.uuid4())
    try:
        with psycopg2.connect(dsn) as conn:
            with conn.cursor() as cur:
                cur.execute(
                    "INSERT INTO orders(id, data) VALUES (%s,%s)",
                    (order_id, psycopg2.extras.Json(payload)),
                )
                cur.execute(
                    "INSERT INTO outbox(event_id, aggregate_id, event_type, event_json) "
                    "VALUES (%s,%s,%s,%s)",
                    (event_id, order_id, "OrderCreated", psycopg2.extras.Json(payload)),
                )
            # context manager commits on success
    except psycopg2.Error as e:
        raise RuntimeError(f"db_tx_failed: {e.pgerror}") from e


def relay_outbox_once(dsn: str, host: str, port: int) -> int:
    """At-least-once relay: lock rows, publish, mark sent.

    Failure behavior:
    - If publish fails mid-batch, transaction rolls back and rows remain unsent.
    - If publish succeeds but ack is lost, rows may be resent => duplicates.
    """
    with psycopg2.connect(dsn) as conn:
        with conn.cursor() as cur:
            cur.execute(
                "SELECT id, event_id, event_type, event_json "
                "FROM outbox "
                "WHERE sent_at IS NULL "
                "ORDER BY id "
                "LIMIT 100 "
                "FOR UPDATE SKIP LOCKED"
            )
            rows = cur.fetchall()
            if not rows:
                return 0

            # In production, publish to Kafka directly (client library) rather than raw sockets.
            # This socket is a stand-in for a durable broker.
            with closing(socket.create_connection((host, port), timeout=2)) as s:
                for outbox_id, event_id, event_type, event_json in rows:
                    envelope = {
                        "event_id": event_id,
                        "event_type": event_type,
                        "payload": event_json,
                    }
                    s.sendall((json.dumps(envelope) + "\n").encode("utf-8"))
                    cur.execute(
                        "UPDATE outbox SET sent_at = NOW() WHERE id = %s",
                        (outbox_id,),
                    )
        # commit
        return len(rows)


# Usage example: run relay in a loop (e.g., Deployment/sidecar)
# while True:
#     try:
#         relay_outbox_once(DSN, "kafka-bridge", 9000)
#     except Exception:
#         time.sleep(1.0)
#     time.sleep(0.2)
```

```sql
-- Transactional Outbox schema (Postgres)
-- Production notes:
-- - event_id is a stable deduplication key.
-- - Consider partitioning outbox by time for retention.

CREATE TABLE orders (
  id TEXT PRIMARY KEY,
  data JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE outbox (
  id BIGSERIAL PRIMARY KEY,
  event_id UUID NOT NULL UNIQUE,
  aggregate_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  event_json JSONB NOT NULL,
  sent_at TIMESTAMPTZ NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX outbox_unsent_idx ON outbox (id) WHERE sent_at IS NULL;
CREATE INDEX outbox_created_at_idx ON outbox (created_at);
```

### Key insight
> Outbox turns a fragile “dual-write” into a single transactional write plus an at-least-once publish.

### Challenge question
What’s the main downside of polling the outbox table? Name one improvement.

---

## [PATTERN] Change Data Capture (CDC) (DB log -> Event Log)

Goal: Publish changes by reading the DB’s replication log instead of writing outbox rows.

How it works:
- Debezium (or native CDC) tails WAL/binlog
- Emits change events into Kafka

[IMAGE: Diagram showing Postgres WAL -> Debezium -> Kafka topics -> consumers -> Elasticsearch/warehouse.]

### [INTERACTIVE] Question
Which is more coupled to your application code?

A) Outbox
B) CDC

Answer: **Outbox** is more explicit in app code; CDC is more infrastructure-driven.

### Trade-offs
- CDC can capture all changes (including ones not emitted by app)
- But schema evolution and event semantics can be tricky

### [MISCONCEPTION]
> “CDC events are domain events.”

CDC produces data change events (row-level). Domain events encode business meaning.

### Production insight: CDC risk areas
- **PII leakage**: CDC sees *all columns*. You must filter/transform before publishing.
- **Schema drift**: DDL changes can break connectors or downstream consumers.
- **Reprocessing**: connector restarts can replay; consumers must be idempotent.
- **Ordering**: ordering is per table/partition; cross-table invariants require careful design.

### Challenge question
If you rely on CDC, how do you prevent leaking sensitive columns into your event stream?

---

## [PATTERN] CQRS + Materialized Views (Write model vs Read model)

Goal: Optimize reads without compromising writes.

How it works:
- Write model: normalized, transactional
- Read model: denormalized, query-optimized (Elasticsearch, Redis, document store)
- Updates propagate asynchronously

[IMAGE: CQRS diagram with write side (commands) -> event log -> projections -> multiple read stores.]

### [INTERACTIVE] Question
Your customer updates their address. For 30 seconds, the order history page still shows the old address. Is that a bug?

Answer: It depends on your consistency SLO and product requirements. In CQRS, staleness is expected unless you add read-your-writes mechanisms.

### Key insight
> CQRS makes staleness a first-class design parameter.

### Production techniques for read-your-writes
- **Read token**: return a version/offset (e.g., Kafka offset, DB commit LSN) and have reads wait until projections catch up.
- **Session routing**: route reads to the same region/leader that accepted the write.
- **Fallback read**: for a short window, read from SoR for the specific entity.

### Challenge question
Name one technique to provide “read-your-writes” in a CQRS system.

---

## [PATTERN] Saga (Orchestration vs Choreography)

Goal: Coordinate multi-step workflows across services/stores without global transactions.

Two styles:
- Orchestrated saga: a coordinator tells participants what to do
- Choreographed saga: participants react to events

[IMAGE: Two side-by-side diagrams comparing orchestrated vs choreographed sagas for order creation and inventory reservation.]

### [DECISION GAME]
Which statement is true?

1) “Sagas guarantee atomicity.”
2) “Sagas guarantee eventual completion or compensation, but intermediate states are visible.”

Answer: **2**.

### Failure scenarios to design for
- duplicate messages
- partial completion
- compensation failure
- long-running timeouts

### [MISCONCEPTION]
> “Compensation is just rollback.”

Compensation is a new business action that attempts to counteract effects. It may not be perfect (e.g., refund vs undo shipment).

### Production insight
Sagas require **state machines** and **operational tooling**:
- explicit saga state persisted durably
- timeouts and retries with jitter
- DLQ + manual resolution playbooks
- idempotent handlers for every step

### Challenge question
If compensation fails, what’s your operational plan? (Hint: dead-letter queues + manual workflows.)

---

## [CHALLENGE] Failure scenarios you must game out (polyglot edition)

### Scenario
Your order write succeeds in Postgres. Kafka is temporarily unavailable. Elasticsearch cluster is green but lagging. Redis fails over.

What does the user see? What do you guarantee?

### [DECISION GAME] Which statement is true?

A) If Postgres commit succeeded, then all other stores must reflect it immediately.

B) If Postgres commit succeeded, other stores can lag, but you must ensure they eventually converge or provide compensating logic.

Answer: **B**.

### Failure matrix (think like an SRE)

| Component | Failure | Symptom | Correctness risk | Typical mitigation |
|---|---|---|---|---|
| Kafka | broker outage / partition unavailable | publish fails, consumer lag | lost events if dual-write; staleness | outbox/CDC, retries, backpressure, alert on lag |
| Elasticsearch | index lag / red cluster | stale search results | user confusion; missing items | rebuild index, reindex pipeline, fallback queries, deterministic IDs |
| Redis | failover / eviction | cache misses, session loss | auth issues, rate limit resets | persistent sessions, token strategy, warmup, avoid SoR-in-cache |
| Warehouse | ingestion delay | stale analytics | reporting inaccuracies | watermarking, late-arriving handling, backfill jobs |
| Network | partition | timeouts, split-brain risks | double writes, divergent state | quorum, fencing tokens, idempotency, circuit breakers |

### Mental model: “Correctness tiers”
Define tiers per feature:
- Tier 0: must be correct (payments)
- Tier 1: should be correct soon (inventory display)
- Tier 2: best-effort (recommendations)

Polyglot persistence works when you assign tiers and design accordingly.

### Challenge question
Pick one feature in your system that should be Tier 0 and one that can be Tier 2. What stores would you use for each?

---

## [INVESTIGATE] Consistency models across stores—what you can and can’t promise

### [CHALLENGE] Users ask for “strong consistency,” but across what boundary?

Within a single datastore, you might get:
- serializable transactions
- linearizable reads

Across multiple stores, you usually get:
- eventual consistency
- causal-ish behavior if you design for it

### Clarification: consistency terms (avoid ambiguity)
- **Linearizable**: reads reflect the latest completed write globally (single-copy illusion).
- **Serializable**: transactions behave as if executed in some serial order.
- **Read-your-writes**: a client sees its own writes.
- **Causal consistency**: causally related writes are observed in order.

In polyglot systems, you typically can only guarantee these **within a boundary** (one DB, one partition, one service) unless you add coordination.

### [INTERACTIVE] Matching exercise
Match the guarantee to the mechanism.

Guarantees:
1) Read-your-writes
2) Monotonic reads
3) Exactly-once processing (practical)
4) No lost updates

Mechanisms:
A) Idempotent consumers + dedup keys + transactional offsets
B) Session stickiness + version vectors / tokens
C) Optimistic concurrency control (compare-and-set)
D) Client-side read tokens + routing to leader/primary

Suggested answers: **1->D, 2->B, 3->A, 4->C**

### [MISCONCEPTION]
> “Kafka gives exactly-once, so my whole pipeline is exactly-once.”

Exactly-once is end-to-end. Any sink without idempotency breaks it.

### Key insight
> Polyglot persistence is a negotiation between user-visible guarantees and cross-store physics (latency, partitions, retries).

### Challenge question
What’s the difference between idempotency and deduplication? Why do you often need both?

---

## [CHALLENGE] Multi-region polyglot persistence (latency vs correctness)

### Scenario
You expand to 3 regions: us-east, eu-west, ap-south.

You want:
- low-latency reads everywhere
- global inventory correctness
- consistent order IDs

### [INTERACTIVE] Question
Which is usually hardest to make both low-latency and strongly consistent globally?

A) Product catalog reads
B) Inventory decrement / reservation
C) Search autocomplete

Answer: **B**.

### Delivery-service analogy
If three warehouses all accept reservations instantly, you risk overselling unless they coordinate. Coordination across oceans costs time.

### Design options (with trade-offs)

| Option | How it works | Pros | Cons |
|---|---|---|---|
| Single-writer region | all Tier 0 writes go to one region | simplest correctness | higher latency for distant users; regional dependency |
| Global consensus DB (Spanner/CRDB) | distributed transactions via quorum | strong consistency | higher write latency; cost; careful schema and hotspots |
| Regional writes + async reconciliation | accept locally, reconcile later | low latency | oversells; complex conflict resolution; user-visible corrections |
| Reservation tokens | allocate inventory buckets per region | fast local reservations | complexity; rebalancing tokens; risk of stranded capacity |

### Failure scenario to explicitly design for
**Inter-region partition**:
- If you choose global consensus, writes may block in minority partitions (CP behavior).
- If you choose regional writes, you remain available (AP-ish) but must resolve conflicts.

### Challenge question
If you choose reservation tokens per region, what happens during a sudden demand spike in one region?

---

## [INVESTIGATE] Schema evolution and contracts across stores

### [CHALLENGE] One field change breaks five systems
You add a field `preferredDeliveryWindow`.

- Postgres: add column
- Kafka: update event schema
- Elasticsearch: update mapping
- Warehouse: update ingestion
- Cache: update serialization

### [INTERACTIVE] Question
Which is the safest evolution strategy?

A) Rename fields in-place and deploy everything at once

B) Add new fields in a backward-compatible way; support both until all consumers migrate

Answer: **B**.

### Mental model: “Schema as a distributed contract”
Schema isn’t just DDL. It’s:
- event schemas
- index mappings
- serialization formats
- query expectations

### [MISCONCEPTION]
> “NoSQL means no schema.”

It means schema-on-read or flexible schema, but you still have contracts.

[CODE: Protobuf schema evolution example]

```javascript
// Protobuf evolution-safe decoding (Node.js)
// Concept: add optional fields; old consumers ignore unknown fields.
const protobuf = require("protobufjs");

async function decodeOrderEvent(buf) {
  try {
    const root = await protobuf.load("./schemas/order.proto");
    const OrderEvent = root.lookupType("events.OrderEvent");
    const msg = OrderEvent.decode(buf);

    // Backward/forward compatibility: field may be absent.
    const preferred = msg.preferredDeliveryWindow || null;
    return { orderId: msg.orderId, status: msg.status, preferredDeliveryWindow: preferred };
  } catch (err) {
    // Production: route to DLQ / alert; don't crash the consumer loop.
    throw new Error(`protobuf_decode_failed: ${err.message}`);
  }
}

// Usage example: const parsed = await decodeOrderEvent(message.value)
```

### Production insight: Elasticsearch mapping pain
Mapping changes can be painful because:
- many field type changes require **reindexing** (new index + rehydrate)
- dynamic mappings can “lock in” the wrong type early
- partial failures (mapping rejects) can silently drop documents unless monitored

### Challenge question
What’s one reason Elasticsearch mapping changes can be painful compared to relational schema changes?

---

## [CHALLENGE] Observability—debugging across stores is a distributed tracing problem

### Scenario
A user reports: “My order doesn’t show up in search.”

Possible causes:
- outbox relay lag
- Kafka consumer stuck
- Elasticsearch indexing error
- mapping rejection
- event schema mismatch

### [INTERACTIVE] Exercise: Build a diagnostic checklist
What are the first 5 metrics/logs you’d check?

Suggested checklist:
1) Outbox backlog size and oldest age
2) Kafka topic lag for the search indexer consumer group
3) Consumer error rate / DLQ volume
4) Elasticsearch indexing throughput + rejected documents
5) Trace a single order ID through logs (correlation ID)

[IMAGE: Observability dashboard mock showing outbox lag, Kafka consumer lag, DLQ rate, Elasticsearch indexing errors, and end-to-end trace timeline.]

### Key insight
> Polyglot persistence demands end-to-end observability, not per-database monitoring.

### Production insight: correlation keys
Propagate:
- `order_id` (business key)
- `event_id` (dedup key)
- `trace_id` (distributed tracing)

This lets you connect:
- Postgres transaction -> outbox row -> Kafka message -> Elasticsearch document.

### Challenge question
What correlation key would you propagate to connect a Postgres transaction to an Elasticsearch document?

---

## [INVESTIGATE] Operational complexity—how to keep the food court clean

### [CHALLENGE] More stores = more operational surfaces
Each datastore adds:
- backups/restores
- upgrades
- capacity planning
- incident runbooks
- access control
- cost management

### [DECISION GAME]
Which statement is true?

1) “Polyglot persistence always increases complexity.”
2) “Polyglot persistence increases operational complexity, but can reduce application complexity by fitting workloads.”

Answer: **2**.

### Analogy
A food court has more vendors (complexity), but each vendor has a simpler menu and better throughput for its niche.

### Key insight
> Polyglot persistence is a complexity trade: you move complexity from query logic and performance tuning into integration and operations.

### Production checklist to standardize across stores
- IAM and least privilege
- encryption at rest + in transit
- backup/restore drills (RTO/RPO)
- SLOs and alerting (including lag/backlog)
- data retention and deletion policies

### Challenge question
Name one operational capability you must standardize across all stores (hint: backups, IAM, encryption, or SLOs).

---

## [CHALLENGE] Anti-patterns (and how they fail)

### [MISCONCEPTION/ANTI-PATTERN] Dual writes without a log
You write to Postgres and Elasticsearch in the same request handler.

Failure:
- Postgres commit succeeds
- Elasticsearch write times out
- request retries
- now you might have duplicates or missing index entries

Fix: outbox/CDC + idempotent indexing.

### [MISCONCEPTION/ANTI-PATTERN] Treating caches as SoR
You store the shopping cart only in Redis.

Failure:
- Redis failover loses keys
- carts disappear

Fix: persist carts (or events) in a durable store; use Redis as cache.

### [MISCONCEPTION/ANTI-PATTERN] Cross-store joins at request time
You fetch user profile from MongoDB and orders from Postgres and recommendations from graph DB on every request.

Failure:
- tail latency explodes
- partial failures become user-visible

Fix: precompute read models; degrade gracefully.

### Production insight: “synchronous fan-out” is a reliability killer
If a request depends on N remote calls, availability roughly multiplies:
- if each dependency is 99.9% available, 5 dependencies yields ~99.5% best-case (ignoring latency).

### Challenge question
Which anti-pattern is most tempting for teams early in a project, and why?

---

## [INVESTIGATE] Real-world usage patterns (what companies actually do)

### [CHALLENGE] “What does polyglot look like in production?”

Common blueprint:
- SoR: relational or NewSQL
- Cache: Redis/Memcached
- Search: Elasticsearch/OpenSearch
- Events: Kafka/Pulsar
- Analytics: warehouse/lakehouse

Why it works:
- clear ownership boundaries
- derived stores rebuildable
- event pipelines provide integration

### [INTERACTIVE] Question
If you had to remove one component to simplify, which would you remove first?

A) Kafka
B) Redis
C) Elasticsearch

Answer: It depends, but often Redis is the easiest to remove if you can tolerate latency and DB load. Kafka and Elasticsearch may be core to integration and search.

### Key insight
> Mature polyglot architectures are opinionated about what is optional and what is foundational.

### Challenge question
In your system, which store is “optional”? What’s the fallback path if it’s down?

---

## [CHALLENGE] Design exercise—build a polyglot persistence plan

### Scenario
You’re designing “BeanCart,” a global coffee subscription service.

Features:
1) Browse coffee catalog with rich filters and full-text search
2) Subscribe (recurring orders) and manage payment methods
3) Track shipments
4) Personalized recommendations
5) Real-time dashboard for operations (orders per minute)

Constraints:
- 99.95% availability for checkout
- Search can be stale up to 2 minutes
- Recommendations can be stale up to 1 day
- Must handle regional outages

### [INTERACTIVE] Worksheet (pause and think)
Fill this table (mentally or on paper):

| Feature | SoR | Derived stores | Consistency target | Failure behavior |
|---|---|---|---|---|
| Catalog | ? | ? | ? | ? |
| Checkout | ? | ? | ? | ? |
| Shipment tracking | ? | ? | ? | ? |
| Recommendations | ? | ? | ? | ? |
| Ops dashboard | ? | ? | ? | ? |

### Progressive reveal: one plausible solution

| Feature | SoR | Derived stores | Consistency target | Failure behavior |
|---|---|---|---|---|
| Catalog | Document DB or relational | Elasticsearch for search; CDN cache | eventual (<=2 min) | fallback to basic browse from SoR |
| Checkout | Relational/NewSQL | Kafka events; cache for idempotency keys | strong in SoR; async elsewhere | if Kafka down, queue via outbox; never lose order |
| Shipment tracking | Relational or event log | read model in document store | eventual minutes | show last known state; retry updates |
| Recommendations | Event log + warehouse | feature store / vector DB | stale allowed (<=1 day) | degrade to popular items |
| Ops dashboard | Event log | time-series DB | seconds | show delayed metrics with watermark |

### Key insight
> The “right” polyglot design is driven by feature SLOs, not by technology fashion.

### Challenge question
Where would you place the outbox: in the checkout service DB, or in a shared integration DB? Why?

---

## [INVESTIGATE] Testing polyglot persistence (the part teams skip)

### [CHALLENGE] Your unit tests pass, but production fails
Polyglot persistence fails in the seams:
- retries
- duplicates
- reordering
- partial outages
- schema drift

### [INTERACTIVE] Question
Which test catches the most real polyglot bugs?

A) Unit tests of repository classes
B) End-to-end tests with fault injection (kill consumers, delay Kafka, drop network)

Answer: **B**.

[CODE: Python, fault injection test outline]

```python
# Fault-injection E2E test (kill/restart consumer)
import os
import signal
import subprocess
import time
from dataclasses import dataclass


@dataclass(frozen=True)
class Proc:
    name: str
    p: subprocess.Popen


def run(cmd: list[str], name: str) -> Proc:
    p = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
    if p.poll() is not None:
        raise RuntimeError(f"{name}_failed_to_start")
    return Proc(name, p)


def test_eventual_convergence() -> None:
    indexer = run(["./bin/search-indexer"], "indexer")
    try:
        subprocess.check_call(["./bin/place-order", "--count", "5"], timeout=10)
        os.kill(indexer.p.pid, signal.SIGKILL)  # fault: consumer dies
        subprocess.check_call(["./bin/place-order", "--count", "5"], timeout=10)
        indexer = run(["./bin/search-indexer"], "indexer")  # restart
        time.sleep(5)  # allow catch-up; in prod use lag-based wait
        subprocess.check_call(["./bin/assert-search", "--min-orders", "10"], timeout=10)
    finally:
        if indexer.p.poll() is None:
            indexer.p.terminate()

# Usage example: run under pytest in CI with docker-compose dependencies
```

### Key insight
> You don’t “believe” in eventual consistency—you test it.

### Challenge question
What’s one invariant you can assert even under eventual consistency? (Example: “No paid order is missing from the SoR.”)

---

## [CHALLENGE] Security and compliance across multiple datastores

### Scenario
You store PII in Postgres, but you also index user names in Elasticsearch and copy events to a warehouse.

Now GDPR deletion requests arrive.

### [INTERACTIVE] Question
What’s the correct interpretation of “delete user data” in polyglot persistence?

A) Delete from the SoR only
B) Delete from all derived stores and logs, or apply compliant anonymization strategies

Answer: **B**.

### Practical approaches
- Encrypt + forget: encrypt PII with per-user key; deleting the key makes data unreadable
- Tombstones: propagate deletion events to derived stores
- Scoped indexing: avoid indexing sensitive fields

### [MISCONCEPTION]
> “We can’t delete from Kafka, so we can’t be compliant.”

You can design to avoid storing raw PII in immutable logs, or use encryption strategies.

### Production insight: “encrypt + forget” trade-off
New operational risk: **key management becomes a Tier 0 dependency**.
- key loss = permanent data loss
- key service outage can block reads
- you need rotation, auditing, and break-glass procedures

### Challenge question
If you use “encrypt + forget,” what new operational risk do you introduce?

---

## [MENTAL MODEL] Polyglot persistence as a pipeline of truth

### [CHALLENGE] Stop thinking in databases; think in truth flows
A durable system often looks like:

1) Truth write happens in a transactional boundary (SoR)
2) Truth is emitted reliably (outbox/CDC)
3) Truth is projected into read-optimized stores
4) Truth is observed via metrics/traces
5) Truth is repaired via replay/rebuild/reconciliation

[IMAGE: “Truth flow” pipeline diagram: SoR -> Outbox/CDC -> Event Log -> Projections (Search, Cache, Warehouse) with a “Rebuild/Reconcile” loop.]

### Key insight
> Polyglot persistence is successful when you can answer: Where is the truth? How does it propagate? How do we repair it?

### Failure scenario: event log unavailable for 30 minutes
What must still function?
- Tier 0 writes to SoR (checkout) should continue.
- Derived views will lag; UI must tolerate staleness.
- Outbox/CDC backlog grows; you must have capacity and alerts.

What must not happen?
- synchronous dependencies on Kafka/ES for Tier 0 paths.

### Challenge question
If your event log is unavailable for 30 minutes, what parts of your system must still function? How?

---

## [FINAL SYNTHESIS] The Polyglot Persistence Incident Game

### [CHALLENGE] Scenario: “Search is wrong, checkout is slow, and metrics are missing”
It’s Black Friday.

Symptoms:
- Checkout latency increased from 200ms to 2s
- Some users can’t find products via search
- Ops dashboard shows 0 orders/min (but orders are coming in)

Constraints:
- You cannot take the system down
- You must preserve correctness for paid orders

### Your tasks (pause and think)
1) Identify the most likely root causes (at least two) that could explain all symptoms.
2) Decide what to do in the next 10 minutes.
3) Decide what to do in the next 2 hours.
4) Propose one architectural change to prevent recurrence.

---

### Progressive reveal: A plausible incident response

1) Likely root causes
- Kafka consumer lag or outage: search indexer and metrics pipeline both depend on Kafka -> search stale and dashboard empty
- Postgres under heavy load: checkout slow; possibly because cache (Redis) degraded and more reads hit Postgres
- Elasticsearch cluster under pressure: indexing/backpressure causes missing search results

2) Next 10 minutes
- Protect Tier 0: ensure checkout writes to SoR remain healthy
  - shed non-critical queries, disable expensive read paths
  - ensure connection pools aren’t exhausted; apply admission control
  - enable circuit breakers to stop synchronous calls to Elasticsearch/warehouse
- Check outbox backlog and Kafka availability
  - if Kafka down: ensure outbox continues accumulating; monitor DB bloat and oldest age
- For search: switch UI to “browse from catalog SoR” fallback for critical flows

3) Next 2 hours
- Restore Kafka throughput / consumer groups
- Reindex missed events (replay from outbox or Kafka offsets)
- Drain DLQs; fix mapping/schema errors
- Add dashboards for outbox age, consumer lag, indexing rejection rates

4) Architectural change
- Make search and metrics strictly derived with clear fallbacks
- Add replay tooling and automated reconciliation jobs
- Implement idempotent consumers with deterministic document IDs in Elasticsearch

### Key insight
> Incident response is easier when your system design already encodes: tiers, fallbacks, and replay.

### Closing challenge
If you could only add one capability to improve a polyglot system’s reliability, what would it be?

A) Better caching
B) Better tracing
C) Deterministic replay + reconciliation
D) More replicas

Suggested answer: **C**. Caching, tracing, and replicas help, but replay/reconciliation is what turns eventual consistency from hope into engineering.

---

## Appendix: Quick reference checklist

### Polyglot persistence readiness checklist
- [ ] Each entity has a declared System of Record
- [ ] Derived stores are rebuildable
- [ ] Cross-store updates use outbox or CDC, not dual writes
- [ ] Consumers are idempotent and handle duplicates
- [ ] You have DLQs and operational playbooks
- [ ] You define consistency SLOs per feature
- [ ] You have replay/reindex tooling
- [ ] You test with fault injection
- [ ] You have end-to-end tracing and lag dashboards
- [ ] You have a data deletion/anonymization strategy

---

[CODE: Suggested deterministic ID strategy for Elasticsearch]

```javascript
// Deterministic, idempotent Elasticsearch upsert (Node.js)
// Concept: use order_id as _id so retries/duplicates converge to one document.
const https = require("https");

function esUpsert({ baseUrl, index, orderId, doc, ifPrimaryTerm, ifSeqNo }) {
  return new Promise((resolve, reject) => {
    const url = new URL(`${baseUrl}/${encodeURIComponent(index)}/_update/${encodeURIComponent(orderId)}`);
    if (ifPrimaryTerm != null && ifSeqNo != null) {
      url.searchParams.set("if_primary_term", String(ifPrimaryTerm));
      url.searchParams.set("if_seq_no", String(ifSeqNo));
    }
    const body = JSON.stringify({ doc, doc_as_upsert: true });
    const req = https.request(
      url,
      { method: "POST", headers: { "content-type": "application/json" } },
      (res) => {
        let data = "";
        res.on("data", (c) => (data += c));
        res.on("end", () =>
          res.statusCode >= 200 && res.statusCode < 300
            ? resolve(JSON.parse(data || "{}"))
            : reject(new Error(`es_upsert_failed: ${res.statusCode} ${data}`))
        );
      }
    );
    req.on("error", (e) => reject(new Error(`es_network_error: ${e.message}`)));
    req.end(body);
  });
}

// Usage example: await esUpsert({ baseUrl: "https://es:9200", index: "orders", orderId, doc })
```

[IMAGE: Matching exercise card set]
A printable set of cards: workloads (search, OLTP, graph traversals, analytics), datastores, and failure modes. Learners match them.
