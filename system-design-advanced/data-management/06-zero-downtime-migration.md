---
slug: zero-downtime-migration
title: Zero Downtime Migration
readTime: 20 min
orderIndex: 6
premium: false
---






# Zero-Downtime Migrations

> You're about to change the shape of your data while the system is live, traffic is unpredictable, and multiple services are reading/writing concurrently.
>
> The goal: **no user-visible downtime** and **no data loss**.

---

## [HANDSHAKE] How to use this article

This is an interactive, progressive-reveal guide. You'll repeatedly see:

- **Pause and think** prompts
- Decision games ("Which statement is true?")
- Matching exercises
- "Common Misconception" callouts
- Mental models and failure scenarios
- Practical patterns and trade-offs

If you're reading this as a team, assign roles:

- **DBA/Platform**: owns schema, migrations, backups, replication, DDL safety
- **Service owner(s)**: owns application changes and deploys
- **SRE**: owns rollout safety, observability, incident response

At the end you'll face a final synthesis challenge that combines everything.

---

## [TARGET] Challenge: Friday 2pm, you need to change a core table

### Scenario
You run a global e-commerce platform. Your `orders` table is central: every checkout writes to it; fulfillment reads it; customer support queries it; analytics streams it.

A product manager asks: "We need to support multi-currency with per-line exchange rates. Can you add `currency`, `fx_rate`, and change `total_amount` semantics?"

Traffic is constant. You can't take a maintenance window.

### Pause and think
If you change the schema in place (e.g., `ALTER TABLE orders ADD COLUMN currency ...`), what can go wrong in a distributed system?

Write down 5 failure modes.

(Keep reading when you've got your list.)

### Progressive reveal: common failure modes
1. **Old services crash** when they see new schema constraints or changed semantics.
2. **New services crash** when deployed before the schema exists (or before replicas catch up).
3. **Mixed versions** (old + new) write inconsistent data.
4. **Long-running DDL locks** block writes and cause cascading timeouts.
5. **Replication lag** means read replicas see schema/data later than primary.
6. **CDC/streaming consumers** break due to schema change events or semantic changes.
7. **Backfill jobs** overload DB and starve OLTP.
8. **Rollback is hard**: you can't "un-migrate" easily under load.

### Real-world analogy (coffee shop)
You're renovating a coffee shop **while it stays open**. Customers keep ordering. If you move the espresso machine, you must ensure:

- Baristas still know where to find it (compatibility)
- Orders aren't lost mid-renovation (consistency)
- The shop doesn't block the entrance for 30 minutes (availability)

### Key insight
> **Zero-downtime migrations are not a single DDL statement.** They're a coordinated, multi-step protocol across schema, application versions, data backfills, and operational safeguards.

**Challenge question:** What's the one thing you must assume in distributed deployments?

Pause and think.

Answer: **You will run mixed versions concurrently** (for minutes to hours), and they will interact with the same data.

---

## [MAGNIFIER] Mental model: migrations as a compatibility contract

### Scenario
You have 12 microservices. 4 write to the same table. Deploys are rolling; some pods update slower. Meanwhile, read replicas serve 60% of reads.

### Pause and think
What does "backward compatibility" mean for a schema?

- A) Old app works with new schema
- B) New app works with old schema
- C) Both

Hold your answer.

### Reveal
It depends on the migration phase, but **you must design for both directions across time**.

- During rollout: **new app must tolerate old schema** (because schema may not be applied everywhere yet, or replicas lag).
- After schema change: **old app must tolerate new schema** (because you might roll back the app).

So in practice: **C) both**, but not always simultaneously for all operations—hence phased protocols.

### The contract
Think of schema + application as a client-server protocol:

- Schema provides "fields" and constraints
- Application is a client reading/writing them

A safe migration is like evolving an API: additive changes first, destructive changes last.

### Real-world parallel (delivery service)
A delivery platform changes its address format. If drivers' apps update gradually, the backend must accept **both old and new address formats** until everyone catches up.

### Key insight
> Treat schema changes like **API versioning**. Your DB is a shared dependency with slow, partial rollouts.

**Challenge question:** What's the schema equivalent of "breaking an API"? List 3.

Examples:
- Dropping/renaming a column that clients read
- Tightening a constraint (e.g., `NOT NULL`, `CHECK`) that old writers violate
- Changing semantics (same column name, different meaning)

---

## [MAGNIFIER] Distributed systems rigor: what you can and cannot assume

### Network assumptions (state them explicitly)
- The network can drop, delay, duplicate, and reorder packets.
- Nodes can be slow (GC pauses, noisy neighbors) without being "down".
- Deployments are rolling and non-atomic.
- Replication is asynchronous unless you explicitly pay for synchronous replication.

### CAP implications (why migrations are hard)
During a migration you often choose between:
- **Consistency**: all readers see the same schema/data immediately
- **Availability**: the system continues to serve requests during partial rollout
- **Partition tolerance**: you don’t get to opt out

Most production systems choose **AP-ish behavior during partitions** (keep serving), which means:
- replicas can be stale
- caches can be stale
- consumers can lag

So your migration protocol must tolerate **inconsistency windows** and reconcile.

### Consistency model clarity
- Primary DB reads are typically **read-your-writes** (within a session/transaction).
- Replica reads are typically **eventually consistent**.
- CDC/event streams are typically **at-least-once** delivery.

**Production insight:** If you require strong consistency during cutover, you must route reads to the primary (or use synchronous replication / quorum reads), which reduces availability and increases latency.

---

## [SIREN] Common Misconception: "Zero downtime means no risk"

### Misconception
"If we do it without downtime, users won't notice and we're safe."

### Reality
Zero-downtime migrations often **increase** complexity and risk because:

- You run **dual writes** or **dual reads**
- You maintain **two representations** of data
- You perform **online backfills** under load
- Rollbacks become **stateful** (data already transformed)

### Pause and think
Which is riskier?

1. A 10-minute planned maintenance window
2. A 2-week dual-write migration

Don't answer "it depends" yet—pick one.

### Reveal
Often **(2)** is riskier operationally, but (1) may be unacceptable product-wise.

The engineering goal becomes: **reduce the risk of (2)** with disciplined patterns.

### Key insight
> "Zero downtime" is a product requirement; "low risk" is an engineering requirement. You must optimize both.

**Challenge question:** What observability signals become more important during dual-write?

Examples:
- mismatch counters between old/new representations
- write error rates by code version
- replica lag and WAL volume
- deadlocks/lock waits

---

## [PUZZLE] The core playbook: Expand -> Migrate -> Contract

This is the foundational protocol used across many systems.

### Scenario
You need to rename `orders.total_amount` to `orders.total_amount_minor` and change semantics.

### The three phases
1. **Expand**: Add new schema elements (columns/tables/indexes) in a backward-compatible way.
2. **Migrate**: Backfill/transform data; update application to read/write new fields.
3. **Contract**: Remove old schema elements once no longer used.

### Pause and think
Why is "contract" last?

Think in terms of rollback.

### Reveal
Because once you drop old columns/tables, **old application versions can no longer function**, and rollback becomes impossible without restoring schema and possibly data.

### Real-world analogy (restaurant menu)
You introduce a new menu item (expand), train staff and shift customers to it (migrate), then remove the old item (contract). Removing the old item first breaks orders during the transition.

### Key insight
> Expand/Contract makes schema evolution **monotonic** during rollout: you only add capabilities until you're sure you don't need the old ones.

**Challenge question:** Name 2 schema changes that are "expand-safe" and 2 that are "contract-only."

Examples:
- Expand-safe: add nullable column; add index concurrently; add new table
- Contract-only: drop column/table; tighten constraint to reject previously valid data

---

## [GAME] Decision game: Which migration is safe under rolling deploy?

### Scenario
You have service `checkout` writing to `orders`. You want to add a NOT NULL column `currency`.

Choose the safest sequence:

**Option A**
1. `ALTER TABLE orders ADD COLUMN currency TEXT NOT NULL;`
2. Deploy app writing `currency`

**Option B**
1. `ALTER TABLE orders ADD COLUMN currency TEXT NULL;`
2. Deploy app writing `currency` (and defaulting if absent)
3. Backfill old rows
4. Add `NOT NULL` constraint

**Option C**
1. Deploy app writing `currency`
2. Add column

Pause and think.

### Reveal
**Option B** is the classic safe approach.

- A can fail because adding `NOT NULL` without a default will fail on existing rows and can take stronger locks.
- C can fail because new app writes a column that doesn't exist yet.

### Key insight
> Prefer **add nullable -> dual-write -> backfill -> enforce constraint**.

**Challenge question:** What's the hidden risk in step 4 (adding NOT NULL) on large tables?

Answer: it can require a table scan and take locks that block writes (Postgres requires a table scan to verify no NULLs; lock level and duration depend on version and concurrent activity). Plan it like a production rollout.

---

## [MAGNIFIER] DDL in distributed environments: locks, replicas, and blast radius

### Scenario
Your primary DB is PostgreSQL with streaming replicas. You run `ALTER TABLE ...` at peak traffic.

### Pause and think
What does "online DDL" actually mean?

- A) It never blocks
- B) It blocks briefly
- C) It depends on the operation and engine

Pick one.

### Reveal
**C) depends**.

Even "online" operations can:

- Take locks that block writes
- Rewrite tables (copy-on-write / heap rewrite)
- Stall replication (WAL volume)
- Cause IO spikes

### [IMAGE: Timeline — DDL + replica lag]
**Diagram description (include in your visuals):**
- A horizontal time axis.
- Primary DB receives writes continuously.
- At `t1`, an `ALTER TABLE` acquires a lock (label lock type, e.g., `ACCESS EXCLUSIVE` vs `SHARE UPDATE EXCLUSIVE` depending on operation).
- Replicas apply WAL later; show a shaded window where **primary has schema S2** but **replica still has schema S1**.
- Label the shaded region: **"Schrödinger’s schema window"**.

### Real-world analogy (single-lane bridge)
Your database is a busy bridge. Some maintenance can happen on the shoulder (online), but certain repairs require closing a lane (locking). Traffic jams propagate backward (timeouts, retries, queue buildup).

### Key insight
> DDL isn't just a schema event; it's a **cluster-wide performance and consistency event**.

**Challenge question:** How can replica lag create "Schrödinger's schema" for read traffic?

Answer: some requests routed to replicas see old schema/data while others routed to primary see new schema/data, producing inconsistent behavior across requests.

---

## [HANDSHAKE] Compatibility matrix: app versions x schema versions

### Scenario
You deploy `checkout` v1 -> v2 while the DB migrates from schema S1 -> S2.

### Exercise: Fill the matrix
For each cell, mark safe or unsafe.

- v1 expects only old column `total_amount`
- v2 writes `total_amount_minor` and reads it if present, otherwise falls back
- S2 includes both `total_amount` and `total_amount_minor`

|            | Schema S1 (old only) | Schema S2 (old+new) |
|------------|-----------------------|----------------------|
| App v1     | ?                     | ?                    |
| App v2     | ?                     | ?                    |

Pause and think.

### Reveal
|            | Schema S1 (old only) | Schema S2 (old+new) |
|------------|-----------------------|----------------------|
| App v1     | safe                  | safe                 |
| App v2     | safe (if coded to tolerate missing new column) | safe |

If v2 assumes the new column exists, then v2 x S1 becomes unsafe.

### Key insight
> Your application must be explicitly coded for **schema feature detection** (or tolerant reads/writes) during the migration window.

**Challenge question:** What's the equivalent of "feature flags" at the schema level?

Answer: **capability detection** (e.g., checking column existence, or using a migration state table), plus **runtime flags** controlling read/write paths.

---

## [FAUCET] Pattern 1: Additive columns with backfill (the classic)

### Scenario
Add `currency` and `fx_rate` to `order_lines`.

### Steps (Expand -> Migrate -> Contract)
1. Expand: Add nullable columns.
2. Deploy app that writes new columns for new orders.
3. Backfill historical rows in controlled batches.
4. Deploy app that reads new columns.
5. Enforce constraints (NOT NULL, CHECK).
6. Contract: Remove old semantics/columns.

### Production DDL guardrails (Postgres)
- Use `lock_timeout` to avoid long blocking.
- Use `statement_timeout` to avoid runaway DDL.
- Prefer `ADD COLUMN` without default (metadata-only in modern Postgres) and backfill separately.
- Prefer `CHECK ... NOT VALID` then `VALIDATE CONSTRAINT`.

### [CODE: Python — safe-ish Postgres DDL execution with timeouts]

```python
import os
import psycopg2


def run_ddl(ddl: str) -> None:
    """Run a single DDL statement with guardrails.

    Notes:
    - Many DDL statements are transactional in Postgres, but some (e.g., CREATE INDEX CONCURRENTLY)
      are not allowed inside a transaction block.
    - This helper is for transactional DDL.
    """
    dsn = os.environ["DATABASE_URL"]
    with psycopg2.connect(dsn) as conn, conn.cursor() as cur:
        cur.execute("SET lock_timeout = '2s'")
        cur.execute("SET statement_timeout = '30s'")
        try:
            cur.execute(ddl)
        except psycopg2.Error as e:
            conn.rollback()
            raise RuntimeError(f"DDL failed safely; investigate and retry: {e.pgerror}") from e


if __name__ == "__main__":
    run_ddl("ALTER TABLE order_lines ADD COLUMN IF NOT EXISTS currency TEXT")
```

### Pause and think
Why can `NOT VALID` constraints (Postgres) be safer than immediate validation?

### Reveal
Because you can add the constraint without scanning the whole table immediately, then validate asynchronously, reducing lock time and IO spikes.

### Failure scenarios
- Backfill job saturates IO -> p99 latency spikes
- Partial backfill + new reads -> inconsistent totals
- Constraint validation fails due to dirty legacy rows

### Performance considerations
- Backfills create **write amplification** (WAL + indexes + vacuum).
- Large updates can bloat tables; plan vacuum/autovacuum capacity.
- If you update indexed columns, you pay index maintenance cost.

### Key insight
> Backfill is a **distributed workload**: it competes with production traffic and must be rate-limited, observable, and restartable.

**Challenge question:** What makes a backfill "restartable" without double-writing errors?

Answer: idempotent updates + durable checkpoints + safe retry semantics.

---

## [SIREN] Common Misconception: "Backfill is just a script"

### Misconception
"We'll write a one-off script that updates all rows."

### Reality
In production distributed systems, backfill is closer to a **data pipeline**:

- It needs checkpointing
- It must be idempotent
- It must handle retries and partial failure
- It needs metrics, alerting, and kill switches

### Pause and think
If your backfill updates 500M rows, what's your plan for:

- progress tracking?
- pausing?
- resuming after deploy?
- avoiding replication overload?

### Key insight
> Treat backfills like production jobs: **SLOs, throttles, and abort paths**.

**Challenge question:** What's the simplest checkpoint key you can use for a table backfill?

Answer: a monotonic, indexed key (often the primary key), plus a job name/version.

---

## [MAGNIFIER] Pattern 2: Dual-write with read-repair

### Scenario
You're moving from `orders.total_amount` to `orders.total_amount_minor` and want to keep them consistent.

### Mental model
Dual-write is like writing the same message to two delivery companies at once. If one delivery fails, you must detect and reconcile.

### Protocol
1. Expand: add new column
2. Deploy app that writes both old and new
3. Deploy app that reads new, falls back to old if missing
4. Backfill old rows to new
5. Add guardrails (constraints, monitoring)
6. Contract: stop writing old, then drop old

### Pause and think
Dual-write sounds safe. What's the hardest part?

- A) writing both fields
- B) guaranteeing atomicity across both representations
- C) removing the old field

Pick one.

### Reveal
Usually B.

If you write two columns in the same row in the same transaction, you get atomicity at the row level.

But if you dual-write across two tables or two databases, atomicity becomes distributed.

### Failure scenarios
- Partial write: new field set, old field not set (or vice versa)
- Different services compute totals differently
- Race conditions with concurrent updates

### [IMAGE: Sequence diagram — partial dual-write + repair]
**Diagram description (include in your visuals):**
- Actors: `checkout v1`, `checkout v2`, `DB`.
- `v2` writes both columns in one transaction (success).
- A failure path where `v2` writes old column, then crashes before writing new (if not in same transaction / across systems).
- A read path that detects mismatch/missing and triggers repair (either synchronous read-repair or async repair queue).
- Label the repair decision: **"serve old but enqueue repair"** vs **"block and retry"**.

### Key insight
> Dual-write is safe only when you have a **reconciliation strategy** (read-repair, async repair, or periodic consistency checks).

**Challenge question:** What's the difference between read-repair and backfill?

Answer:
- Backfill is a planned batch process to populate missing data.
- Read-repair is opportunistic correction triggered by reads (often for long-tail or missed rows).

---

## [GAME] Decision game: Dual-write across systems

### Scenario
You're migrating from Postgres to a new distributed SQL cluster. For months you will write to both.

Which statement is true?

1. "If we write to both in the same request, we get atomicity."
2. "If we use a distributed transaction (2PC), we get atomicity but may reduce availability."
3. "If we use async replication, we get atomicity and high availability."

Pause and think.

### Reveal
2 is the most accurate.

- Writing to both in one request is not atomic unless you have a transaction spanning both.
- Async replication trades consistency for availability.

### CAP trade-off callout
- 2PC increases coordination and can turn transient partitions into user-visible failures.
- If you require availability, you typically accept eventual consistency + reconciliation.

**Challenge question:** When is 2PC acceptable, and when is it a trap?

Guideline:
- Acceptable for low-QPS, high-value operations where correctness dominates and latency budget allows.
- A trap for high-QPS hot paths where coordinator/participant failures will cascade into outages.

---

## [FAUCET] Pattern 3: Shadow tables (copy + cutover)

### Scenario
You need to change a table's primary key or partitioning scheme—operations that are expensive in place.

### Approach
1. Create `orders_v2` with new schema
2. Start writing new rows to both `orders` and `orders_v2` (or capture changes via CDC)
3. Backfill historical data into `orders_v2`
4. Verify consistency (counts, checksums, sample queries)
5. Cut over reads to `orders_v2`
6. Stop writes to old table
7. Drop old table later

### Real-world analogy (new kitchen prep line)
You build a new prep line next to the old one. For a while, every order is prepared on both lines to validate the new process. Once proven, you route all orders to the new line.

### [CODE: Node.js — dual-write to shadow table with idempotency]

```javascript
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function createOrderDualWrite(order) {
  // One transaction => atomic across both tables (same DB).
  // Use an idempotency key (order.id) so retries are safe.
  const client = await pool.connect();
  try {
    await client.query("BEGIN");

    await client.query(
      "INSERT INTO orders(id, total_amount) VALUES($1,$2) ON CONFLICT (id) DO NOTHING",
      [order.id, order.total_amount]
    );

    await client.query(
      "INSERT INTO orders_v2(id, total_amount_minor, currency) VALUES($1,$2,$3) ON CONFLICT (id) DO NOTHING",
      [order.id, order.total_amount_minor, order.currency]
    );

    await client.query("COMMIT");
  } catch (e) {
    await client.query("ROLLBACK");
    throw new Error(`dual-write failed; safe to retry: ${e.message}`);
  } finally {
    client.release();
  }
}
```

### Pause and think
What's your cutover risk if you "swap tables" instantly?

### Reveal
Instant cutover can cause:

- cache stampedes
- query plan regressions
- read replica inconsistencies
- application bugs triggered by new constraints

A safer cutover is progressive: route a small percentage of reads first.

### Key insight
> Shadow table migrations are powerful, but you must treat cutover like a **traffic migration** (canary, progressive rollout, rollback plan).

**Challenge question:** How do you rollback after cutover if writes already went to the new table?

Answer: you usually rollback by **routing** (feature flag / traffic shift) while keeping dual-write on, not by reverting the DB. If you must revert, you need a reverse replication/backfill plan.

---

## [MAGNIFIER] Pattern 4: Online index changes and query plan safety

### Scenario
You add an index to support a new query, but index build can be heavy. In distributed systems, index builds can also impact replication and storage.

### Pause and think
Which is safer for large Postgres tables?

- A) `CREATE INDEX ...;`
- B) `CREATE INDEX CONCURRENTLY ...;`

### Reveal
Usually B: `CONCURRENTLY` reduces blocking but takes longer and has different failure modes.

### [CODE: Python — create index concurrently with cleanup on failure]

```python
import os
import psycopg2


def create_index_concurrently(index_name: str, ddl: str) -> None:
    dsn = os.environ["DATABASE_URL"]
    # CONCURRENTLY cannot run inside a transaction block.
    with psycopg2.connect(dsn) as conn, conn.cursor() as cur:
        conn.autocommit = True
        try:
            cur.execute("SET statement_timeout = '0'")  # allow long build
            cur.execute(ddl)
        except psycopg2.Error as e:
            # Failed concurrent builds can leave INVALID indexes; drop to allow retry.
            try:
                cur.execute(f"DROP INDEX CONCURRENTLY IF EXISTS {index_name}")
            except psycopg2.Error:
                pass
            raise RuntimeError(
                f"index build failed; cleaned up {index_name}: {e.pgerror}"
            ) from e


if __name__ == "__main__":
    create_index_concurrently(
        "idx_orders_created_at",
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_created_at ON orders(created_at)",
    )
```

### Failure scenarios
- Invalid index left behind after failure
- Increased WAL volume -> replica lag
- Planner flips to new index unexpectedly -> latency regression

### Performance considerations
- Index builds compete for IO and CPU; they can starve OLTP.
- New indexes increase write cost (every insert/update pays).

### Key insight
> Index changes are performance migrations. Treat them like production rollouts with metrics and rollback.

**Challenge question:** What metrics would you watch immediately after an index appears?

Examples:
- p95/p99 latency for affected endpoints
- query latency and rows scanned (pg_stat_statements)
- buffer hit rate / IO utilization
- replica lag / WAL rate

---

## [HANDSHAKE] Distributed failure modes you must design for

### Scenario
Your migration plan looks perfect in staging. Production fails anyway.

### Matching exercise: failure -> mitigation
Match each failure mode to a mitigation.

Failures:
A. Replica lag causes reads to miss new column
B. Old service writes rows without new required field
C. Backfill job overloads DB
D. CDC consumer crashes on schema change
E. Partial dual-write across databases

Mitigations:
1. Feature-detect schema; fallback reads
2. Add nullable column first; enforce later
3. Throttle + chunk + sleep; run off-peak; use adaptive rate
4. Schema registry + compatibility rules; versioned events
5. Outbox pattern + async reconciliation; or 2PC if required

Pause and think.

### Reveal
A->1, B->2, C->3, D->4, E->5

### Key insight
> In distributed systems, schema changes propagate at different speeds to different components.

**Challenge question:** Which component is often forgotten in migration plans: OLTP DB, replicas, caches, CDC, or analytics?

Answer: often **CDC + downstream consumers** (and caches).

---

## [SIREN] Common Misconception: "Our ORM will handle it"

### Misconception
"We use an ORM. Schema migrations are safe."

### Reality
ORMs help generate DDL, but they don't automatically solve:

- mixed-version deployments
- long-running locks
- backfill throttling
- CDC compatibility
- operational rollback

### Pause and think
What's one ORM behavior that can be dangerous during migration?

### Reveal
Auto-generated migrations might:

- drop and recreate columns
- rebuild tables
- create indexes non-concurrently
- rename columns in a way that breaks old code

### Production policy suggestion
- Require human review for any migration that:
  - rewrites a large table
  - adds `NOT NULL` / `UNIQUE` / `FOREIGN KEY` without a phased plan
  - creates indexes without `CONCURRENTLY` (Postgres)
  - uses `SELECT *` in critical queries

---

## [PUZZLE] Event-driven systems: schema changes beyond the database

### Scenario
You publish `OrderCreated` events to Kafka. Multiple teams consume them.

You add `currency` and change amount semantics.

### Pause and think
Is a DB migration enough?

### Reveal
No. In distributed systems, **data contracts** exist at multiple layers:

- DB schema
- Event schema (Kafka/Protobuf/Avro/JSON)
- Cache keys and payloads
- Search indexes
- Materialized views

### [IMAGE: Layered contract diagram]
**Diagram description (include in your visuals):**
- Layers: `DB` -> `Service` -> `Event Stream` -> `Consumers` -> `Cache` / `Search` / `Warehouse`.
- Annotate each arrow with a consistency model (e.g., DB tx = strong; events = at-least-once; cache = best-effort).
- Highlight that schema/semantics must be compatible across all layers.

### Pattern: schema registry + compatibility
If using Avro/Protobuf:

- enforce backward/forward compatibility rules
- version events
- allow additive fields with defaults

**Important Protobuf note:** never reuse tag numbers; removing fields requires reserving tags.

### [CODE: Node.js — tolerant event consumer with defaults]

```javascript
import net from "node:net";

export function startOrderEventConsumer({ host = "127.0.0.1", port = 9000 }) {
  // Concept: consumers must tolerate missing/new fields (forward/backward compat).
  const sock = net.createConnection({ host, port });
  sock.setEncoding("utf8");

  sock.on("data", (chunk) => {
    for (const line of chunk.split("\n").filter(Boolean)) {
      try {
        const evt = JSON.parse(line);
        const currency = evt.currency ?? "USD"; // default for older producers
        const amountMinor = evt.total_amount_minor ?? evt.total_amount; // fallback
        if (amountMinor == null) throw new Error("missing amount");
        console.log("OrderCreated", { id: evt.id, currency, amountMinor });
      } catch (e) {
        // In production: send to DLQ with context, do not spin.
        console.error("bad event; send to DLQ", e.message);
      }
    }
  });

  sock.on("error", (e) => console.error("consumer socket error", e.message));
}
```

### Key insight
> Zero-downtime migrations require **contract evolution** across all data planes, not just the database.

**Challenge question:** If you change semantics (not just fields), what kind of compatibility is that?

Answer: it’s a **semantic breaking change** even if the schema is backward-compatible. You need versioning at the meaning level (new field name, new event version, or explicit `amount_type`).

---

## [FAUCET] Pattern 5: Expand/Contract for caches and derived stores

### Scenario
You store `order_total` in Redis for fast reads, and also in Elasticsearch for search.

You change the meaning of totals.

### Pause and think
What's the risk if you backfill DB but forget Redis?

### Reveal
You get "split brain" at the application layer:

- DB returns new totals
- cache returns old totals
- users see inconsistent UI depending on cache hit

### Approach
- Version cache keys: `order:v1:{id}` -> `order:v2:{id}`
- Dual-populate during migration
- Cut over reads gradually
- Let old keys expire

### [CODE: Python — cache key versioning + fallback reads]

```python
import json
from typing import Optional
import redis

r = redis.Redis.from_url("redis://localhost:6379/0", decode_responses=True)


def get_order_total_minor(order_id: str) -> Optional[int]:
    # Read-new-first, fallback-old to avoid user-visible misses during cutover.
    for key in (f"order:v2:{order_id}", f"order:v1:{order_id}"):
        try:
            raw = r.get(key)
        except redis.RedisError as e:
            raise RuntimeError(f"redis read failed: {e}") from e
        if raw:
            payload = json.loads(raw)
            # v1 payload might not have total_amount_minor; handle both.
            return payload.get("total_amount_minor") or payload.get("total_amount")
    return None


def set_order_total_v2(order_id: str, total_minor: int, ttl_s: int = 3600) -> None:
    payload = json.dumps({"total_amount_minor": total_minor})
    try:
        r.setex(f"order:v2:{order_id}", ttl_s, payload)
    except redis.RedisError as e:
        raise RuntimeError(f"redis write failed: {e}") from e
```

### Key insight
> Caches are schemas too—just implicit ones.

**Challenge question:** Why is "delete all cache keys" often not an acceptable migration plan?

Answer: it can cause cache stampedes, overload downstream systems, and create user-visible latency spikes.

---

## [MAGNIFIER] Operational discipline: feature flags, phased deploys, and kill switches

### Scenario
You've added new columns and deployed dual-write. Now you need to turn on "read-new" safely.

### Pause and think
What's safer?

- A) Deploy code that immediately reads the new column
- B) Deploy code that can read new, but keep it off behind a flag; enable gradually

### Reveal
B.

### Recommended controls
- Feature flag for read path
- Feature flag for write path (dual-write on/off)
- Kill switch for backfill job
- Rate limiters
- Canary analysis

### Production insight
Treat flags as part of the migration protocol:
- flags must be **fast to flip**
- flags must be **audited** (who flipped, when)
- flags must have **safe defaults**

---

## [PUZZLE] Backfill engineering: idempotency, chunking, and correctness

### Scenario
You must backfill `currency` for 800M order lines.

### Pause and think
Design a backfill loop. What do you page by?

- OFFSET/LIMIT
- primary key range
- timestamp

Choose one.

### Reveal
Prefer primary key range (or another indexed, monotonic key). OFFSET becomes slower as you progress.

### [CODE: Python — restartable backfill with checkpoints (fixed correctness)]

Key fixes vs the earlier naive version:
- don’t use `cursor.rowcount` after multiple statements (it only reflects the last statement)
- track updated rows explicitly
- keep checkpoint updates consistent with work done

```python
import os
import time
import psycopg2

BATCH = 10_000
JOB = "currency_v1"  # version your backfill jobs


def backfill_currency() -> None:
    dsn = os.environ["DATABASE_URL"]
    with psycopg2.connect(dsn) as conn, conn.cursor() as cur:
        cur.execute(
            "CREATE TABLE IF NOT EXISTS backfill_ckpt("
            "job TEXT PRIMARY KEY, last_id BIGINT NOT NULL)"
        )
        cur.execute(
            "INSERT INTO backfill_ckpt(job,last_id) VALUES(%s,0) "
            "ON CONFLICT (job) DO NOTHING",
            (JOB,),
        )
        conn.commit()

        while True:
            cur.execute("SELECT last_id FROM backfill_ckpt WHERE job=%s", (JOB,))
            last_id = int(cur.fetchone()[0])
            hi = last_id + BATCH

            try:
                cur.execute("BEGIN")
                cur.execute(
                    "UPDATE order_lines "
                    "SET currency='USD' "
                    "WHERE currency IS NULL AND id > %s AND id <= %s",
                    (last_id, hi),
                )
                updated = cur.rowcount

                # Advance checkpoint monotonically regardless of updated count.
                # This avoids getting stuck on sparse ranges.
                cur.execute(
                    "UPDATE backfill_ckpt SET last_id=%s WHERE job=%s",
                    (hi, JOB),
                )
                cur.execute("COMMIT")
            except psycopg2.Error:
                cur.execute("ROLLBACK")
                time.sleep(1.0)
                continue

            # Termination condition: if we updated nothing for a while, we might be done.
            # In production, prefer a known max(id) snapshot or a "remaining NULL" query.
            if updated == 0:
                # Check if any NULLs remain beyond current checkpoint.
                cur.execute("SELECT 1 FROM order_lines WHERE currency IS NULL LIMIT 1")
                if cur.fetchone() is None:
                    break

            time.sleep(0.05)  # throttle; tune using p99/replica lag


if __name__ == "__main__":
    backfill_currency()
```

### Correctness techniques
- Idempotent update: `... WHERE new_col IS NULL AND id BETWEEN ...`
- Checkpoint table: store last processed id
- Adaptive throttling based on DB latency and replica lag
- Use `FOR UPDATE SKIP LOCKED` if multiple workers claim work from a queue/table

### Failure scenarios
- Worker crashes mid-transaction (checkpoint not advanced; safe retry)
- Hot partitions cause uneven progress
- Dead tuples bloat storage (vacuum pressure)

### Key insight
> Backfills must be **safe to retry** and **safe to run in parallel**.

**Challenge question:** What does "exactly-once" mean for a backfill update, and do you actually need it?

Answer: exactly-once would mean each row is updated once and only once. For most backfills you don’t need exactly-once; you need **idempotent at-least-once** (safe retries) with correctness invariants.

---

## [GAME] Quiz: Choose the correct statement (locks & DDL)

Which statement is true?

1. "Adding a nullable column is always instant and lock-free."
2. "Adding a column with a default is always instant in modern Postgres."
3. "Renaming a column is always safe for ORMs and cached prepared statements."
4. "Even metadata-only changes can break clients if they depend on column order or SELECT *."

Pause and think.

### Reveal
4 is reliably true.

1 and 2 depend on engine/version and may still take locks.
3 can break clients that use cached query plans, `SELECT *`, or ORM field mapping assumptions.

### Key insight
> The risk is not just the DB; it's **every client's assumptions**.

**Challenge question:** Why is `SELECT *` a migration hazard?

Answer: column order and presence can change; clients may deserialize positionally or assume a fixed set of fields.

---

## [SIREN] Common Misconception: "We can always roll back"

### Misconception
"If something goes wrong, we'll roll back."

### Reality
Rollbacks become hard when:

- you've backfilled data into new schema
- you've dropped old columns
- you've changed semantics (meaning) not just structure
- you've published events consumed downstream

### Pause and think
What's the rollback plan for a semantic change?

Example: `total_amount` used to mean "pre-tax," now means "post-tax."

### Reveal
You may need:

- dual fields (`total_pre_tax`, `total_post_tax`)
- versioned events
- read-path flags
- a "recompute" pipeline

Rollback might mean forward-fixing rather than reverting.

### Key insight
> In data migrations, rollback is often another migration.

**Challenge question:** What must you preserve to make rollback possible: old schema, old data, or old semantics?

Answer: usually **old semantics** (or a way to recompute them). Schema alone is not enough.

---

## [MAGNIFIER] Contract changes: renames, type changes, and semantic changes

### Scenario
You want to:

- rename `user_id` -> `customer_id`
- change `amount` from `INT` cents to `DECIMAL`

### Pause and think
Which is easiest to do zero-downtime?

- A) rename
- B) type change
- C) semantic change

### Reveal
Often A looks easy but can still break clients; B can be expensive; C is the hardest because it affects correctness.

### Safer rename pattern
- Add new column `customer_id`
- Dual-write `user_id` and `customer_id`
- Backfill
- Switch reads
- Drop old

### Type change pattern
- Add new column with new type
- Dual-write with conversion
- Validate
- Switch reads
- Drop old

### [CODE: Node.js — schema feature detection (safer)]

Production fixes vs naive approach:
- `information_schema` can be slow; prefer `to_regclass` / `pg_attribute` in Postgres.
- schema can differ between primary and replica; cache with TTL and handle runtime errors.

```javascript
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

let hasNewCol = null;
let hasNewColCheckedAtMs = 0;
const SCHEMA_CACHE_TTL_MS = 60_000;

async function detectNewColumn() {
  const now = Date.now();
  if (hasNewCol !== null && now - hasNewColCheckedAtMs < SCHEMA_CACHE_TTL_MS) return hasNewCol;

  // Postgres-specific: check attribute existence quickly.
  const q = `
    SELECT 1
    FROM pg_attribute a
    JOIN pg_class c ON c.oid = a.attrelid
    JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE n.nspname = 'public'
      AND c.relname = 'orders'
      AND a.attname = 'total_amount_minor'
      AND a.attnum > 0
      AND NOT a.attisdropped
    LIMIT 1;
  `;

  const { rowCount } = await pool.query(q);
  hasNewCol = rowCount > 0;
  hasNewColCheckedAtMs = now;
  return hasNewCol;
}

export async function fetchOrderAmount(orderId) {
  const newCol = await detectNewColumn();
  const sql = newCol
    ? "SELECT total_amount_minor AS amount FROM orders WHERE id=$1"
    : "SELECT total_amount AS amount FROM orders WHERE id=$1";

  try {
    const { rows } = await pool.query(sql, [orderId]);
    return rows[0]?.amount ?? null;
  } catch (e) {
    // If replica lag causes schema mismatch, retry with old path once.
    if (newCol && /column .*total_amount_minor.* does not exist/i.test(e.message)) {
      hasNewCol = false; // degrade quickly
      const { rows } = await pool.query("SELECT total_amount AS amount FROM orders WHERE id=$1", [orderId]);
      return rows[0]?.amount ?? null;
    }
    throw new Error(`order read failed (schema mismatch/replica lag?): ${e.message}`);
  }
}
```

### Key insight
> Avoid in-place destructive transformations; prefer **parallel representation**.

**Challenge question:** When would you still choose an in-place type change?

Answer: when the table is small, traffic is low, you can tolerate a maintenance window, or the DB provides a truly online type change with bounded locks—and you’ve tested it under production-like load.

---

## [HANDSHAKE] Multi-service coordination: who owns the migration?

### Scenario
Three services write to the same table:

- `checkout` writes orders
- `refunds` updates status and amounts
- `fraud` annotates risk fields

### Pause and think
Who should drive the migration timeline?

- A) DB team
- B) The service with the most traffic
- C) A cross-team migration owner with an explicit plan

### Reveal
C.

You need a single migration plan with:

- sequencing
- compatibility guarantees
- ownership of backfill
- cutover criteria
- rollback plan

### Real-world analogy (airport runway maintenance)
Multiple airlines (services) use the same runway (table). Maintenance requires a coordinated schedule, not unilateral changes.

### Key insight
> Shared databases create **organizational coupling**. Zero-downtime migrations are as much coordination as engineering.

**Challenge question:** What artifact should exist for every migration: a ticket, a runbook, or a design doc?

Answer: all three, but minimally a **design doc + runbook** for anything that can impact availability/correctness.

---

## [FAUCET] Observability for migrations: what to measure

### Scenario
You start a backfill and dual-write. How do you know it's safe?

### Metrics to watch
- DB: lock waits, deadlocks, replication lag, WAL volume, buffer/cache hit, CPU/IO
- App: error rates by version, p95/p99 latency, retry rates, queue depths
- Data correctness: mismatch counters between old and new fields
- Backfill: rows/sec, ETA, failure rate, checkpoint lag

### [IMAGE: Migration dashboard mock]
**Diagram description (include in your visuals):**
- Panels: `replication_lag_seconds`, `lock_waits`, `p99_latency`, `backfill_rows_per_sec`, `backfill_checkpoint`, `mismatch_rate`.
- Add alert thresholds (e.g., lag > 30s, mismatch > 0.01%).

### Pause and think
What's the single most important correctness metric during dual-write?

### Reveal
A mismatch rate (or checksum discrepancy) between old and new representations.

### Key insight
> You can't observe correctness indirectly via latency. You must instrument it.

**Challenge question:** Where do you compute mismatches: in the app, in SQL queries, or in offline jobs?

Answer: ideally all three at different cadences—fast signal in-app, authoritative checks in SQL/offline.

---

## [PUZZLE] Cutover strategies: dark reads, canaries, and progressive delivery

### Scenario
You have `orders_v2` shadow table fully backfilled.

### Strategies
- Dark read: read from v2 but don't serve it; compare to v1
- Canary: serve v2 to 1% of users
- Progressive: ramp 1% -> 10% -> 50% -> 100%

### Pause and think
Which is safer first: dark reads or canary?

### Reveal
Dark reads first: you validate correctness without user impact.

### [CODE: Python — dark reads with mismatch logging]

```python
import logging
from concurrent.futures import ThreadPoolExecutor, TimeoutError

log = logging.getLogger("darkread")
executor = ThreadPoolExecutor(max_workers=8)


def fetch_v1(order_id: str) -> int:  # placeholder
    return 123


def fetch_v2(order_id: str) -> int:  # placeholder
    return 123


def handle_request(order_id: str) -> int:
    served = fetch_v1(order_id)
    fut = executor.submit(fetch_v2, order_id)
    try:
        shadow = fut.result(timeout=0.05)
        if shadow != served:
            log.warning("mismatch order_id=%s v1=%s v2=%s", order_id, served, shadow)
    except TimeoutError:
        log.info("shadow read timed out order_id=%s", order_id)
    except Exception as e:
        log.exception("shadow read failed order_id=%s err=%s", order_id, e)
    return served
```

### Key insight
> Treat data cutovers like traffic routing: validate silently, then gradually expose.

**Challenge question:** What's your rollback lever during canary: routing, feature flag, or DB revert?

Answer: almost always **routing/feature flag**.

---

## [SIREN] Common Misconception: "Replication makes it consistent everywhere"

### Misconception
"Our replicas will catch up quickly; it's fine."

### Reality
During migrations, replication lag can spike due to:

- backfill write amplification
- index builds
- vacuum pressure
- network hiccups

Applications reading from replicas may see:

- old schema
- missing backfilled data
- inconsistent constraints

### Pause and think
What's the safest read strategy during critical migration phases?

### Reveal
Often:

- route critical reads to primary
- or ensure new app version can tolerate replica staleness
- or temporarily reduce replica read traffic

### Key insight
> Replicas are not just "slower primaries"; they're different timelines.

**Challenge question:** How could you detect schema mismatch errors caused by replica lag?

Answer: track error signatures like `column does not exist`, correlate with replica host, and monitor lag metrics; add structured logs with DB endpoint identity.

---

## [MAGNIFIER] Advanced topic: migrations with sharding and partitioning

### Scenario
Your `orders` table is sharded by `customer_id`. Some shards are busier.

You need to add a column and backfill.

### Pause and think
What's different vs single-node?

### Reveal
- You must coordinate schema change across shards
- Backfill load is uneven (hot shards)
- Cutover may need shard-by-shard progress
- Failures may be partial (some shards migrated, others not)

### Tactics
- shard-by-shard rollout
- per-shard throttles
- global progress tracking
- avoid cross-shard transactions during dual-write

### [IMAGE: Sharded rollout state diagram]
**Diagram description (include in your visuals):**
- Shards `A..N` each labeled with state: `S1`, `S1+S2`, `S2`.
- A coordinator component tracking per-shard progress and gating cutover.

### Key insight
> In sharded systems, mixed versions happens at the shard level too.

**Challenge question:** Would you rather migrate all shards at once or one shard at a time? Why?

Answer: usually one shard at a time (or small batches) to limit blast radius and learn from early shards.

---

## [FAUCET] Advanced topic: online schema changes in MySQL and friends

### Scenario
You run MySQL with large tables. Some ALTER operations copy tables.

### Pause and think
What's a common zero-downtime approach in MySQL ecosystems?

### Reveal
Tools/patterns like:

- `pt-online-schema-change`
- `gh-ost`

They create a shadow table, copy data, and apply changes via triggers/binlog.

### Production risk note (important)
Trigger-based copy under heavy write load can:
- add significant write latency
- increase replication lag
- create drift if triggers fail or are disabled

### Key insight
> Some databases require external tooling to approximate online DDL safely.

**Challenge question:** What's the main risk of trigger-based copy during heavy write load?

Answer: it amplifies write cost and can fall behind, creating lag and potentially inconsistent cutover if drift accumulates.

---

## [PUZZLE] Data correctness: invariants, checksums, and reconciliation

### Scenario
You dual-write amounts in old and new formats.

### Pause and think
How do you prove correctness before contract?

### Techniques
- Invariant queries: `COUNT(*) WHERE old != convert(new)`
- Sampling: compare for random IDs
- Checksums per partition
- Reconciliation jobs

### [CODE: Python — mismatch counter query + alert threshold]

```python
import os
import psycopg2


def count_mismatches() -> int:
    dsn = os.environ["DATABASE_URL"]
    q = (
        "SELECT COUNT(*) FROM orders "
        "WHERE total_amount_minor IS NOT NULL "
        "AND total_amount IS NOT NULL "
        "AND total_amount_minor <> total_amount"
    )
    with psycopg2.connect(dsn) as conn, conn.cursor() as cur:
        cur.execute(q)
        return int(cur.fetchone()[0])


if __name__ == "__main__":
    mismatches = count_mismatches()
    if mismatches > 0:
        raise SystemExit(f"BLOCK CONTRACT: mismatches={mismatches}")
```

### Key insight
> "Looks fine" is not a correctness proof. You need explicit invariants.

**Challenge question:** What's the trade-off between full-table checksums and sampling?

Answer: full checksums are expensive but comprehensive; sampling is cheap but can miss rare edge cases.

---

## [GAME] Matching exercise: choose the right pattern

### Scenario
Match the migration goal to the best pattern.

Goals:
1. Add a column used by new feature
2. Change primary key
3. Move data to a new database
4. Change semantics of a field
5. Add a large index without blocking writes

Patterns:
A. Add nullable + backfill + enforce constraint
B. Shadow table + cutover
C. Dual-write across systems + reconciliation
D. Parallel fields + versioned semantics
E. Online index build + canary query plan

Pause and think.

### Reveal
1->A, 2->B, 3->C, 4->D, 5->E

### Key insight
> Different migration goals require different risk envelopes.

**Challenge question:** Which of these patterns is hardest to roll back cleanly?

Answer: cross-system dual-write (C) and semantic changes (D) are typically hardest.

---

## [MAGNIFIER] Contract phase: how to safely drop old schema

### Scenario
You've been dual-writing for weeks. You want to drop old columns.

### Pause and think
What must be true before contract?

Write a checklist.

### Reveal: contract readiness checklist
- No services read old fields (verified via code search + runtime metrics)
- No services write old fields
- Backfill completed and verified
- Mismatch rate near zero for a sustained period
- CDC consumers updated
- Dashboards and alerts in place
- Rollback plan defined (often forward-fix)

### Safe contract tactics
- Stop writing old field first (keep it for rollback reads)
- Wait one full deploy cycle
- Then drop old field

### [CODE: Node.js — guarded contract step (drop column) with lock timeout]

```javascript
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function dropOldColumnSafely() {
  const ddl = "ALTER TABLE orders DROP COLUMN IF EXISTS total_amount";
  const client = await pool.connect();
  try {
    await client.query("SET lock_timeout = '2s'");
    await client.query("SET statement_timeout = '30s'");

    // Guardrail: require an env var to be set in the runbook step.
    if (process.env.ALLOW_CONTRACT !== "true") throw new Error("contract not allowed");

    await client.query(ddl);
  } catch (e) {
    throw new Error(`contract DDL failed safely; retry later: ${e.message}`);
  } finally {
    client.release();
  }
}
```

### Key insight
> Contract is where you pay down complexity—but it's also where you can permanently break rollback.

**Challenge question:** Why do teams often postpone contract forever, and what's the cost?

Answer: fear of breaking something + lack of ownership. Cost: schema bloat, ongoing dual-write bugs, higher storage/index costs, slower queries, and cognitive load.

---

## [SIREN] Common Misconception: "We can leave both forever"

### Misconception
"We'll keep old and new columns indefinitely. No need to drop."

### Reality
Long-lived dual schemas cause:

- developer confusion
- bugs from writing only one side
- schema bloat and slower queries
- higher storage and index costs
- unclear source of truth

### Key insight
> Contract is not optional; it's how you restore simplicity.

**Challenge question:** What governance process ensures contract actually happens?

Answer: a migration RFC/runbook with an explicit contract date, plus an owner and an SLO/OKR to remove deprecated schema.

---

## [PUZZLE] Case study walkthrough: changing order totals safely

### Scenario
You must change from `orders.total_amount` (INT cents) to:

- `orders.total_amount_minor` (BIGINT)
- `orders.currency` (TEXT)

And you must update events and caches.

### Phase 0: design the invariants
- `total_amount_minor >= 0`
- `currency in (supported set)`
- `total_amount_minor == total_amount` for USD legacy orders (if applicable)

### Phase 1: expand
- Add new columns nullable
- Add check constraints `NOT VALID`
- Update event schema (add optional fields)
- Version cache keys

### Phase 2: deploy tolerant code
- Writes: dual-write old + new
- Reads: still read old (flag off)
- Emit events with both fields

### Phase 3: backfill
- Run restartable job
- Throttle
- Monitor lag and mismatch

### Phase 4: dark reads
- Compare computed totals from new columns
- Log mismatches

### Phase 5: cutover reads
- Enable flag for 1% -> 10% -> 100%
- Monitor correctness + latency

### Phase 6: enforce constraints
- Validate check constraints
- Set NOT NULL

### Phase 7: contract
- Stop writing old
- Wait
- Drop old
- Remove v1 cache keys
- Deprecate old event fields (long tail)

### [IMAGE: End-to-end migration timeline]
**Diagram description (include in your visuals):**
- Phases 0–7 on a timeline.
- Mark deploy points, backfill window, dark read window, canary ramp, contract.
- Annotate "rollback lever" at each phase (usually feature flags / routing).

### Key insight
> A "simple" column change becomes a distributed choreography across DB, services, events, caches, and ops.

**Challenge question:** Which phase would you schedule during the lowest traffic period, and why?

Answer: backfill and constraint validation (and sometimes index builds) because they are IO-heavy and can increase lag.

---

## [TARGET] Final synthesis challenge: design your own zero-downtime migration plan

### Scenario
You run a ride-sharing platform.

You need to migrate from:

- `trips` table storing `pickup_lat`, `pickup_lng` as floats

to:

- a new `pickup_location` as a geospatial type
- a new index to support "nearby pickups" queries

Constraints:

- Multiple services read/write `trips`
- You have read replicas
- You publish `TripCreated` events
- You cache trip summaries in Redis
- You cannot pause writes

### Your task
Write a plan using **Expand -> Migrate -> Contract**.

Include:

1. Schema steps
2. Application deploy steps
3. Backfill strategy (idempotent + restartable)
4. Observability signals
5. Cutover strategy (dark reads/canary)
6. Rollback plan
7. Contract criteria

Pause and think. Draft it.

---

### Progressive reveal: a strong answer outline
(Compare to your plan; adjust yours.)

1. **Expand**
   - Add nullable `pickup_location` column
   - Add geospatial index concurrently (or shadow index strategy)
   - Update event schema to include optional `pickup_location`
   - Add Redis key versioning for trip summary payload

2. **Deploy tolerant code (phase A)**
   - Writers: compute and write both (`lat/lng` and `pickup_location`)
   - Readers: still read old (`lat/lng`) by default; add dark-read from new
   - Consumers: tolerate optional field

3. **Backfill**
   - Worker pages by trip id range
   - Updates only rows where `pickup_location IS NULL`
   - Throttle based on DB latency/replica lag
   - Store checkpoint per shard/partition

4. **Validate correctness**
   - Mismatch metric: distance between old lat/lng and decoded location
   - Sample queries on both representations
   - Monitor index build health, query latency

5. **Cutover reads**
   - Enable feature flag for 1% canary
   - Ramp gradually
   - Watch p99, error rates, mismatch rates

6. **Enforce constraints**
   - Validate constraints
   - Set NOT NULL when safe (if required)

7. **Contract**
   - Stop writing lat/lng
   - Wait at least one full deploy cycle
   - Drop old columns
   - Keep event fields for long tail (deprecation policy)
   - Let old Redis keys expire

8. **Rollback**
   - Disable read flag (serve old)
   - Keep dual-write until stable
   - If index causes regression, drop/disable it (plan)

### Key insight
> A good migration plan is a **protocol with verification**, not a sequence of hopeful commands.

**Final challenge question:** If you could add only one thing to reduce risk, would you add (a) dark reads, (b) mismatch metrics, or (c) throttled backfill? Defend your choice.

Production answer: **mismatch metrics**. Dark reads and throttling help, but mismatch metrics are the earliest, most actionable signal that you’re corrupting data or diverging semantics.

---

## [WAVE] Closing: your migration maturity checklist

- [ ] Expand-first mindset
- [ ] Mixed-version compatibility tested
- [ ] Backfills are production jobs (idempotent, restartable, throttled)
- [ ] Correctness metrics exist (mismatch counters)
- [ ] Cutovers are progressive (dark reads -> canary -> ramp)
- [ ] Contract is planned and enforced
- [ ] Downstream contracts (events, caches, search) are versioned

If you implement only one habit: **instrument correctness**. Latency tells you you're slow; correctness tells you you're right.
