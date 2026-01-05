---
slug: kappa-archiecture-vs-lambda-architecture
title: Kappa Architecture vs Lambda Architecture
readTime: 20 min
orderIndex: 6
premium: false
---




# Kappa Architecture vs Lambda Architecture

> Audience: advanced distributed-systems engineers and architects building streaming + analytics platforms at scale.
>
> Goal: understand how Lambda and Kappa behave in real distributed environments (failure modes, trade-offs, operational realities) and practice choosing between them.

---

## [CHALLENGE] Your analytics platform is melting down

It is Monday 9:07 AM. Your product team wants:

- Real-time dashboards (seconds latency)
- Accurate historical reports (correctness over months)
- Ability to reprocess when business logic changes
- Cost control

Your current system:

- A Kafka cluster ingesting events from mobile + web
- A stream processor doing near-real-time aggregations
- A batch pipeline producing "official" daily numbers

And now:

- A bug in the stream job inflated revenue for 3 hours
- A schema change broke batch parsing
- The CEO wants a single "source of truth" by tomorrow

Pause and think:

1) Do you fix this by adding more batch reliability? Or by making streaming replayable enough to replace batch?
2) If you had to pick one: would you rather maintain two pipelines (stream + batch) or one pipeline (stream-only) but with stricter operational discipline?

We are about to compare two architectural patterns designed to answer exactly this:

- **Lambda Architecture**: two paths (speed + batch) merged at serving time
- **Kappa Architecture**: one path (stream) + replay for "batch-like" recomputation

---

## [MENTAL MODEL] Two kitchens vs one kitchen with a freezer

Imagine you run a restaurant that must serve:

- Instant orders (walk-ins)
- Perfect banquet meals (planned events)

**Lambda = two kitchens**
- Fast kitchen: makes quick meals now (stream layer)
- Slow kitchen: makes perfect meals later (batch layer)
- Waiter merges plates from both kitchens (serving layer)

**Kappa = one kitchen + freezer + strict recipes**
- One kitchen does everything (stream processing)
- If recipe changes, you take ingredients from the freezer (event log replay) and redo meals

Key insight:

> Lambda optimizes for correctness by redundancy.
> Kappa optimizes for simplicity by replay.

---

## [DEEP DIVE] What problem are we really solving?

### [CHALLENGE] Scenario
You have an immutable event log (Kafka/Pulsar/Kinesis) and you want:

- Low latency insights
- Correct historical views
- Evolvable logic

### Interactive question (pause and think)
Which is harder in distributed systems?

A) Processing data once
B) Processing data continuously
C) Processing data continuously and being able to recompute history when logic changes

Think about it. Most teams can do A or B with enough time. C is where architecture matters.

### Explanation
Recomputation needs:

- Durable, replayable source of truth
- Deterministic processing (or at least controlled nondeterminism)
- Versioned schemas and logic
- A way to publish corrected outputs without breaking consumers

### Real-world parallel
It is like accounting:

- You record every transaction (event log)
- You produce a running balance (real-time)
- If tax rules change, you must recompute past statements

Key insight:

> Both Lambda and Kappa assume you keep the raw events long enough to replay or recompute.

Challenge question:
If you only retain raw events for 7 days, what does "recompute history" actually mean in your org?

---

## [OVERVIEW] Lambda in one picture

[IMAGE: Diagram of Lambda Architecture. Left: event ingestion into both a Batch Layer (S3/HDFS + batch compute) and a Speed Layer (stream processor). Both produce views to a Serving Layer (query engine) that merges results. Include labels: recomputation via batch, low latency via speed, and merge logic complexity.]

### [CHALLENGE] Scenario
Your pipeline writes:

- Batch views: computed from all historical data nightly
- Speed views: computed from recent data in real time
- Serving layer: merges "batch up to T" + "speed after T"

### Interactive question
What is the core promise of Lambda?

1) Low latency with no correctness compromise
2) Correctness by having a batch "truth" and a speed "approximation"
3) No need for an event log

Pause and think.

### Answer (progressive reveal)
Correct: 2.

Lambda's idea: batch is authoritative; speed is temporary until batch catches up.

### Distributed systems mechanics
- Batch layer is often Spark on S3/HDFS, producing immutable batch views.
- Speed layer is Flink/Storm/Spark Streaming producing incremental updates.
- Serving layer might be Druid/Pinot/Cassandra/Elastic/ClickHouse, merging results.

Key insight:

> Lambda pushes complexity into the merge: two pipelines, two semantics, one query result.

### Production clarification: the "T" boundary is a contract
In production, the boundary **T** is not a vague timestamp; it is a **data contract** between layers.

Common ways to define T:
- **Ingestion-time boundary**: batch covers all events ingested up to a Kafka offset / lake partition; speed covers the rest.
- **Event-time boundary**: batch covers event-time up to watermark W; speed covers after W.

Failure mode:
- If batch uses ingestion-time partitions but speed uses event-time windows, you can double-count or miss late events.

Production practice:
- Define T in terms of **input completeness** (e.g., "all partitions up to date X") and publish it as metadata.

Challenge question:
Where do you put the "T" boundary in practice, and how do you ensure batch and speed agree on it?

---

## [OVERVIEW] Kappa in one picture

[IMAGE: Diagram of Kappa Architecture. Event ingestion into a durable log (Kafka). Single Stream Processing Layer consumes log and writes materialized views to serving stores. Reprocessing shown by spinning up a new consumer group reading from earliest offset and writing to a new versioned output, then switching traffic.]

### [CHALLENGE] Scenario
You decide: "No batch pipeline. Everything is a stream."

- A single stream processor reads the event log
- It writes materialized views
- If logic changes, you replay from the beginning (or from a checkpoint)

### Interactive question
What is the core promise of Kappa?

A) Exactly-once processing is guaranteed
B) Simpler operations by having one pipeline
C) No need to store data long-term

Pause and think.

### Answer (progressive reveal)
Correct: B.

Kappa does not magically guarantee exactly-once. It reduces architectural surface area: one processing model.

### Distributed systems mechanics
- Event log must be durable and retained long enough.
- Stream processor must support stateful processing, checkpointing, and reprocessing with versioned outputs.

Key insight:

> Kappa trades batch complexity for log retention + replay discipline.

### Production clarification: replay is a distributed migration
A replay is not just "reset offsets":
- You need **output isolation** (new topic/table/index)
- You need **throttling** to protect sinks
- You need **validation** and an **atomic cutover**

Challenge question:
What is your maximum acceptable replay time (RTO for "logic correction")?

---

## [PUZZLE] Matching exercise: Components and responsibilities

Match the component to its primary responsibility.

| Component | Responsibility options |
|---|---|
| Batch layer (Lambda) | (i) low-latency incremental updates, (ii) recompute full history, (iii) merge results |
| Speed layer (Lambda) | (i) low-latency incremental updates, (ii) recompute full history, (iii) merge results |
| Serving layer (Lambda) | (i) low-latency incremental updates, (ii) recompute full history, (iii) merge results |
| Log + stream processor (Kappa) | (i) low-latency + recompute via replay, (ii) merge batch+speed, (iii) store immutable views |

Pause and think.

Answer:
- Batch layer -> (ii)
- Speed layer -> (i)
- Serving layer -> (iii)
- Log + stream processor (Kappa) -> (i)

Key insight:

> Lambda decomposes responsibilities into separate systems; Kappa composes them into one processing model.

Challenge question:
Where does "data quality validation" belong in each architecture?

---

## [DEEP DIVE] The real distributed challenge: time, ordering, determinism

### [CHALLENGE] Scenario
Events arrive out of order:

- Mobile devices buffer events offline
- Network partitions reorder messages
- Producers retry and duplicate

Your business logic:

- Count unique purchases per user per day
- Compute session length
- Detect fraud within 2 minutes

### Decision game: Which statement is true?

1) Batch pipelines naturally fix out-of-order data; streaming cannot.
2) Streaming pipelines can handle out-of-order data with event-time + watermarks.
3) Kappa requires strictly ordered events.

Pause and think.

### Answer
Correct: 2.

Batch often appears to "fix" ordering because it processes a static dataset, but it still needs correct event-time logic. Modern stream processors (Flink, Beam) model event-time explicitly.

### Production insight: ordering guarantees are per-partition
Kafka provides ordering **per partition**, not globally.
- If your keying changes between versions (e.g., user_id -> session_id), you can change ordering and state distribution.
- Replays can produce different results if your logic depends on cross-partition ordering.

Key insight:

> The architecture choice does not remove event-time complexity; it changes where you pay for it.

Challenge question:
What watermark strategy would you choose if 99.9% of events arrive within 5 minutes but 0.1% arrive within 48 hours?

---

## [MISCONCEPTION] "Lambda is obsolete because streaming is mature"

Misconception:
"Now that we have Flink/Beam and exactly-once, everyone should do Kappa."

Reality: Lambda still appears when

- Stream processing is not trusted as the only correctness path
- You need heavy historical computations that are cheaper in batch
- Retention in the event log is limited (cost or regulation)
- You have a legacy data lake with batch governance

Key insight:

> Maturity of streaming reduces the need for Lambda, but does not eliminate the economic and organizational reasons for it.

Challenge question:
If your Kafka retention is 7 days, can you do true Kappa for "recompute last 6 months"? What would you need to change?

---

## [DEEP DIVE] Correctness semantics: exactly-once, at-least-once, effectively-once

### [CHALLENGE] Scenario
You compute revenue in a materialized view store.

- Processor crashes after writing to the DB but before committing Kafka offsets.
- On restart, it reprocesses some events.

### Interactive question
What happens if your sink is not idempotent?

A) Nothing; Kafka prevents duplicates
B) Revenue is overcounted
C) The job will not restart

Pause and think.

### Answer
Correct: B.

Kafka provides ordering per partition and durable offsets, but not transactional coupling with arbitrary sinks.

### Mental model
Think of Kafka offsets as "I have read up to this page in the ledger." If you update a separate spreadsheet (DB) and then forget to mark the ledger page, you might update the spreadsheet twice.

### Distributed systems detail
- Exactly-once requires atomicity between consuming and producing side effects.
- Some systems provide this end-to-end:
  - Kafka Streams with Kafka transactional producer to Kafka sinks
  - Flink with two-phase commit sinks
- Many real systems settle for effectively-once:
  - idempotent writes
  - dedup keys
  - upserts

**Important nuance:** "exactly-once" is always scoped.
- Flink's exactly-once is typically **exactly-once state updates + exactly-once sink semantics** *if* the sink participates correctly.
- External systems (HTTP calls, non-transactional DBs) break the guarantee unless you add idempotency/outbox.

[CODE: python, idempotent sink upsert using event_id]
```python
# Implementing an idempotent sink (upsert by event_id) for effectively-once processing
import sqlite3
from contextlib import closing

DDL = """
CREATE TABLE IF NOT EXISTS revenue_events (
  event_id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  amount_cents INTEGER NOT NULL,
  ts_ms INTEGER NOT NULL
);
"""

def upsert_revenue_event(db_path: str, event: dict) -> None:
    # Idempotency: PRIMARY KEY(event_id) makes duplicates safe (INSERT OR IGNORE)
    try:
        with closing(sqlite3.connect(db_path, timeout=5)) as conn:
            conn.execute("PRAGMA journal_mode=WAL")  # safer under concurrent writers
            conn.execute(DDL)
            conn.execute(
                "INSERT OR IGNORE INTO revenue_events(event_id,user_id,amount_cents,ts_ms) VALUES(?,?,?,?)",
                (event["event_id"], event["user_id"], event["amount_cents"], event["ts_ms"]),
            )
            conn.commit()
    except (KeyError, sqlite3.Error) as e:
        raise RuntimeError(f"sink write failed for event={event!r}: {e}")

# Usage example
upsert_revenue_event("./revenue.db", {"event_id": "e-123", "user_id": "u1", "amount_cents": 499, "ts_ms": 1700000000000})
upsert_revenue_event("./revenue.db", {"event_id": "e-123", "user_id": "u1", "amount_cents": 499, "ts_ms": 1700000000000})  # duplicate ignored
```

Production caveat:
- `INSERT OR IGNORE` is safe for dedup, but it also **drops legitimate corrections** if the same `event_id` can be re-emitted with corrected fields.
  - If corrections are possible, model them explicitly (e.g., `event_id` + `version`, or use upsert with last-write-wins by `(event_id, updated_at)`).

Key insight:

> Kappa's replay makes correctness more visible: you will replay, so your pipeline must be deterministic and your sinks must tolerate duplicates.

Challenge question:
If you cannot make the sink idempotent, what alternative architecture pattern can you use (hint: write-ahead log / outbox / transactional sink)?

---

## [DISTRIBUTED SYSTEMS RIGOR] CAP, partitions, and what you actually choose

### Network assumptions (state them explicitly)
- Partitions happen: between producers and Kafka, between Kafka brokers, between processors and sinks.
- Clocks are not perfectly synchronized; event-time is a logical model, not a physical truth.

### CAP implications in this domain
You are not choosing CAP for "the architecture"; you are choosing it for each subsystem:
- **Kafka**: designed to be partition-tolerant; availability depends on replication/acks and ISR health.
- **Stream processor state** (e.g., RocksDB + checkpoints): partition-tolerant; availability depends on checkpoint storage and failover.
- **Serving store** (Pinot/Druid/Cassandra/ClickHouse): each has its own CAP-ish trade-offs.

Practical takeaway:
- During a partition, you often choose between:
  - **Serving stale data** (availability) vs
  - **Serving no data / error** (consistency of freshness)

Challenge question:
What does your product prefer during an incident: stale-but-available dashboards, or hard errors?

---

## [OPERATIONS] Two pipelines vs one pipeline

### [CHALLENGE] Scenario
You are on-call. It is 2 AM.

In Lambda, you might have:

- Spark batch job failures (YARN/K8s)
- Streaming job failures (Flink/K8s)
- Serving layer merge bugs

In Kappa, you might have:

- One streaming job
- But replay procedures, versioned outputs, and long retention

### Decision game: Which statement is true?

1) Lambda always costs more to operate.
2) Kappa always costs more to operate.
3) It depends on where your organization is strong: batch governance vs streaming discipline.

Pause and think.

### Answer
Correct: 3.

### Comparison table: operational surfaces

| Dimension | Lambda | Kappa |
|---|---|---|
| Pipelines | Two (batch + speed) | One (stream) |
| Serving complexity | Merge logic required | No merge, but versioning during replay |
| Reprocessing | Batch recompute is natural | Replay log; may require long retention |
| Failure blast radius | Isolated per layer, but merge can hide inconsistencies | Single pipeline; failures directly affect all views |
| Skill set | Batch + streaming + serving | Deep streaming + state + exactly-once-ish sinks |
| Debugging | Compare batch vs speed outputs | Trace event log + state snapshots |

Key insight:

> Lambda spreads complexity across systems; Kappa concentrates complexity into the stream processor + log discipline.

Challenge question:
Which architecture is easier to staff if your team has strong Spark skills but minimal streaming experience?

---

## [FAILURES] Failure scenarios (distributed reality)

### [CHALLENGE] Scenario 1: Stream processor crash loop
Your stream job repeatedly fails due to a bad deployment.

Lambda impact:
- Speed layer down -> dashboards stale
- Batch still runs -> eventually "correct" numbers

Kappa impact:
- Only pipeline down -> both real-time and historical incremental views stop updating

Pause and think:
Which is more acceptable for your business: temporary stale dashboards, or incorrect dashboards?

Key insight:

> Lambda can degrade into "batch-only mode." Kappa needs strong availability practices for the stream layer.

Production guardrails:
- Canary deployments for stream jobs (separate consumer group writing to shadow outputs)
- Schema compatibility checks at deploy time (Schema Registry rules)
- Feature flags for new metrics logic
- Automated rollback on error-rate / lag thresholds

Challenge question:
What deployment guardrails would prevent crash loops (canary, shadow traffic, schema checks, etc.)?

---

### [CHALLENGE] Scenario 2: Serving layer inconsistency
Your serving DB has partial writes due to a node failure.

Some partitions updated, others not.

Interactive question:
Which architecture makes it easier to rebuild the serving store?

A) Lambda, because batch can rebuild it
B) Kappa, because replay can rebuild it
C) Both, but the rebuild procedure differs

Pause and think.

Answer:
Correct: C.

- Lambda: rebuild from batch outputs (or recompute batch view)
- Kappa: rebuild by replaying events into a fresh store

Key insight:

> Rebuildability is a shared goal; the difference is whether you rebuild from a batch snapshot or from the event log.

Production pattern: avoid partial rebuild visibility
- Build into a **new index/table/topic**
- Validate completeness (row counts, checksums, reconciliation)
- Atomically switch (alias swap / routing)

Challenge question:
How do you ensure consumers do not see partial rebuild results?

---

### [CHALLENGE] Scenario 3: Backfill after a bug
You discover a bug in sessionization logic that existed for 30 days.

Lambda:
- Fix batch job -> recompute last 30 days in batch
- Speed layer continues for new events
- Serving layer must reconcile corrected batch with speed

Kappa:
- Fix stream job -> replay last 30 days (or full log)
- Write to a new versioned output
- Atomically switch readers

Decision game: Which is riskier?

1) Recomputing batch while speed continues with different logic
2) Replaying stream while keeping old output until cutover

Pause and think.

Discussion:
Both have risks:

- Lambda risk: dual semantics during transition; merge boundary errors.
- Kappa risk: long replay time; resource contention; ensuring deterministic results.

Key insight:

> Backfills are where architectures reveal their true operational cost.

Production validation checklist (before cutover):
- Reconciliation vs source-of-truth totals (payments ledger, orders DB)
- Invariants (e.g., revenue >= 0, sessions end after start)
- Diff sampling (compare v1 vs v2 for a stratified sample)
- Late-data behavior (does v2 change historical windows as expected?)

Challenge question:
What validation checks would you run before switching from old to new outputs?

---

## [MENTAL MODEL] Where does truth live?

Two competing truths:
- Truth in computed views (batch outputs are canonical)
- Truth in raw events (log is canonical)

Lambda often treats batch views as the "official truth".
Kappa treats the log as the "official truth".

Interactive question:
If your business asks, "What was the revenue number we reported last Tuesday at 10 AM?" which architecture naturally supports that audit question?

Pause and think.

Answer:
Neither automatically.

You need:
- Versioned outputs (views with metadata)
- Immutable snapshots or time-travel
- Audit logs of deployments and logic versions

Key insight:

> Architecture patterns do not replace governance. They change what is practical to govern.

Challenge question:
What metadata would you store with every materialized view build (git SHA, schema version, watermark, input offsets)?

---

## [DEEP DIVE] Data modeling: materialized views, late data, corrections

### [CHALLENGE] Scenario
You maintain a "daily active users" (DAU) table.

But events arrive late (up to 48 hours). You need corrections.

Interactive question:
Which approach is more natural?

A) Only append to DAU; never correct
B) Use upserts with event-time windows and allow retractions/corrections
C) Ignore late events

Pause and think.

Answer:
Correct: B.

Real-world parallel:
A coffee shop tallying daily sales: if a delivery receipt arrives late, you adjust yesterday's totals.

Distributed systems angle:
To support corrections you need:

- Event-time windows + allowed lateness
- State that can be updated
- Sinks that support upserts or compaction

[CODE: sql, Flink SQL event-time windows + watermark + upsert sink]
```sql
-- Implementing event-time DAU with late-data handling and an upsert sink (Flink SQL)
-- Assumes a Kafka source with event_time in milliseconds and a compacted Kafka sink.

CREATE TABLE user_events (
  user_id STRING,
  event_type STRING,
  event_time_ms BIGINT,
  event_time AS TO_TIMESTAMP_LTZ(event_time_ms, 3),
  WATERMARK FOR event_time AS event_time - INTERVAL '5' MINUTE
) WITH (
  'connector' = 'kafka',
  'topic' = 'user_events',
  'properties.bootstrap.servers' = 'kafka:9092',
  'format' = 'json',
  'scan.startup.mode' = 'group-offsets'
);

CREATE TABLE dau_by_day (
  day DATE,
  dau BIGINT,
  PRIMARY KEY (day) NOT ENFORCED
) WITH (
  'connector' = 'upsert-kafka',
  'topic' = 'dau_by_day',
  'properties.bootstrap.servers' = 'kafka:9092',
  'key.format' = 'json',
  'value.format' = 'json'
);

INSERT INTO dau_by_day
SELECT CAST(TUMBLE_START(event_time, INTERVAL '1' DAY) AS DATE) AS day,
       COUNT(DISTINCT user_id) AS dau
FROM user_events
WHERE event_type = 'app_open'
GROUP BY TUMBLE(event_time, INTERVAL '1' DAY);
```

Production caveat:
- `COUNT(DISTINCT ...)` in streaming can be expensive and may require large state.
  - Consider approximate distinct (HyperLogLog) if exactness is not required.
  - Or pre-aggregate by user/day with a keyed state and emit 0/1 transitions.

Key insight:

> Kappa pushes you toward mutable materialized views built from immutable events.

Challenge question:
If your serving store does not support upserts, how do you model corrections (delta tables, compaction jobs, or append-only with query-time reconciliation)?

---

## [MISCONCEPTION] "Kappa means no batch ever"

Misconception:
"Kappa = streaming only, so we never run batch jobs."

Reality:
Kappa is about one processing model, not banning batch compute.

You might still run:

- Offline ML training
- Ad-hoc analytics (Spark/Trino)
- One-time migrations

But the authoritative pipeline for derived views is stream + replay.

Key insight:

> Kappa reduces pipeline duplication, not the existence of all batch workloads.

Challenge question:
What batch workloads remain even if you adopt Kappa for analytics views?

---

## [ECONOMICS] Performance and cost: the economics of recomputation

### [CHALLENGE] Scenario
You store 1 TB/day of events.

- Retain 180 days = 180 TB raw
- Need to recompute monthly reports when logic changes

Interactive question:
Which is cheaper?

1) Recompute from raw events every time (Kappa replay)
2) Keep periodic batch snapshots and recompute from snapshots (Lambda-ish)

Pause and think.

Explanation:
Often, snapshots reduce recomputation cost:

- Replaying 180 TB may take hours/days and heavy cluster resources.
- Snapshotting intermediate state (e.g., daily aggregates) can reduce replay scope.

This leads to a hybrid reality:

- Kappa-like streaming pipeline
- Plus periodic checkpoints/snapshots stored in a lake

Key insight:

> The "pure" patterns are ideals; real systems often adopt snapshotting to control replay cost.

Performance considerations for replay:
- **Read amplification**: re-reading raw events stresses brokers/object store.
- **Write amplification**: rebuilding views can saturate serving stores.
- **State size**: large keyed state increases checkpoint time and recovery time.

Challenge question:
How would you choose a snapshot cadence (hourly/daily/weekly) given your replay RTO and storage budget?

---

## [SERVING] Serving layer design: merging vs switching

### [CHALLENGE] Scenario
You expose a query API for dashboards.

- In Lambda, query must combine batch + speed.
- In Kappa, query reads one materialized view.

Interactive question:
Which is simpler for consumers?

A) Consumers query two sources and merge
B) Serving layer merges behind one API
C) A single source per view (Kappa), but versioned cutovers

Pause and think.

Answer:
Correct: C is often simplest at query time.

But note: Kappa's simplicity shifts complexity to deployment and versioning.

Real-world parallel:
A restaurant menu:

- Lambda: waiter combines dishes from two kitchens.
- Kappa: one kitchen, but when you change the recipe you run a "new menu version" and switch all tables at once.

Key insight:

> Lambda complicates reads; Kappa complicates upgrades.

Challenge question:
What does "atomic switch" mean for your serving tech (alias swap, view swap, routing layer)?

---

## [GAME] Choose an architecture under constraints

### [CHALLENGE] Scenario
You are building analytics for a global marketplace.

Constraints:

- Kafka retention: 14 days
- Need: recompute 90 days on logic changes
- Stream processor: Flink
- Serving: Pinot
- Team: strong Spark, moderate Flink
- Compliance: must delete user data on request within 7 days

Which statement is most true?

1) Kappa is impossible because retention is 14 days.
2) Lambda is required because you need 90-day recompute.
3) You can do Kappa if you treat the data lake as the long-term log and replay from it.

Pause and think.

Answer (progressive reveal):
Correct: 3.

You can implement a "log" in two tiers:

- Kafka for hot ingestion (14 days)
- S3/HDFS as cold immutable storage (years)

Then you can replay via:

- Backfill jobs that republish from S3 to Kafka, or
- Directly run a streaming/batch engine over S3 with the same code path (Beam model)

Key insight:

> In practice, Kappa's "log" may be Kafka + lake, not Kafka alone.

Compliance clarification:
- Deletion within 7 days conflicts with immutable retention unless you:
  - avoid storing PII in the log,
  - encrypt payloads and delete keys (crypto-shredding), and
  - ensure derived views also purge.

Challenge question:
How does the deletion requirement interact with immutable logs? What strategies exist?

---

## [DEEP DIVE] The merge problem in Lambda: where bugs hide

### [CHALLENGE] Scenario
Your serving layer answers:

result = batch_view(up_to_T) + speed_view(after_T)

But:

- Speed layer uses event-time
- Batch layer uses processing-time
- Or different dedup logic

Interactive question:
What is the most common Lambda failure mode?

A) Batch layer is too slow
B) Speed layer is too slow
C) Batch and speed compute different answers for the same events

Pause and think.

Answer:
Correct: C.

Why it happens:
Two pipelines drift:

- Different codebases or libraries
- Different schema evolution handling
- Different windowing semantics
- Different late-data policies

Real-world parallel:
Two accountants with different rules producing two ledgers; the final report merges them and hopes they align.

Key insight:

> Lambda's core operational risk is semantic divergence.

Production mitigation:
- Single source of truth for business logic (shared library or single SQL definition)
- Golden datasets + regression tests run against both pipelines
- Dual-run comparisons (shadow compute) with alerting on divergence

Challenge question:
What automated tests would catch divergence (golden datasets, property tests, dual-run comparisons)?

---

## [MISCONCEPTION] "Lambda guarantees correctness"

Misconception:
"Batch layer is the source of truth, so the overall system is correct."

Reality:
Correctness depends on:

- Batch job correctness
- Merge correctness
- Handling of late data and duplicates
- Consistent schemas

Lambda can still be wrong, just wrong with more moving parts.

Challenge question:
How would you detect semantic divergence between batch and speed outputs automatically?

---

## [DEEP DIVE] Reprocessing in Kappa: replay is not a button

### [CHALLENGE] Scenario
You need to replay 60 days of events.

Interactive question:
What do you need before you replay?

Pick all that apply:

- [ ] A) A new output location (versioned topic/table)
- [ ] B) A plan to throttle replay to protect downstream systems
- [ ] C) A deterministic job (same inputs -> same outputs)
- [ ] D) A way to cut over readers atomically

Pause and think.

Answer:
All of them: A, B, C, D.

Explanation:
Replay is a distributed migration:

- You are effectively running a batch job using streaming machinery.
- The sink might not handle sudden write amplification.

Real-world parallel:
You are recalling and re-delivering 60 days of packages with corrected labels; your warehouse and drivers must handle the surge.

Key insight:

> Kappa makes recomputation conceptually simple, but operationally it is a large-scale data migration.

Production replay runbook (minimum viable):
- Capacity plan (CPU, network, sink write IOPS)
- Rate limits (records/sec per partition)
- Isolation (dedicated backfill cluster or lower-priority queue)
- Checkpointing strategy (avoid huge checkpoints during replay)

Challenge question:
What is your replay throttling mechanism (rate limits, partition-by-partition, priority queues)?

---

## [OPERATIONS] Versioning strategy: blue/green for data products

### [CHALLENGE] Scenario
You maintain a "user_metrics" materialized view.

You want to deploy v2 of the logic.

Interactive question:
Which cutover strategy is safest?

1) Overwrite the existing table in place
2) Write v2 to a new table, validate, then switch
3) Deploy and hope

Pause and think.

Answer:
Correct: 2.

Distributed systems pattern:
- Blue/green outputs:
  - Write to user_metrics_v2
  - Run validation checks
  - Switch query routing / views
  - Decommission v1

[IMAGE: Blue/green data pipeline cutover diagram with v1 and v2 materialized views, validation step, and traffic switch]

Key insight:

> In Kappa, "deployment" includes output versioning and validation, not just code rollout.

Challenge question:
How do you validate v2 without doubling serving cost forever?

---

## [DEEP DIVE] Tooling reality: frameworks and what they imply

### [CHALLENGE] Scenario
You are choosing a stack.

Comparison table: common ecosystem choices

| Layer | Lambda typical | Kappa typical |
|---|---|---|
| Log | Kafka/Pulsar | Kafka/Pulsar + long retention (or lake) |
| Stream processing | Flink / Kafka Streams / Beam | Flink / Kafka Streams / Beam |
| Batch processing | Spark / Hive / Trino | Optional; often still Spark/Trino for ad-hoc |
| Serving | Druid/Pinot/ClickHouse/Cassandra/Elastic | Same, but usually one canonical view |
| Governance | Data lake catalogs, batch SLAs | Stream lineage, schema registry, replay runbooks |

Interactive question:
Which framework makes it easiest to share code between "batch" and "stream"?

A) Apache Beam
B) Anything, it is just code

Pause and think.

Answer:
A, conceptually.

Beam's unified model can reduce semantic drift by using one API and runner.

Key insight:

> Unified programming models reduce Lambda's drift risk and make Kappa-style replay more approachable.

Challenge question:
If you are not using Beam, what practices keep batch and stream semantics aligned (shared libraries, golden tests, single SQL definition)?

---

## [PUZZLE] Spot the hidden assumptions

### [CHALLENGE] Scenario
A teammate says:

"Let's do Kappa. We'll just replay Kafka from earliest when we need to backfill."

Pause and think:
What assumptions are hidden here? Write down at least three.

Answer (possible list):
- Kafka retains data long enough
- Replaying will not overload sinks
- The job is deterministic across versions
- You can isolate old vs new outputs
- You can validate correctness
- You can handle schema evolution across old events

Key insight:

> Many Kappa failures come from assuming replay is trivial.

Challenge question:
Which assumption is most likely to fail first in your environment?

---

## [DEEP DIVE] Schema evolution and compatibility: the slow-burn distributed failure

### [CHALLENGE] Scenario
You add a field to events. Old events do not have it.

- Speed layer sees new schema immediately.
- Batch layer reads months of mixed schemas.

Interactive question:
Which architecture is more sensitive to schema evolution mistakes?

A) Lambda, because two pipelines parse data differently
B) Kappa, because replay re-reads old events with new logic
C) Both

Pause and think.

Answer:
Correct: C, but in different ways.

- Lambda: drift between batch and speed parsers.
- Kappa: replay can fail if new code cannot read old events.

Best practices:
- Schema registry with compatibility rules
- Versioned event types
- "Never break old readers" discipline

[CODE: javascript, Protobuf backward-compatible schema evolution with defaults]
```javascript
// Implementing backward-compatible Protobuf evolution: new field with safe default
// Requires: npm i protobufjs
const protobuf = require("protobufjs");

async function demoSchemaEvolution() {
  try {
    // v1: older producers only know user_id + amount_cents
    const v1 = `syntax = "proto3"; message Purchase { string user_id = 1; int64 amount_cents = 2; }`;
    // v2: adds currency (field 3). Old data decodes with default "" (treat as "USD").
    const v2 = `syntax = "proto3"; message Purchase { string user_id = 1; int64 amount_cents = 2; string currency = 3; }`;

    const RootV1 = protobuf.parse(v1).root;
    const RootV2 = protobuf.parse(v2).root;
    const PurchaseV1 = RootV1.lookupType("Purchase");
    const PurchaseV2 = RootV2.lookupType("Purchase");

    const bytesFromOldProducer = PurchaseV1.encode({ user_id: "u1", amount_cents: 499 }).finish();
    const decodedByNewConsumer = PurchaseV2.decode(bytesFromOldProducer);
    const currency = decodedByNewConsumer.currency || "USD"; // explicit defaulting policy

    console.log({ ...decodedByNewConsumer, currency });
  } catch (err) {
    throw new Error(`schema evolution demo failed: ${err.message}`);
  }
}

// Usage example
demoSchemaEvolution().catch((e) => console.error(e));
```

Key insight:

> The event log is an API. Treat it like one.

Challenge question:
How do you test that your new consumer can read 12 months of historical events?

---

## [MISCONCEPTION] "Event sourcing = Kappa"

Misconception:
"If we do event sourcing, we are doing Kappa."

Reality:
Event sourcing is a domain modeling approach: store state changes as events.

Kappa is a data processing architecture: compute views from a log.

They often pair well, but neither implies the other.

Challenge question:
How would you build queryable projections from an event-sourced system? What looks like Kappa there?

---

## [DEEP DIVE] Consistency and isolation: what do readers see?

### [CHALLENGE] Scenario
Your dashboard queries a materialized view while it is being rebuilt.

Interactive question:
Which is safer?

1) Rebuild in place (partial results visible)
2) Build new version and switch atomically

Pause and think.

Answer:
2.

Distributed systems detail:
Atomic switch can be done via:

- Alias swap in Elasticsearch
- Table/view swap in warehouses
- Pinot/Druid segment versioning
- Feature flag routing at API layer

Key insight:

> Readers hate partial truth. Use versioned outputs to provide isolation.

Challenge question:
What is your rollback plan if v2 looks wrong after cutover?

---

## [OPERATIONS] Observability: how do you know you are right?

### [CHALLENGE] Scenario
Your pipeline outputs "revenue per minute".

You need to detect:

- Dropped events
- Duplicates
- Lag
- Divergence between versions

Interactive question:
Which metrics are most diagnostic? Pick two:

A) Consumer lag
B) Output row count
C) End-to-end reconciliation vs source-of-truth totals
D) CPU usage

Pause and think.

Answer:
Most diagnostic: A and C.

- Lag tells you timeliness.
- Reconciliation tells you correctness.

Production additions:
- Input vs output rate by partition (detect hot partitions / skew)
- Checkpoint duration + failure rate (Flink)
- Sink error rate / retries / timeouts
- Data quality metrics (null rates, schema violations)

Key insight:

> Distributed pipelines need both liveness (lag, throughput) and correctness (reconciliation) signals.

Challenge question:
What is your reconciliation source of truth (payments DB, ledger, warehouse snapshot), and how often do you compare?

---

## [GAME] Quiz: Identify the architecture from symptoms

Symptom set 1
- Two codebases for the same metric
- Dashboards show "preliminary" numbers
- Nightly job corrects yesterday

Which is it? Pause and think.

Answer: Lambda.

Symptom set 2
- One stream job computes everything
- When logic changes, you run a 3-day replay
- You maintain versioned materialized views

Which is it? Pause and think.

Answer: Kappa.

Challenge question:
What symptom would indicate you are accidentally building a third, undocumented pipeline?

---

## [DEEP DIVE] When Lambda still wins (pragmatically)

### [CHALLENGE] Scenario
You run complex ML feature extraction over a year of data.

- Requires heavy joins across datasets
- Not naturally expressible as incremental streaming

Interactive question:
What is a reasonable design?

A) Force it into streaming no matter what
B) Use Lambda/hybrid: batch for heavy historical joins, stream for online features

Pause and think.

Answer:
B.

Key insight:

> Some computations are inherently batch-friendly. Architectures are tools, not identities.

Challenge question:
Name a computation that is hard to make incremental and why.

---

## [DEEP DIVE] When Kappa wins (pragmatically)

### [CHALLENGE] Scenario
You have many derived views and frequent logic changes.

- Maintaining two pipelines is causing drift
- Your stream processor is mature
- You can retain events long enough

Interactive question:
What advantage matters most?

A) Reduced semantic drift
B) Reduced storage cost

Pause and think.

Answer:
A.

Key insight:

> Kappa shines when "same logic twice" is your biggest pain.

Challenge question:
What is your plan for replaying without impacting real-time SLAs (separate clusters, priority scheduling, dedicated backfill jobs)?

---

## [PATTERN] Kappa + snapshots (a common hybrid)

### [CHALLENGE] Scenario
Pure replay from day 0 is too expensive.

Pattern:
- Keep the raw log (Kafka + lake)
- Periodically checkpoint derived state (daily snapshots)
- For replay, start from the latest snapshot + replay recent events

[IMAGE: Hybrid diagram showing raw log, periodic snapshots, and replay starting from snapshot rather than from beginning]

Interactive question:
Is this still Kappa?

Pause and think.

Answer:
It is Kappa-inspired. The core remains: one processing model and replayable truth.

Key insight:

> Snapshots reduce replay cost without reintroducing a separate batch semantics pipeline.

Correctness risk of snapshots:
- Snapshot corruption or partial writes
- Snapshot taken with a different logic/schema version than the replay job

Mitigations:
- Store snapshot metadata (job version, schema version, input offsets)
- Validate snapshot integrity (checksums, row counts)
- Keep at least N previous snapshots for rollback

Challenge question:
What is the correctness risk of starting from a snapshot (snapshot corruption, version mismatch), and how do you mitigate it?

---

## [DEEP DIVE] Security and compliance: immutable logs meet deletion requests

### [CHALLENGE] Scenario
You must delete a user's data within 7 days.

But your event log is immutable and retained for 180 days.

Decision game: Which statement is true?

1) Immutable logs make compliance impossible.
2) You can comply by encrypting per-user data and deleting keys, or by storing PII separately.

Pause and think.

Answer:
Correct: 2.

Strategies:
- Crypto-shredding: encrypt user payload with per-user key; delete key to render data unreadable.
- PII indirection: store PII in a separate store with deletion; events contain references.
- Tombstone events: mark deletions and ensure projections respect them (still leaves raw data).

Production caveat:
- Tombstones alone rarely satisfy strict regimes if raw PII remains accessible.
- Crypto-shredding requires key management discipline and re-encryption strategy.

Key insight:

> Compliance requirements often drive retention and architecture more than technical preference.

Challenge question:
If you crypto-shred, how do you ensure projections and serving stores also purge derived PII?

---

## [GAME] Final decision game: Which architecture should you choose?

### [CHALLENGE] Scenario
Choose between Lambda and Kappa for each case.

Case A: Ad analytics at massive scale
- Need sub-second dashboards
- Reports are reconciled daily
- Data volume huge; retention cost matters
- Team already runs Spark batch reliably

Pause and think: Lambda or Kappa?

Reveal:
Often Lambda/hybrid, because batch reconciliation is already institutionalized and replaying months of ad logs can be expensive.

Case B: Product metrics for fast iteration
- Metrics definitions change weekly
- Need consistent semantics across real-time and historical
- Retention is affordable
- Stream processing expertise is strong

Pause and think: Lambda or Kappa?

Reveal:
Often Kappa, because semantic drift is the enemy and replay is a feature.

Case C: Regulated finance reporting
- Must produce certified monthly statements
- Strong audit requirements
- Corrections must be tracked and explainable

Pause and think: Lambda or Kappa?

Reveal:
Often Lambda or Kappa+governance, but the key is auditability: versioned outputs, immutable snapshots, and reconciliation. The pattern matters less than governance.

Key insight:

> Architecture choice is rarely about ideology; it is about operational risk under your constraints.

Challenge question:
For each case, what is the primary on-call nightmare (semantic drift, replay overload, merge boundary bugs, or retention gaps)?

---

## [SYNTHESIS] Design your own pipeline

### [CHALLENGE] Scenario
You ingest events for a ride-sharing app:

- TripRequested, DriverAssigned, TripStarted, TripEnded, PaymentCaptured

Requirements:

- Real-time surge pricing metrics (seconds)
- Daily financial reconciliation (must be correct)
- Ability to recompute 120 days when pricing logic changes
- Kafka retention: 30 days
- Data lake available (S3)

Your tasks (pause and think)
1) Choose Lambda, Kappa, or hybrid.
2) Where is the source of truth?
3) How do you handle replay/backfill?
4) How do you cut over view versions safely?
5) What failure mode scares you most?

Reveal (one strong design)
- Hybrid Kappa:
  - Kafka for hot log (30 days)
  - S3 as cold immutable event store (120+ days)
  - Flink job computes materialized views
  - Periodic snapshots of state to accelerate replays
  - Blue/green versioned outputs for logic upgrades
  - Reconciliation job compares aggregates against accounting system

Key insight:

> The best architectures are replayable, observable, and operationally boring.

Final challenge question:
If you had to bet your on-call sanity on one capability, which would you invest in?

1) Automated reconciliation between raw log and views
2) One-click replay/backfill runbooks with throttling + validation
3) A serving layer that can merge two sources flawlessly

Write down your answer and the failure that answer prevents.
