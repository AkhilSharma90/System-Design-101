---
slug: saga-patterns
title: Saga Patterns
readTime: 20 min
orderIndex: 1
premium: false
---




# Saga Pattern in Distributed Systems: Choreography vs Orchestration (Production Review)

[CHALLENGE] **Challenge: Your checkout succeeded... but your inventory didn't**

You run an e-commerce platform. A customer clicks **"Place Order"**.

- The **Order Service** creates an order (OK)
- The **Payment Service** charges the card (OK)
- The **Inventory Service** tries to reserve stock (FAIL: out of stock)
- The **Shipping Service** never even hears about it

Now you've charged a customer for something you can't ship.

If this were a monolith with a single database, you'd wrap everything in a transaction and roll back. In a distributed system, you don't get that luxury.

So you need a way to coordinate *multiple independent services* and still achieve a business-level "all-or-nothing" outcome.

Welcome to the **Saga pattern**.

---

## [HANDSHAKE] Section 1 - What problem does a Saga solve?

### Scenario
Your system is decomposed into services. Each service owns its data. Cross-service operations are **distributed transactions** in disguise.

Traditional **2PC (Two-Phase Commit)** tries to make distributed operations behave like a single ACID transaction. But 2PC has sharp edges:

- Tight coupling: participants must implement a coordinator protocol
- Blocking: coordinator failure can stall participants (classic availability hit)
- Operational complexity: timeouts, heuristics, recovery
- Scalability bottleneck: coordinator and lock management

A **Saga** replaces a single distributed transaction with:

1. A sequence of **local transactions** (each service commits its own DB changes)
2. A sequence of **compensating transactions** to undo work when something fails

### Network assumptions (state them explicitly)
In production, assume:

- The network can drop, delay, duplicate, and reorder messages.
- Timeouts do **not** imply failure (unknown outcome).
- Brokers are typically **at-least-once** delivery.
- Partitions happen; you must choose what to sacrifice (CAP trade-offs).

### Interactive question
Pause and think: If each local transaction commits independently, what prevents the system from ending up in a "half-done" state?

(Write down your guess: retries? timeouts? compensation? idempotency? reconciliation?)

### Explanation with analogy
Think of a restaurant group with separate counters:

- Counter A takes your order and prints a receipt.
- Counter B charges your card.
- Counter C allocates ingredients.
- Counter D schedules delivery.

There's no single manager who can "uncharge your card" automatically if ingredients aren't available - unless you design a **refund process** (compensation).

A Saga is that refund-and-reversal playbook.

### Real-world parallel
Airline booking:

- Reserve seat
- Charge payment
- Issue ticket

If ticket issuing fails, you might cancel the seat and refund payment. That's a Saga.

### Key insight
> **Key insight:** A Saga does not give you atomicity across services. It gives you a *business-consistent outcome over time* using **coordination + compensations + reconciliation**.

### Production clarifications
- **Eventual consistency is not eventual correctness.** Eventual consistency only says replicas converge *if updates stop and communication resumes*. Correctness requires you to define invariants and enforce them with idempotency, fencing, and compensations.
- **Compensation is not rollback.** Rollback is a database primitive with isolation guarantees. Compensation is a *new business operation* that can fail, be partial, or be legally constrained.

### Challenge questions
1. Why is "eventual consistency" not the same thing as "eventual correctness"?
2. What makes a compensation harder than a rollback?

---

## [ALERT] Section 2 - Mental model: Sagas are state machines, not transactions

### Scenario
You want to reason about correctness. With ACID transactions, you reason about isolation and atomicity. With Sagas, you reason about **states** and **transitions**.

### Interactive question
Pause and think: In a Saga, what are the two "directions" the state machine can move?

### Explanation with analogy
Imagine a delivery route:

- You drive forward through stops: pickup -> sort -> load -> deliver.
- If the truck breaks down at "load", you might drive backward through a return route: unload -> return -> refund.

Sagas move:

- **Forward** via local transactions
- **Backward** via compensating transactions

### Key insight
> **Key insight:** Model Sagas explicitly as a **durable workflow/state machine** with replay-safe transitions - not as a "transaction with retries".

[IMAGE: **State machine diagram**. Forward steps `T1 -> T2 -> T3`. Compensation steps `C2 <- C1`. Terminal states: `SUCCEEDED`, `FAILED`, `COMPENSATED`, and optionally `PARTIAL_ACCEPTED` (domain-specific).]

### Production insight: terminal states
A Saga should have explicit terminal states and policies:

- `SUCCEEDED`: all required steps completed
- `COMPENSATED`: forward steps undone to an acceptable business state
- `FAILED_NEEDS_MANUAL`: cannot compensate automatically
- `TIMED_OUT`: exceeded SLA; may still complete later unless fenced

### Challenge questions
1. What is a "terminal state" for a Saga?
2. Can a Saga have partial success as an acceptable terminal state?

---

## [CHALLENGE] Section 3 - Two flavors: Choreography vs Orchestration

### Scenario
You need to implement a Saga. You must decide: do services coordinate **implicitly** via events, or **explicitly** via a controller?

Two common styles:

1. **Choreography**: services react to events and publish events; no central controller.
2. **Orchestration**: a Saga orchestrator tells services what to do next; services reply.

### Interactive question
Pause and think: Which one sounds more scalable? Which one sounds easier to debug at 3am?

### Key insight
> **Key insight:** Choreography optimizes for **local autonomy**; orchestration optimizes for **global visibility and control**.

### CAP/consistency framing (practical)
- Under partitions, you typically choose between:
  - **Availability** (accept requests, reconcile later) -> more compensation/reconciliation
  - **Consistency** for specific invariants (reject/queue requests) -> lower availability
- Sagas are often used to keep the system **available** while maintaining **business invariants** via compensations and fences.

---

## [MAGNIFY] Section 4 - Choreography Saga: how it works (and how it fails)

### Example workflow (checkout)
1. Order Service emits `OrderCreated`
2. Payment Service consumes `OrderCreated`, authorizes/charges, emits `PaymentAuthorized` or `PaymentFailed`
3. Inventory Service consumes `PaymentAuthorized`, reserves, emits `InventoryReserved` or `InventoryFailed`
4. Shipping Service consumes `InventoryReserved`, creates shipment, emits `ShipmentCreated` or `ShipmentFailed`
5. Order Service consumes final events, marks order `COMPLETED` or `CANCELLED`

[IMAGE: **Event flow diagram** with services as boxes and events as arrows. Include:
- success path: `OrderCreated -> PaymentAuthorized -> InventoryReserved -> ShipmentCreated -> OrderCompleted`
- failure path: `InventoryFailed -> PaymentVoided/Refunded -> OrderCancelled`]

### Interactive question
Pause and think: Where does the system decide to compensate? Is there a single place that "knows" the whole workflow?

### Key insight
> **Key insight:** Choreography spreads workflow logic across services. The system's behavior emerges from event interactions.

### Failure scenarios (distributed-systems reality)

#### 1) Duplicate events (at-least-once)
`PaymentAuthorized` may arrive twice.

- Payment must be **idempotent** (don't double-charge)
- Inventory must be idempotent (don't reserve twice)

**Production pattern:** store a processed-message key (e.g., `eventId`) in an **inbox/dedup table** with TTL aligned to replay window.

#### 2) Out-of-order events
The real issue is:

- **Out-of-order delivery across different event types** (e.g., `PaymentFailed` arrives after `PaymentAuthorized` due to retries)
- **Replays** after consumer restarts

**Mitigation:** consumers must validate current aggregate state (or saga step version) before applying.

#### 3) Lost "compensate" signal
Inventory emits `InventoryFailed`, but Order Service never receives it.

- Use durable subscriptions, DLQs, replay, and periodic reconciliation.
- Prefer emitting facts from an outbox so they are not lost.

#### 4) Semantic coupling
A new service subscribes to `PaymentAuthorized` and changes behavior. Suddenly the Saga has new hidden dependencies.

- Events become an implicit API.

#### 5) Split-brain / partitions
During a partition, some services may continue processing while others lag.

- You can end up with **temporarily inconsistent** states.
- You need **fencing** to prevent late-arriving forward steps from "reviving" a compensated saga.

### Challenge questions
1. What's the difference between *transport-level* deduplication and *business-level* idempotency?
2. In choreography, who owns the "definition" of the Saga?

---

## [ALERT] Section 5 - Common misconception: "Choreography means no coupling"

### Scenario
Teams often say: "We're event-driven, so we're loosely coupled." Then they add 12 subscribers to a single event.

### Key insight
> **Key insight:** Choreography reduces *control-plane coupling* but can increase *semantic coupling* unless events are versioned and governed.

### Production practices to reduce semantic coupling
- Treat events as **public contracts**: versioning, compatibility rules, schema registry
- Prefer **facts** over commands on the bus
- Use **bounded contexts**: don't publish internal domain details broadly
- Use **topic partitioning by domain** and access controls
- Consumer-driven contract tests

---

## [GAME] Section 6 - Decision game: Which statement is true?

Pick the true statement(s). Pause first.

A. In choreography, there is no need for a state machine.

B. In orchestration, services do not need idempotency.

C. In choreography, adding a new subscriber can change system behavior without changing producers.

D. In orchestration, the orchestrator is a single point of failure.

**Reveal:**

- A is **false**: the state machine exists, but it's distributed and implicit.
- B is **false**: idempotency is still required due to retries/timeouts.
- C is **true**: subscribers can introduce emergent behavior.
- D is **partly true**: it can be a SPOF if not replicated and durable; well-designed orchestrators persist state and run HA.

> **Key insight:** Both styles require idempotency, durable state, and careful failure handling. The difference is *where the workflow logic lives*.

---

## [MAGNIFY] Section 7 - Orchestration Saga: how it works (and how it fails)

### Scenario
Implement the same checkout with an orchestrator.

The orchestrator drives the flow:

1. `CreateOrder`
2. `AuthorizePayment`
3. `ReserveInventory`
4. `CreateShipment`

On failure, it triggers compensations:

- If inventory reservation fails: `VoidOrRefundPayment` + `CancelOrder`

[IMAGE: **Sequence diagram** showing orchestrator sending commands to services and receiving replies/events. Include compensation path and retries.]

### Interactive question
Pause and think: What new failure mode did we introduce by adding an orchestrator?

### Key insight
> **Key insight:** Orchestration centralizes workflow decisions, improving observability and control, but it adds a critical component that must be engineered for HA and correctness.

### Failure scenarios

#### 1) Orchestrator crashes mid-Saga
- If state is persisted durably, it resumes.
- If state is only in memory, you get partial executions and ambiguity.

#### 2) Command delivered, response lost (unknown outcome)
Orchestrator sends `ReserveInventory`. Inventory reserves, but response is lost.

- Orchestrator retries
- Inventory must be idempotent using a stable **idempotency key** (often `sagaId + stepName`)

#### 3) Duplicate compensations
Due to retry, orchestrator might send `CancelPayment` twice.

- Compensations must be idempotent too.

#### 4) Orchestrator logic bug
Centralized logic means a single bug can affect all flows.

**Production mitigation:** feature flags, canarying, workflow versioning, and strong unit tests on the state machine.

---

## [ALERT] Section 8 - Common misconception: "Orchestration is just a monolith in disguise"

### Key insight
> **Key insight:** Orchestration centralizes *control flow*, not *data ownership*. The monolith risk comes from putting domain logic into the orchestrator instead of services.

### Production guidance: what belongs where?
- **Orchestrator:** sequencing, timeouts, retries, compensation decisions, audit trail
- **Services:** domain invariants, local transactions, idempotency, validation

---

## [HANDSHAKE] Section 9 - Comparison table: Choreography vs Orchestration

| Dimension | Choreography Saga | Orchestration Saga |
|---|---|---|
| Workflow logic location | Distributed across services | Centralized in orchestrator |
| Operational visibility | Harder (need tracing across events) | Easier (single workflow view) |
| Coupling | Lower control coupling, higher semantic coupling risk | Higher control coupling to orchestrator API |
| Change management | Adding steps can be tricky (many subscribers) | Easier to change flow in one place |
| Failure handling | Distributed compensation logic | Centralized compensation logic |
| Testing | Complex integration tests across services | Orchestrator can be tested as a state machine |
| Runtime dependencies | Broker/event bus is critical | Orchestrator + persistence is critical |
| Scaling | Naturally scales with services | Orchestrator must scale; often horizontally |
| Best when | Simple flows, high autonomy, event-native domains | Complex workflows, strict auditability, need clear ownership |

> **Key insight:** Neither is universally better. Choose based on **workflow complexity**, **observability needs**, and **organizational ownership**.

[IMAGE: **Decision matrix**: x-axis workflow complexity, y-axis need for auditability/visibility; show choreography favored bottom-left, orchestration top-right, hybrid in the middle.]

---

## [PUZZLE] Section 10 - Matching exercise: Map failure modes to mitigations

### Failure modes
1. Duplicate `AuthorizePayment` command
2. Lost `PaymentAuthorized` event
3. Orchestrator retries after timeout, but operation succeeded
4. Inventory reserved twice due to replay
5. Compensation executed, but forward step later completes (race)

### Mitigations
A. Idempotency keys + dedup store (inbox)

B. Outbox pattern + reliable event publishing

C. Saga step fencing tokens / version checks

D. Exactly-once delivery

E. Durable workflow state + at-least-once commands

**Reveal (one reasonable mapping):**

- 1 -> A
- 2 -> B
- 3 -> E + A
- 4 -> A
- 5 -> C

Why not D? Because "exactly-once delivery" is rarely achievable end-to-end; we typically build **effectively-once processing** from at-least-once + idempotency.

### Production clarification: fencing
A common fencing technique is a **monotonic saga step/version** stored with the aggregate. A late message with an older step/version is ignored.

[IMAGE: **Fencing diagram** showing `COMPENSATED` state with a higher version; late-arriving `ReserveInventorySucceeded` with lower version is rejected.]

---

## [MAGNIFY] Section 11 - The hard part: designing compensations

Compensations are not perfect inverses.

If you can't undo a step (email sent, external bank transfer settled), you can only:

- Initiate a return
- Offer refund/credit
- Create a customer support case

### Compensation patterns
1. **Semantic undo**: `ReserveInventory` -> `ReleaseInventory`
2. **Counteraction**: `ChargeCard` -> `RefundCard` (or `VoidAuthorization` if not captured)
3. **Corrective workflow**: support ticket / manual intervention
4. **Alternative forward path**: backorder instead of cancel

> **Key insight:** Compensations must be modeled as first-class business operations with their own failure modes, SLAs, and audit trails.

[IMAGE: **Compensation ladder**: void auth (best) -> refund (ok) -> credit note (ok) -> manual case (last resort).]

---

## [ALERT] Section 12 - Isolation levels in Sagas (why anomalies happen)

Two customers try to buy the last item.

Without global isolation, anomalies include:

- Oversell / write skew
- Stale reads leading to invalid decisions

### Common isolation strategies
- **Pessimistic reservation**: reserve stock early; may reduce availability
- **Optimistic allocation**: accept order then fail later; needs compensation
- **Escrow / bounded counters**: allocate tokens of inventory

[IMAGE: **Concurrency diagram** showing two sagas competing for one inventory token; only one gets it.]

> **Key insight:** Sagas shift isolation problems from the database to the application. You must choose domain-appropriate concurrency control.

---

## [CHALLENGE] Section 13 - Implementation building blocks (the unglamorous essentials)

A Saga design is only as reliable as its messaging and persistence patterns.

You typically need:

1. **Durable state** (order state, saga state)
2. **Reliable message delivery** (at-least-once)
3. **Idempotent handlers**
4. **Outbox / inbox patterns**
5. **Timeouts and retries** with backoff
6. **Dead-letter handling** + replay
7. **Observability** (correlation IDs, tracing)

### Outbox pattern (why it matters)
If a service updates its DB and publishes an event, you can get:

- DB commit succeeds
- Event publish fails

Now other services never learn about the change.

Outbox solves this by writing the event to an **outbox table** in the same local transaction, then publishing asynchronously.

#### SQL: transactional outbox (Postgres)
```sql
BEGIN;

-- 1) Local business write
INSERT INTO orders(order_id, status, total_cents)
VALUES ($1, 'CREATED', $2);

-- 2) Outbox write in the SAME transaction
INSERT INTO outbox(
  id,
  aggregate_id,
  event_type,
  payload_json,
  created_at,
  published_at
)
VALUES (
  gen_random_uuid(),
  $1,
  'OrderCreated.v1',
  $3::jsonb,
  now(),
  NULL
);

COMMIT;

-- Index for fast polling/locking:
-- CREATE INDEX outbox_unpublished_idx ON outbox(created_at)
--   WHERE published_at IS NULL;
```

#### Python: outbox publisher loop (correct locking + transaction boundaries)
The original code used `FOR UPDATE SKIP LOCKED` without an explicit transaction boundary and updated `published_at` before ensuring the broker accepted the message. Below is a safer pattern:

- Claim rows in a transaction
- Publish
- Mark published in the same transaction
- If publish fails, rollback so rows become visible again

```python
import json
import logging
import time
from contextlib import closing
import socket

log = logging.getLogger("outbox")

# db is assumed to provide: begin(), commit(), rollback(), fetch_all(), execute()

def publish_outbox_rows(db, broker_host="127.0.0.1", broker_port=9000, batch=50):
    while True:
        try:
            db.begin()
            rows = db.fetch_all(
                """
                SELECT id, event_type, payload_json
                FROM outbox
                WHERE published_at IS NULL
                ORDER BY created_at
                FOR UPDATE SKIP LOCKED
                LIMIT %s
                """,
                (batch,),
            )

            if not rows:
                db.commit()
                time.sleep(0.25)
                continue

            # Publish outside the DB engine but inside our logical try.
            # If publish fails, we rollback so rows are not marked published.
            with closing(socket.create_connection((broker_host, broker_port), timeout=2)) as s:
                for (event_id, event_type, payload_json) in rows:
                    msg = json.dumps(
                        {"id": str(event_id), "type": event_type, "payload": payload_json}
                    ) + "\n"
                    s.sendall(msg.encode("utf-8"))

            # Mark published after successful send.
            ids = tuple(r[0] for r in rows)
            db.execute(
                "UPDATE outbox SET published_at = now() WHERE id = ANY(%s)",
                (list(ids),),
            )
            db.commit()

        except (OSError, TimeoutError) as e:
            log.warning("publish failed; will retry: %s", e)
            db.rollback()
            time.sleep(1)
        except Exception:
            log.exception("unexpected publisher error")
            db.rollback()
            time.sleep(2)
```

**Important production note:** this still provides **at-least-once** publishing. Consumers must deduplicate by `event_id`.

#### Inbox/dedup (missing but critical)
For effectively-once processing, consumers should store processed `event_id` (or `(producer, sequence)`):

[IMAGE: **Inbox/outbox diagram** showing producer outbox -> broker -> consumer inbox/dedup table -> handler.]

---

## [MAGNIFY] Section 14 - Choreography in practice: event design and governance

Choreography works best with **facts**.

- Prefer past-tense facts: `PaymentAuthorized.v1`
- Include correlation identifiers: `orderId`, `sagaId`, `causationId`, `correlationId`
- Version events and enforce compatibility
- Avoid leaking internal fields that become hard to change

### When does an event become a command?
If the message is addressed to a specific service and implies obligation ("do X"), it's a command. Commands are usually better as direct RPC or a command queue with clear ownership.

[IMAGE: **Facts vs commands** diagram: broadcast facts on event bus; targeted commands via orchestrator/command channel.]

---

## [FAUCET] Section 15 - Orchestration in practice: workflow engines vs custom orchestrators

### Build vs buy
Building your own orchestrator means implementing:

- Durable state machine
- Timers, retries, backoff
- Replay semantics
- Versioning/migrations
- Visibility and audit

Workflow engines (Temporal/Cadence/Zeebe/Step Functions) provide these primitives.

### Corrected example: lightweight orchestrator (Node.js)
The original snippet used `fetch` without importing it (Node < 18) and persisted state to a local file (not HA). Below is a **didactic** example that:

- Uses Node 18+ (global `fetch`) or imports `node-fetch`
- Persists state to a durable store abstraction (replace with Postgres/DynamoDB)
- Uses stable idempotency keys per step

```javascript
// Node.js 18+ assumed for global fetch.
// For older Node: import fetch from "node-fetch";

async function withTimeout(promise, ms) {
  const timeout = new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), ms));
  return Promise.race([promise, timeout]);
}

async function callService(path, payload) {
  const res = await fetch(`http://localhost:8080/${path}`, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error(`${path} failed: ${res.status}`);
  return res.json();
}

// store must be durable in production (DB). This is an interface.
// store.get(sagaId) -> { step, ... }
// store.put(sagaId, state) -> void

export async function runCheckoutSaga(store, sagaId, orderId) {
  const state = (await store.get(sagaId)) ?? { step: "CREATE_ORDER" };

  // Stable idempotency keys per step (not just sagaId).
  const key = (step) => `${sagaId}:${step}`;

  try {
    if (state.step === "CREATE_ORDER") {
      await withTimeout(callService("createOrder", { orderId, sagaId, idempotencyKey: key("CREATE_ORDER") }), 2000);
      await store.put(sagaId, { step: "AUTH_PAYMENT" });
    }

    const s1 = (await store.get(sagaId)).step;
    if (s1 === "AUTH_PAYMENT") {
      await withTimeout(callService("authorizePayment", { orderId, sagaId, idempotencyKey: key("AUTH_PAYMENT") }), 2500);
      await store.put(sagaId, { step: "RESERVE_INV" });
    }

    const s2 = (await store.get(sagaId)).step;
    if (s2 === "RESERVE_INV") {
      await withTimeout(callService("reserveInventory", { orderId, sagaId, idempotencyKey: key("RESERVE_INV") }), 2500);
      await store.put(sagaId, { step: "CREATE_SHIPMENT" });
    }

    const s3 = (await store.get(sagaId)).step;
    if (s3 === "CREATE_SHIPMENT") {
      await withTimeout(callService("createShipment", { orderId, sagaId, idempotencyKey: key("CREATE_SHIPMENT") }), 4000);
      await store.put(sagaId, { step: "DONE" });
    }

    return { sagaId, orderId, status: "DONE" };
  } catch (e) {
    // Compensation is also at-least-once: cancel endpoints must be idempotent.
    await callService("voidOrRefundPayment", { orderId, sagaId, idempotencyKey: key("COMP_PAYMENT") }).catch(() => {});
    await callService("releaseInventory", { orderId, sagaId, idempotencyKey: key("COMP_INV") }).catch(() => {});
    await callService("cancelOrder", { orderId, sagaId, idempotencyKey: key("COMP_ORDER") }).catch(() => {});

    await store.put(sagaId, { step: "COMPENSATED", error: String(e) });
    throw e;
  }
}
```

**Production note:** a real orchestrator also needs timers for long-running steps, workflow versioning, and a way to query status.

[IMAGE: **Orchestrator durability diagram**: orchestrator instances (HA) + durable DB + command dispatch + replies/events.]

---

## [GAME] Section 16 - Quiz: pick the best design

### Workflow A: Simple email notification chain
Order placed -> send confirmation email -> send SMS

### Workflow B: Money movement across multiple ledgers and external bank
Debit wallet -> credit merchant -> initiate bank transfer -> confirm settlement

### Workflow C: Ride-hailing dispatch
Create ride request -> match driver -> driver accepts -> start trip -> end trip

**Reveal (one reasonable answer):**

- A: **Choreography** (simple, low risk, naturally event-driven)
- B: **Orchestration** (high auditability, complex failure handling, external dependencies)
- C: Often **Orchestration** (complex state machine, timeouts, reassignments), though some parts can be choreographed.

---

## [MAGNIFY] Section 17 - Observability: making Sagas debuggable

Minimum identifiers to propagate:

- `sagaId` (workflow instance)
- `orderId` (business key)
- `correlationId` (ties a request chain together)
- `causationId` (points to the message that caused this message)
- `stepName` + `attempt`

Also:

- Distributed tracing (W3C trace context)
- Structured logs with consistent fields
- Metrics: success rate, compensation rate, time-to-complete, stuck-saga count

[IMAGE: **Trace timeline** showing sagaId across services with retries and compensation steps.]

---

## [ALERT] Section 18 - Timeouts, retries, and the "unknown outcome" problem

Orchestrator calls Payment Service. It times out.

Did the payment fail? Or did it succeed but the response got lost?

**Only safe assumption after a timeout:** you do **not** know.

Mitigations:

- Use idempotency keys
- Make operations queryable (`GetPaymentStatus`)
- Prefer payment flows like **authorize then capture**

[IMAGE: **Unknown outcome diagram**: request sent -> timeout -> two possible realities (succeeded/failed) -> reconcile via status API.]

---

## [PUZZLE] Section 19 - Data consistency and read models during a Saga

While a Saga is in progress, users refresh the UI.

Expose in-progress states explicitly:

- `PENDING_PAYMENT`
- `PENDING_INVENTORY`
- `PENDING_SHIPMENT`
- `COMPLETING`

Avoid lying with a premature "confirmed".

Patterns:

- Process manager updates order status as events arrive
- Materialized views for user-facing queries
- Backpressure: disable actions until terminal state

[IMAGE: **UI state diagram** mapping saga steps to user-visible statuses.]

---

## [CHALLENGE] Section 20 - Hybrid approach: choreograph inside, orchestrate outside

Many real systems mix both styles.

Example:

- Orchestrator coordinates cross-domain steps: payment, inventory, shipping
- Within inventory, internal services use choreography events

> **Key insight:** Use orchestration for cross-domain workflows; use choreography for intra-domain reactions.

[IMAGE: **Hybrid architecture diagram**: orchestrator at domain boundary; internal domain event mesh inside each bounded context.]

---

## [ALERT] Section 21 - Security and compliance: auditability, PII, and least privilege

Choreography often broadcasts events broadly, increasing risk of:

- Over-sharing PII
- Unknown consumers storing sensitive fields

Mitigations:

- Data minimization in events (prefer IDs/tokens)
- Tokenization
- Access-controlled topics
- Auditable workflow histories

[IMAGE: **PII minimization diagram**: event contains `customerId` token; PII fetched only by authorized service.]

---

## [MAGNIFY] Section 22 - Real-world usage patterns

Common domains:

- E-commerce checkout and fulfillment
- Travel booking
- Fintech payments and ledger operations
- Telecom provisioning
- SaaS onboarding workflows

Common infrastructure choices:

- Choreography: Kafka/RabbitMQ/PubSub + outbox/inbox + stream processing
- Orchestration: Temporal/Cadence/Zeebe/Step Functions + service RPC + durable state

---

## [CHALLENGE] Section 23 - Final synthesis challenge: design a Saga under chaos

Design a checkout Saga for:

- Order Service (Postgres)
- Payment Service (external PSP + internal ledger)
- Inventory Service (Redis + Postgres)
- Shipping Service (third-party carrier API)
- Notification Service (email/SMS)

Constraints:

- Broker is at-least-once
- Network partitions happen
- Carrier API is flaky
- You must not double-charge
- You must not oversell

### Strong production design (often hybrid with orchestration)
- Durable orchestrator owns the high-level checkout workflow.
- Payment uses **authorize then capture** with idempotency keys.
- Inventory uses **reservation tokens/escrow** to avoid oversell.
- Shipping step retries with exponential backoff and a timeout; if it times out, orchestrator can:
  - keep retrying while order is `PENDING_SHIPMENT`, or
  - compensate (release inventory, void authorization)
- Notifications are choreographed off `OrderCompleted` / `OrderCancelled` facts.

[IMAGE: **Final architecture diagram** showing orchestrator, services, broker, outbox/inbox tables, and external dependencies; include correlation IDs flow.]

### Stuck saga handling (missing in many designs)
- Define an SLA per step and a global saga TTL.
- Run a sweeper that finds sagas in non-terminal states beyond TTL.
- Move to `FAILED_NEEDS_MANUAL` and page/queue for ops.

---

## [BYE] Epilogue - What you should remember

- Sagas coordinate distributed workflows using local transactions + compensations.
- Choreography distributes control flow across event subscribers.
- Orchestration centralizes control flow in a durable workflow engine/service.
- Both require: idempotency, durable state, reliable messaging, and observability.
- Compensations are business operations, not rollbacks.

If you can answer: "After a timeout, how do I know what happened?" and "How do I undo this step safely?" you're thinking like a distributed systems engineer.
