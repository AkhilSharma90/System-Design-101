---
slug: database-federation
title: Database Federation
readTime: 20 min
orderIndex: 4
premium: false
---

# Database Federation (Distributed Systems) - Interactive TCP-Style Article

> **Audience**: advanced distributed systems engineers, architects, SREs, and platform teams.

---

##  Challenge: "One app, five databases, and a CEO demo in 30 minutes"

It is 9:30 AM. Your product has grown into multiple domains:

- **Orders** live in a sharded PostgreSQL cluster.
- **Customers** live in a separate MySQL instance owned by the CRM team.
- **Inventory** is in a legacy Oracle system.
- **Payments** are in a PCI-isolated database.
- **Analytics** are in a columnar store.

The CEO wants a **single dashboard**:

> "Show me customers with late orders where inventory is low and payment is pending."

You could:

1. **Copy everything** into a data warehouse (ETL/ELT) and query there.
2. **Build microservices** and compose results at the application layer.
3. **Federate queries** so a single query spans multiple sources.

Today's topic: **Database Federation** - how it works, why it is hard, and how to survive it in production.

### Pause and think
If your CEO needs real-time results (not yesterday's ETL snapshot), which approach becomes tempting - and why?

<details>
<summary>Progressive reveal</summary>
**Federation** becomes tempting because it promises "query across sources now" without waiting for pipelines or building new services for every join.
</details>

---

##  Partnering: What is Database Federation (and what it is not)

### Scenario
Your org has multiple databases that cannot be merged easily:

- Different owners
- Different schemas
- Different SLAs
- Different compliance boundaries

You want to query them as if they were one.

### Interactive question (pause and think)
Which statement best matches **database federation**?

A) "All data is physically copied into a single database."

B) "A system provides a unified query interface that can read from multiple databases, often pushing down parts of the query to each source."

C) "All writes go through a single leader database that replicates to followers."

<details>
<summary>Answer</summary>
**B**.
</details>

### Explanation with analogy (coffee shop chain)
Think of a **coffee shop chain**:

- Each branch has its own inventory system.
- You (HQ) want to answer: "Which branches are out of oat milk and have high foot traffic?"

**Federation** is like calling each branch, asking for their numbers, and combining the answers - rather than moving all inventory into a single warehouse database.

### Real-world parallel
- **Trino/Presto** federating queries across Hive/S3, Postgres, MySQL, Kafka.
- **PostgreSQL FDW** querying remote Postgres/MySQL/Oracle.
- **SQL Server PolyBase**.
- **GraphQL federation** (conceptually similar, but API-level).

### Key insight
> **Federation unifies access, not ownership.** You do not magically get one consistent database - you get a *coordinator* that mediates queries across multiple systems.

---

## Warning: The hidden constraint - distributed joins are expensive

### Scenario
You run:

```sql
SELECT c.id, c.name, o.id, o.created_at
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.country = 'DE' AND o.status = 'LATE';
```

But `customers` is in MySQL and `orders` is in sharded Postgres.

### Pause and think
Where will the **join** happen?

1) In MySQL
2) In Postgres
3) In the federation layer (coordinator)
4) Split across all systems

<details>
<summary>Answer</summary>
Often **3**: the federation coordinator pulls partial results from each source and joins them.

Sometimes it can push down the join to one system if both tables are accessible there (rare across heterogeneous databases), or if the federation system can create a temporary table / use semi-join strategies.
</details>

### Explanation with analogy (restaurant group)
Imagine a restaurant group:

- One kitchen makes pasta (Orders DB).
- Another kitchen makes desserts (Customers DB).

If a waiter needs to create a "combo plate" (join), they have to:

- Fetch pasta from kitchen A
- Fetch dessert from kitchen B
- Assemble on the serving counter (federation coordinator)

The counter becomes the bottleneck.

### Real-world parallel
Distributed joins may require:

- Shipping data across the network
- Materializing intermediate results
- Handling skew (one customer has 10M orders)

### Key insight
> **Federation is a networked query planner.** Your "single SQL statement" becomes a distributed execution plan with data movement.

### Challenge question
If `customers.country='DE'` filters down to 2% of customers, what optimization do you *hope* the federation engine does?

<details>
<summary>Progressive reveal</summary>
Push down the filter to the `customers` source, reduce results early, and ideally use a **semi-join** strategy (send customer IDs to the orders source) instead of pulling full tables.
</details>

---

##  Decision game: Federation vs ETL vs Services

### Scenario
You need a unified "Customer 360" view.

Pick the best approach for each requirement.

#### Which statement is true?

1) "If you need sub-second OLTP latency and strict consistency, federation is usually the best choice."

2) "If you need historical analytics with heavy aggregation, ETL into a warehouse/lake is usually better."

3) "If you need domain ownership and evolvable contracts, service APIs often beat federation."

Pause and think.

<details>
<summary>Answer</summary>
**2 and 3 are generally true. 1 is generally false.**

Federation can work for OLTP-ish reads, but strict consistency and low latency across heterogeneous systems is hard.
</details>

### Comparison table

| Approach | Data movement | Latency | Consistency | Operational cost | Best for |
|---|---:|---:|---:|---:|---|
| Federation | On-demand, query-time | Medium to High | Usually weak across sources | Medium to High | Ad-hoc cross-source queries, transitional architectures |
| ETL/ELT to warehouse/lake | Batch/streaming pipelines | Low for reads (after load) | Depends on pipeline freshness | High (pipelines plus storage) | Analytics, reporting, ML feature sets |
| Service composition | App-time calls | Medium to High | Depends on design (sagas, etc.) | Medium | Domain-aligned product flows |
| "Single database" consolidation | Upfront migration | Low | Strong | Very high migration risk | Smaller orgs, early stage |

### Key insight
> Federation is often a **bridge**: useful when you cannot consolidate yet, but you should still plan for long-term ownership boundaries.

---

##  Mental model: The federation coordinator is a "query shipping plus join factory"

### Scenario
Your federation system receives a SQL query.

### Mental model (step-by-step)
1. **Parse** query into AST.
2. **Catalog resolution**: map `customers` to MySQL, `orders` to Postgres.
3. **Cost-based planning**: decide pushdowns, join order, join algorithms.
4. **Query decomposition**: generate subqueries per source.
5. **Execution**: run subqueries concurrently.
6. **Data movement**: stream results to coordinator.
7. **Finalization**: join/aggregate/sort at coordinator.

[IMAGE: Diagram showing a federation coordinator in the center with arrows to multiple databases; query decomposed into subqueries; results streamed back; join/aggregate performed; include labels for pushdown filters, join strategy, and data movement.]

### Interactive question
Where do you expect **backpressure** to show up first when one source is slow?

A) Only at the slow source
B) Only at the coordinator
C) Throughout the pipeline (slow source to coordinator to client)

<details>
<summary>Answer</summary>
**C**. The coordinator cannot finish the query until it receives enough data; streaming pipelines propagate backpressure.
</details>

### Real-world parallel (delivery dispatcher)
It is like a delivery dispatcher:

- They assign deliveries to multiple warehouses.
- If one warehouse is delayed, the final "bundle" delivery is delayed.

### Key insight
> Federation turns database queries into **distributed dataflow**. Treat it like a distributed system: timeouts, retries, partial failures, and backpressure matter.

---

## Deep dive: How query pushdown works (and fails)

### Scenario
You query:

```sql
SELECT *
FROM orders
WHERE created_at >= now() - interval '7 days'
  AND status = 'LATE';
```

If `orders` is remote, the federation engine may push down the filter.

### Pause and think
Which of these is most likely to block pushdown?

A) The filter uses a function not supported by the remote DB.
B) The remote DB lacks an index.
C) The query has a LIMIT.

<details>
<summary>Answer</summary>
**A**. Pushdown depends on translating expressions into the remote dialect/capabilities. Missing indexes hurts performance but does not necessarily prevent pushdown.
</details>

### Clarification (production reality)
Pushdown is not binary. Many engines push down *some* predicates but not others. A single non-translatable expression can force a much larger scan + post-filter at the coordinator.

### Common Misconception
> "If federation is slow, it is because the engine is bad."

Often it is because:

- Pushdown fails due to SQL dialect differences
- Stats are missing, causing poor join order decisions
- Remote sources have unpredictable latency
- Network transfer dominates

### Matching exercise
Match the limitation to the likely symptom.

1) Cannot push down `REGEXP` filter
2) Remote source has no stats or stale stats
3) Cross-source join with high cardinality
4) Remote source rate limits queries

Symptoms:

A) Coordinator OOM or spills to disk
B) Full table scan on remote, huge data transfer
C) Unstable query plans (sometimes fast, sometimes terrible)
D) Intermittent timeouts even when data is small

Pause and think.

<details>
<summary>Answer</summary>
1->B, 2->C, 3->A, 4->D
</details>

### Key insight
> Federation performance is dominated by **what can be pushed down** and **how much data must move**.

---

## Puzzle: Join strategies in federated systems (broadcast, repartition, semi-join)

### Scenario
You need to join:

- `customers` (10M rows) in MySQL
- `orders` (2B rows) in Postgres shards

### Interactive question
If only 10k customers match your filter, which join strategy is best?

A) Pull all orders to coordinator and join
B) Pull all customers to coordinator and join
C) Semi-join: fetch matching customer IDs, send IDs to orders source, retrieve only matching orders

<details>
<summary>Answer</summary>
**C**.
</details>

### Explanation with analogy (gift basket assembly)
You are assembling gift baskets:

- Instead of shipping the entire warehouse inventory to your house (coordinator), you send a list of needed items to each warehouse and receive only those.

### Join strategy comparison table

| Strategy | What moves over network | When it works | Failure mode |
|---|---|---|---|
| Broadcast join | Small table broadcast to all workers/sources | One side tiny | "Tiny" becomes not tiny -> memory blowups |
| Repartition (shuffle) join | Both sides partitioned by join key | Large-large joins in distributed engines | Network heavy, skew problems |
| Semi-join | IDs/keys move first, then fetch matches | Highly selective filters | Key list too large -> becomes shuffle |
| Pushdown join | Join executed in one source | Same source or capable connector | Rare across heterogeneous DBs |

### Challenge question
What is the "tell" that a broadcast join is about to hurt you?

<details>
<summary>Progressive reveal</summary>
When the "small" side grows (or cardinality estimate is wrong), causing coordinator/worker memory pressure and spills.
</details>

---

## Warning: Failure scenarios - federation is where partial failures come to party

### Scenario
A federated query touches three sources:

- MySQL (customers)
- Postgres (orders)
- Oracle (inventory)

During execution:

- Oracle has a 2-second pause (GC or IO stall)
- Postgres shard 7 times out
- MySQL returns quickly

### Pause and think
What should the federation engine do?

A) Return partial results from MySQL and Oracle
B) Fail the whole query
C) Retry only the Postgres shard
D) It depends on semantics, idempotency, and query type

<details>
<summary>Answer</summary>
**D**.

For a single SQL query expecting complete results, you usually fail the query (B). Some systems support partial results or "best effort" modes, but that changes semantics.

Retries are tricky: reads are typically safe, but repeated queries can overload sources or hit inconsistent snapshots.
</details>

### Failure taxonomy (distributed-systems lens)

| Failure type | What it looks like | Why federation amplifies it | Mitigation |
|---|---|---|---|
| Slow source | Tail latency dominates | Query completion waits for slowest | Per-source timeouts, circuit breakers, caching |
| Source outage | Connector errors | Cross-source query becomes unavailable | Fallbacks, partial modes, feature flags |
| Network partition | Intermittent timeouts | Coordinator cannot distinguish slow vs dead | Adaptive timeouts, retries with jitter |
| Throttling | 429 or rate limits | Coordinator fans out many subqueries | Concurrency limits, admission control |
| Schema drift | Column missing/type change | Planner fails at runtime | Contract tests, schema registry, versioned views |

### Production insight: retries and amplification
A single user query can fan out into **N subqueries**. If you retry blindly, you can multiply load by N x retries and create a self-inflicted outage.

Practical guidance:
- Prefer **fail-fast** for interactive dashboards (tight timeouts, no retries) and **retry** only for batch jobs.
- If you do retry, use **bounded retries + jitter**, and consider **hedged requests** only when sources can tolerate it.

### Common Misconception
> "Federation is read-only, so it is safe."

Read-only systems can still:

- Take down production DBs via heavy scans
- Cause cascading failures via fan-out
- Leak sensitive data through unified access

### Key insight
> Federation concentrates risk: a single query interface can become a **blast radius multiplier**.

---

## Partnering: Consistency and isolation - what snapshot did you actually read?

### Scenario
You run a federated query that reads:

- Customer status from MySQL
- Orders from Postgres

A customer is updated mid-query.

### Pause and think
Can federation provide a **single consistent snapshot** across both databases?

A) Yes, always, because SQL
B) Yes, if both databases support transactions
C) Only with distributed transactions or special infrastructure; otherwise you get "fuzzy" reads

<details>
<summary>Answer</summary>
**C**.
</details>

### Mental model
Federation often provides **per-source consistency** (each subquery is consistent within its DB) but not **global consistency** across sources.

It is like asking two cashiers for receipts at slightly different times: each receipt is valid, but they may not align.

### Real-world parallel
- Trino queries are not globally serializable across sources.
- FDWs can participate in local transactions, but cross-DB snapshot isolation is not guaranteed without two-phase commit and compatible engines.

### CAP + consistency note (rigor)
Federation is inherently **partition-tolerant** (the network can fail). Under partitions, you choose between:
- **Availability**: return partial/best-effort/stale answers.
- **Consistency**: fail the query (or block) to avoid incorrect answers.

Most production federated query endpoints choose **CP-ish behavior per query** (fail rather than lie) for correctness, but may offer an explicit **AP-ish mode** for exploratory analytics.

### Trade-off box

> **Trade-off**: global consistency vs availability and simplicity.

To get global consistency you might need:

- Distributed transactions (2PC) and compatible participants
- Or change data capture plus log-based materialization into a single store

### Challenge question
If your query is used for **billing**, is "fuzzy snapshot" acceptable?

<details>
<summary>Progressive reveal</summary>
Usually no. Billing tends to require strong correctness guarantees; federation may be unsuitable unless you build a consistent materialized view or enforce write ordering.
</details>

---

## Challenge: Federation architectures - three common patterns

### Scenario
Your team is choosing a federation approach.

### Pattern 1: Query engine federation (Trino/Presto-like)
- Coordinator plus workers
- Connectors to sources
- Distributed execution, parallelism

### Pattern 2: Database-level federation (FDW or linked servers)
- One DB queries another via foreign tables
- Often simpler, closer to SQL semantics
- Limited optimization across heterogeneous sources

### Pattern 3: Virtualization layer or semantic layer
- Central catalog, governed views
- Caching, row/column-level security
- Often used by BI teams

[IMAGE: Three-panel diagram comparing (1) distributed query engine with coordinator/workers, (2) single DB with foreign data wrappers, (3) semantic layer with governed views plus caching.]

### Decision game
Which pattern is most appropriate?

1) You need to run large analytical joins across S3 plus multiple DBs.
2) You mostly need to join two Postgres databases with modest data.
3) You need governed metrics definitions for BI with caching.

Pause and think.

<details>
<summary>Answer</summary>
1->Pattern 1, 2->Pattern 2, 3->Pattern 3
</details>

### Key insight
> Federation is not one thing; it is a family of architectures with different failure and performance profiles.

---

## Deep dive: Security and governance - the "unified door" problem

### Scenario
You expose a federated SQL endpoint to analysts.

Now one endpoint can reach:

- PCI payments DB
- PII customer DB
- Internal operational DB

### Pause and think
What is the biggest risk?

A) Analysts run expensive queries
B) Credential sprawl
C) Data exfiltration via joins that were previously impossible
D) All of the above

<details>
<summary>Answer</summary>
**D**.
</details>

### Common Misconception
> "If each source has permissions, federation inherits them safely."

In practice:

- Federation may use **service credentials** to access sources.
- Row-level security may not push down.
- Joining datasets can reveal sensitive attributes (inference attacks).

### Production insight: policy evaluation location matters
If masking/RLS is enforced only at the federation layer, you must treat the federation cluster as **in-scope** for the most sensitive data it can access (PCI/PII). That affects:
- network segmentation
- secrets management
- audit retention
- incident response

### Practical controls
- Centralized authN/authZ (OIDC, LDAP) mapped to source credentials
- Row/column-level security in the federation layer
- Query auditing and lineage
- Resource groups, quotas, admission control
- **Egress controls** (prevent `UNLOAD`/`EXPORT` to arbitrary buckets)

[IMAGE: Security flow diagram showing user to federation auth to policy engine to source connectors, with audit logs and masking.]

### Key insight
> Federation is a **data access platform**, not just a query convenience. Treat it like production infrastructure with governance.

---

## Mental model: Operational realities - caching, spill, and "the coordinator is your new database"

### Scenario
Your federation coordinator starts OOMing during peak hours.

### Pause and think
Which is most likely?

A) Too many concurrent queries causing large intermediate results during joins/sorts
B) Remote DBs are down
C) The coordinator disk is too fast

<details>
<summary>Answer</summary>
**A**.
</details>

### Explanation with analogy (smoothie blender)
The coordinator is like a smoothie blender:

- If you keep pouring ingredients (rows) faster than it can blend (join/sort), it overflows.

### Production insight: coordinator vs workers
In distributed engines (Trino/Presto), the coordinator can still become a choke point due to:
- planning overhead
- result buffering
- exchange management
- metadata/catalog calls

Mitigate with:
- separate coordinator sizing
- limiting result set sizes
- avoiding huge `ORDER BY` without `LIMIT`

### Operational checklist
- **Memory limits** per query and per node
- **Spill-to-disk** configuration and fast local disks
- **Concurrency controls** (resource groups)
- **Query timeouts** and kill switches
- **Result size limits** for interactive endpoints
- **Caching** for repeated dimension lookups

### Key insight
> In many deployments, the federation layer becomes the **new choke point**. You must capacity-plan it like a database cluster.

### Challenge question
If you enable aggressive caching, what failure mode might get worse?

<details>
<summary>Progressive reveal</summary>
Staleness or inconsistency: cached results can hide updates, creating correctness surprises unless you define freshness SLAs.
</details>

---

## Puzzle: Worked example - building a federated query plan (progressive reveal)

### Scenario
You need "late orders with low inventory":

- `orders` in Postgres
- `inventory` in Oracle

You write:

```sql
SELECT o.order_id, o.sku, o.promised_date, i.available
FROM orders o
JOIN inventory i ON i.sku = o.sku
WHERE o.status = 'LATE'
  AND i.available < 5;
```

### Pause and think
What is the best plan if only 0.1% of orders are late?

<details>
<summary>Think first...</summary>
Try to minimize cross-system data movement.
</details>

<details>
<summary>Answer</summary>
1) Push down `o.status='LATE'` to Postgres; retrieve only late orders (order_id, sku, promised_date).
2) Extract distinct SKUs from late orders.
3) Query Oracle inventory for those SKUs with `available < 5`.
4) Join results at coordinator.

This is a semi-join pattern with key shipping.
</details>

### Production insight: key shipping mechanics
Key shipping often becomes either:
- a large `IN (...)` list (hits SQL length/parameter limits), or
- a temp table / staging table (requires write permissions and cleanup), or
- a connector-specific mechanism (e.g., Trino dynamic filtering).

Prefer **batched key shipping** and enforce a **max key count** to avoid turning a selective query into a denial-of-service.

[IMAGE: Query plan diagram showing filter pushdown to orders source, key extraction, second query to inventory source using IN-list or temp table, final join at coordinator.]

### Key insight
> The best federated plans are usually **two-phase**: filter early, ship keys, then fetch matching rows.

---

## [CODE: SQL, demonstrating practical federation patterns]

### Example 1: PostgreSQL foreign data wrapper (FDW) to another Postgres

[CODE: SQL, show setting up postgres_fdw, creating server/user mapping, importing schema, and running a join]

```sql
-- On the "federating" Postgres instance
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER crm_pg
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'crm-db.internal', dbname 'crm', port '5432');

CREATE USER MAPPING FOR app_user
  SERVER crm_pg
  OPTIONS (user 'crm_reader', password '***');

IMPORT FOREIGN SCHEMA public
  LIMIT TO (customers)
  FROM SERVER crm_pg
  INTO federated;

-- Local table: orders
-- Foreign table: federated.customers
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_id, c.country, count(*)
FROM federated.customers c
JOIN public.orders o ON o.customer_id = c.customer_id
WHERE c.country = 'DE'
  AND o.created_at >= now() - interval '7 days'
GROUP BY 1,2;
```

**Pause and think**: Where do you expect most time to go?

<details>
<summary>Answer</summary>
Often in pulling `customers` rows over FDW and joining locally, unless the planner can push down filters and join conditions. FDW pushdown varies by wrapper and query shape.
</details>

#### Production insight: FDW pushdown and knobs
- `postgres_fdw` can push down many filters/joins, but not all; check `EXPLAIN VERBOSE` for `Remote SQL`.
- Consider `use_remote_estimate` (trade-off: better plans vs extra remote planning latency).
- Use `fetch_size` to tune network round trips.
- Ensure `ANALYZE` runs on foreign tables (or use remote estimates) to avoid catastrophic join order.

---

## [CODE] Implementing a federation coordinator (toy examples)

> These examples are intentionally simplified. Real coordinators need streaming, spill, cancellation, and workload isolation.

### Python: concurrent subqueries + local hash-join (fixed ordering + safer timeouts)

```python
import os
from concurrent.futures import ThreadPoolExecutor

import psycopg


def fetch_rows(dsn: str, sql: str, params: tuple = (), timeout_s: int = 5):
    """Fetch all rows with connect + statement timeouts.

    Notes:
    - connect_timeout is in seconds.
    - statement_timeout is in milliseconds.
    - In production, prefer server-side cursors / streaming to avoid OOM.
    """
    with psycopg.connect(dsn, connect_timeout=timeout_s) as conn:
        with conn.cursor() as cur:
            cur.execute("SET LOCAL statement_timeout = %s", (timeout_s * 1000,))
            cur.execute(sql, params)
            return cur.fetchall()


def federated_join(country: str = "DE", timeout_s: int = 5):
    """Query shipping + coordinator-side join.

    Failure semantics:
    - If either source times out/errors, the whole query fails.
    - This is typical for SQL semantics (no partial results).
    """
    crm_dsn = os.environ["CRM_PG_DSN"]
    orders_dsn = os.environ["ORDERS_PG_DSN"]

    q_customers = "SELECT customer_id FROM customers WHERE country=%s"
    q_orders = "SELECT customer_id, order_id FROM orders WHERE created_at >= now()-interval '7 days'"

    with ThreadPoolExecutor(max_workers=2) as ex:
        fut_customers = ex.submit(fetch_rows, crm_dsn, q_customers, (country,), timeout_s)
        fut_orders = ex.submit(fetch_rows, orders_dsn, q_orders, (), timeout_s)

        customers = fut_customers.result()
        orders = fut_orders.result()

    cust_ids = {cid for (cid,) in customers}
    return [(cid, oid) for (cid, oid) in orders if cid in cust_ids]


# Usage example:
# print(federated_join("DE")[:10])
```

**What was fixed vs the original?**
- Removed `as_completed` ordering bug (results could be swapped).
- Avoided `f"SET LOCAL ..."` string interpolation.
- Documented semantics and memory caveats.

### Node.js: parallel queries + join at coordinator (timeout correctness)

```javascript
import pg from 'pg';

async function queryWithTimeout(pool, text, values = [], timeoutMs = 5000) {
  const client = await pool.connect();
  try {
    // statement_timeout expects milliseconds
    await client.query('SET LOCAL statement_timeout = $1', [timeoutMs]);
    const res = await client.query({ text, values });
    return res.rows;
  } finally {
    client.release();
  }
}

export async function federatedJoin(country = 'DE', timeoutMs = 5000) {
  const crm = new pg.Pool({ connectionString: process.env.CRM_PG_DSN, max: 4 });
  const orders = new pg.Pool({ connectionString: process.env.ORDERS_PG_DSN, max: 4 });

  try {
    const [customers, recentOrders] = await Promise.all([
      queryWithTimeout(crm, 'SELECT customer_id FROM customers WHERE country=$1', [country], timeoutMs),
      queryWithTimeout(orders, "SELECT customer_id, order_id FROM orders WHERE created_at >= now()-interval '7 days'", [], timeoutMs)
    ]);

    const ids = new Set(customers.map(r => r.customer_id));
    return recentOrders.filter(r => ids.has(r.customer_id));
  } finally {
    await Promise.allSettled([crm.end(), orders.end()]);
  }
}

// Usage example:
// federatedJoin('DE').then(r => console.log(r.slice(0, 10))).catch(console.error);
```

### Production insight: why these toy coordinators are dangerous
- `fetchall()` pulls entire result sets into memory.
- No cancellation propagation (client cancels, sources keep running).
- No admission control (a burst of requests can overload sources).
- No snapshot semantics across sources.

---

### Example 2: Trino connector query with selective joins

[CODE: SQL, Trino query showing pushing down filters and using WITH to reduce cardinality]

```sql
-- Trino catalogs: mysql.crm, postgres.orders
WITH de_customers AS (
  SELECT customer_id
  FROM mysql.crm.customers
  WHERE country = 'DE'
), late_orders AS (
  SELECT order_id, customer_id, sku
  FROM postgres.orders.orders
  WHERE status = 'LATE'
    AND created_at >= current_timestamp - INTERVAL '7' DAY
)
SELECT o.order_id, o.sku
FROM late_orders o
JOIN de_customers c ON c.customer_id = o.customer_id;
```

**Think about it**: Why use CTEs here?

<details>
<summary>Progressive reveal</summary>
CTEs can help readability and sometimes encourage early filtering (though optimizers may inline them). The intent is to reduce row counts before the cross-catalog join.
</details>

#### Production insight: Trino dynamic filtering
Trino can apply **dynamic filtering**: it builds a filter from one side of a join and pushes it into the scan of the other side (connector permitting). This is essentially an automated semi-join.

To validate:
- Use `EXPLAIN ANALYZE` and look for dynamic filter stats.
- Ensure connectors support predicate pushdown and that join keys are compatible.

---

### Python: semi-join (key shipping) with batching + limits

```python
import os

import psycopg


def semi_join_late_orders(country: str = "DE", max_keys: int = 5000, timeout_s: int = 5):
    # Phase 1: fetch selective keys from CRM (pushdown filter at source).
    with psycopg.connect(os.environ["CRM_PG_DSN"], connect_timeout=timeout_s) as crm:
        with crm.cursor() as cur:
            cur.execute("SET LOCAL statement_timeout = %s", (timeout_s * 1000,))
            cur.execute(
                "SELECT customer_id FROM customers WHERE country=%s LIMIT %s",
                (country, max_keys),
            )
            customer_ids = [r[0] for r in cur.fetchall()]

    if not customer_ids:
        return []

    # Phase 2: ship keys to Orders DB and fetch only matching rows.
    sql = """
      SELECT order_id, customer_id, sku
      FROM orders
      WHERE status='LATE'
        AND created_at >= now()-interval '7 days'
        AND customer_id = ANY(%s)
    """

    with psycopg.connect(os.environ["ORDERS_PG_DSN"], connect_timeout=timeout_s) as orders:
        with orders.cursor() as cur:
            cur.execute("SET LOCAL statement_timeout = %s", (timeout_s * 1000,))
            cur.execute(sql, (customer_ids,))
            return cur.fetchall()


# Usage example:
# print(semi_join_late_orders("DE")[:10])
```

### Node.js: key-first query design with batching (parameter limits)

```javascript
import pg from 'pg';

export async function semiJoinLateOrders(country = 'DE', batchSize = 1000, timeoutMs = 5000) {
  const crm = new pg.Pool({ connectionString: process.env.CRM_PG_DSN, max: 4 });
  const orders = new pg.Pool({ connectionString: process.env.ORDERS_PG_DSN, max: 4 });

  try {
    const client = await crm.connect();
    let ids;
    try {
      await client.query('SET LOCAL statement_timeout = $1', [timeoutMs]);
      const res = await client.query('SELECT customer_id FROM customers WHERE country=$1', [country]);
      ids = res.rows.map(r => r.customer_id);
    } finally {
      client.release();
    }

    const out = [];
    for (let i = 0; i < ids.length; i += batchSize) {
      const batch = ids.slice(i, i + batchSize);
      const res = await queryWithTimeout(
        orders,
        "SELECT order_id, customer_id, sku FROM orders WHERE status='LATE' AND created_at >= now()-interval '7 days' AND customer_id = ANY($1)",
        [batch],
        timeoutMs
      );
      out.push(...res);
    }

    return out;
  } finally {
    await Promise.allSettled([crm.end(), orders.end()]);
  }
}

async function queryWithTimeout(pool, text, values = [], timeoutMs = 5000) {
  const client = await pool.connect();
  try {
    await client.query('SET LOCAL statement_timeout = $1', [timeoutMs]);
    const res = await client.query({ text, values });
    return res.rows;
  } finally {
    client.release();
  }
}
```

---

## Warning: Write federation and distributed transactions - "can we just update both?"

### Scenario
A product manager asks:

> "Can we update customer status in MySQL and create an order in Postgres in one transaction?"

### Pause and think
What is the honest answer?

A) Yes, SQL transactions are universal
B) Yes, just use 2PC everywhere
C) Usually no (or not safely), unless you accept complexity and limited availability

<details>
<summary>Answer</summary>
**C**.
</details>

### Explanation with analogy (two terminals)
You are trying to pay with **two credit cards** at two different terminals and want a single receipt that is either fully paid or fully canceled.

You *can* do it with coordination (2PC), but:

- If one terminal freezes, you are stuck.
- If the coordinator crashes mid-commit, you may need recovery protocols.

### Distributed systems rigor: 2PC availability impact
2PC is **blocking** under coordinator failure and can reduce availability during partitions. In CAP terms, you are choosing stronger consistency/atomicity at the cost of availability.

### Trade-offs
- **2PC**: stronger atomicity but can reduce availability and increase operational risk.
- **Sagas / outbox**: eventual consistency with compensation.
- **Materialized views**: write to one system, replicate to others.

### Common Misconception
> "Federation implies distributed transactions."

Most federation solutions focus on **read federation**. Write federation is a different beast.

### Key insight
> Federated writes often turn into **distributed transaction design**, which is rarely worth it unless the business truly requires it.

---

##  Deep dive: Observability - how to debug "this query is slow" across five systems

### Scenario
An analyst says: "The dashboard is slow."

But the query spans:

- two DBs
- one object store
- a cache

### Pause and think
What do you need first?

A) A faster coordinator machine
B) End-to-end query tracing with per-source timings
C) More indexes everywhere

<details>
<summary>Answer</summary>
**B**.
</details>

### What good looks like
- Query ID that propagates to connectors
- Per-stage timing (scan, exchange, join, aggregation)
- Per-source metrics (latency, bytes read, rows read)
- Top-N expensive queries
- Audit logs for access and lineage

[IMAGE: Screenshot-style mock showing a query timeline: Stage 1 scan MySQL 200ms, Stage 2 scan Postgres 4s, Stage 3 join 1s, Stage 4 sort spill 2s.]

### Production insight: cancellation and timeouts
Make sure:
- client cancellation propagates to the federation engine
- the engine cancels remote subqueries (or at least closes connections)
- timeouts are enforced **per stage** and **end-to-end**

Otherwise, you get "zombie queries" consuming resources long after users gave up.

### Key insight
> Without distributed tracing, federation is a black box that everyone blames.

### Challenge question
If Postgres is fast but the federation query is slow, what is a likely culprit?

<details>
<summary>Progressive reveal</summary>
Data movement and coordinator work (join/sort/aggregation) or non-pushed filters causing large transfers.
</details>

---

## Puzzle: Design patterns that make federation survivable

### Pattern A: "Dimensional caching"
Cache small, slow-changing dimensions (countries, product catalog) near the federation engine.

**Risk**: staleness.

### Pattern B: "Key-first query design"
Structure queries to:

1) select keys with high selectivity
2) fetch facts by key

### Pattern C: "Materialized views for hot joins"
If the same cross-source join is used constantly, consider materializing it into a dedicated store.

### Pattern D: "Federation as an internal product"
Provide:

- documented catalogs
- stable views
- SLAs
- on-call ownership

### Matching exercise
Match the pattern to the primary benefit.

1) Dimensional caching
2) Key-first query design
3) Materialized views
4) Federation as product

Benefits:

A) Reduced repeated remote lookups
B) Reduced data movement by selectivity
C) Stable performance for repeated workloads
D) Predictable governance and operational maturity

Pause and think.

<details>
<summary>Answer</summary>
1->A, 2->B, 3->C, 4->D
</details>

### Key insight
> Federation works best when you **shape workloads** to its strengths instead of treating it like a magic cross-DB join button.

---

##  Warning: Anti-patterns - how federation projects fail

### Anti-pattern 1: Treating federation as a replacement for domain boundaries
If you join everything with everything, you have built a distributed monolith.

### Anti-pattern 2: No workload isolation
One analyst query can starve production dashboards.

### Anti-pattern 3: Unlimited fan-out
A single query that fans out to 200 shards times 10 sources becomes a thundering herd.

### Anti-pattern 4: No schema contracts
Schema drift breaks dashboards daily.

### Pause and think
Which anti-pattern is most dangerous in a multi-tenant environment?

<details>
<summary>Answer</summary>
No workload isolation. Multi-tenant federation without quotas or admission control invites noisy-neighbor incidents.
</details>

### Production insight: workload isolation primitives
Look for:
- resource groups / queues
- per-user / per-team quotas
- per-source concurrency limits
- query pattern guardrails (require partition predicates, deny cross joins)

### Key insight
> Federation failure is often **organizational** (ownership, SLAs, governance) as much as technical.

---

## Challenge: Real-world usage - where federation shines

### Use case 1: Transitional integration
During migrations, federation can provide a unified view while data moves.

### Use case 2: Analytics across heterogeneous sources
Query S3 logs plus relational metadata.

### Use case 3: Data mesh / decentralized ownership
Teams own data products; federation provides discoverability and query access.

### Use case 4: Incident response and forensics
Ad-hoc cross-system queries during outages.

### Challenge question
Why is federation often popular with incident responders?

<details>
<summary>Progressive reveal</summary>
It reduces time-to-answer by allowing one query across logs/metrics/metadata sources - at the cost of performance predictability.
</details>

---

## Deep dive: Trade-offs summary - pick your pain deliberately

### Comparison table: federation trade-offs

| Dimension | Federation advantage | Federation cost |
|---|---|---|
| Time-to-integrate | Fast: no big migrations | Hidden complexity in connectors and planning |
| Freshness | Near real-time reads | Inconsistent snapshots across sources |
| Performance | Good for selective queries | Poor for large cross-source joins |
| Reliability | Single interface | Single blast radius; partial failures |
| Governance | Central control point | Central risk point; requires strong policies |

### Decision game
Which statement is most accurate?

A) Federation reduces the need for data modeling.
B) Federation increases the importance of data modeling.

Pause and think.

<details>
<summary>Answer</summary>
**B**. You need careful modeling (views, contracts, ownership) to avoid chaos and performance traps.
</details>

---

## Final synthesis challenge: Design a federation plan you would bet your on-call on

### Scenario
You are asked to deliver a "Customer Risk Dashboard" that shows:

- Customer PII (MySQL)
- Payment status (PCI Postgres)
- Recent support tickets (SaaS API ingested into a small Postgres)
- Fraud signals (Kafka topic or log store)

Constraints:

- Dashboard must load in **< 3 seconds p95** for 200 concurrent users.
- PCI DB must not receive unbounded scans.
- Support tickets schema changes monthly.
- Fraud signals are high volume.

### Your mission (pause and think)
Propose:

1) What you federate directly
2) What you materialize
3) What you cache
4) How you isolate workloads
5) What failure mode you accept

Stop and sketch a plan.

<details>
<summary>One possible strong answer</summary>
- **Federate directly**: small/medium relational sources with strong indexing and stable schemas (support tickets DB, customer MySQL) via connectors with strict predicate pushdown.
- **Materialize**: fraud signals into a curated, query-optimized store (for example, daily or near-real-time aggregates) to avoid scanning Kafka/logs at dashboard time.
- **Cache**: customer dimension attributes that change rarely (for example, risk tier, country), with explicit freshness (for example, 5 to 15 minutes) and cache invalidation on critical updates.
- **Workload isolation**: resource groups for dashboards vs ad-hoc; per-source concurrency limits; query timeouts; denylist expensive patterns (cross joins, missing predicates).
- **Failure mode accepted**: if PCI payment source is slow/unavailable, show "payment status unavailable" with a clear indicator (partial mode) *only if business approves semantics*; otherwise fail fast with a friendly error.

Key: meet latency by reducing cross-source join size and avoiding high-volume sources at query time.
</details>

### Key insight
> A production-grade federation design is mostly about **controlling data movement, controlling concurrency, and controlling semantics under failure**.

---

## Closing challenge questions

1) What are the top 3 queries you expect users to run, and how will you ensure pushdown for each?
2) Which sources are fragile and how will you protect them from federation fan-out?
3) What correctness guarantees do you publish (freshness, snapshot semantics, partial results)?
4) What is your escape hatch when federation becomes a bottleneck (materialization roadmap)?

---

### Appendix: Quick glossary

- **Federation**: unified query interface over multiple data sources.
- **Pushdown**: executing filters/aggregations at the source to reduce data movement.
- **Coordinator**: node that plans and orchestrates distributed query execution.
- **Semi-join**: two-phase join that ships keys to reduce data transfer.
- **2PC**: two-phase commit for distributed transactions.
