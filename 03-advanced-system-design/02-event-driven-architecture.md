# Event-Driven Architecture

Event-driven architecture (EDA) is a design paradigm where services communicate by producing and consuming events. Instead of services calling each other directly, they emit events describing what happened — and other services react. It's one of the most powerful patterns for building loosely coupled, scalable systems.

---

## Events vs Commands vs Queries

Understanding these three communication types is foundational:

```
COMMAND — "Do this"
  POST /payments/charge   → "Charge this card"
  Direct service call. Synchronous. Expects a response.
  One producer, one consumer.

QUERY — "Tell me this"
  GET /users/42           → "Get user 42's data"
  Read-only. Returns data. Synchronous.

EVENT — "This happened"
  "order.placed"          → "An order was placed"
  Past tense. Fire-and-forget. Asynchronous.
  One producer, many consumers.
```

Events are facts about what happened in the world. They're immutable — you can't un-place an order.

---

## Event-Driven Patterns

### 1. Event Notification

Services emit events to notify others that something happened. Consumers decide what to do.

```
Order Service                   Kafka Topic: "order.placed"
      │                                 │
      └── emit "order.placed" ─────────▶├──▶ Email Service (sends confirmation)
                                         ├──▶ Inventory Service (reserves stock)
                                         ├──▶ Analytics Service (logs for reporting)
                                         └──▶ Fraud Service (checks for fraud)

Order Service doesn't know or care who's listening.
Adding a new consumer requires zero changes to Order Service.
```

**Benefits:** Extreme decoupling. Adding consumers is free.
**Risks:** Hard to know all effects of an action. Debugging flows requires tracing.

### 2. Event-Carried State Transfer

Events carry enough data that consumers don't need to call back for details.

```
❌ Thin event (requires callback):
   { "event": "order.placed", "order_id": "12345" }
   → Consumer must call Order Service to get order details
   → Coupling reintroduced via the callback

✅ Fat event (self-contained):
   {
     "event": "order.placed",
     "order_id": "12345",
     "user_id": 42,
     "items": [{"product_id": 99, "qty": 2, "price": 29.99}],
     "total": 59.98,
     "shipping_address": {...},
     "placed_at": "2024-01-15T10:30:00Z"
   }
   → Consumer has everything it needs — no callback required
```

---

### 3. Event Sourcing

Instead of storing the current state of an entity, store **every event** that led to that state. The current state is derived by replaying events.

```
Traditional DB (stores current state):
  orders table: { id: 1, status: "shipped", total: $99 }

Event Sourcing (stores events):
  event store: [
    { event: "order.placed",  timestamp: t1, data: {...} },
    { event: "order.paid",    timestamp: t2, data: {...} },
    { event: "order.shipped", timestamp: t3, data: {...} }
  ]

Current state = replay all events:
  start: {}
  + order.placed  → { status: "placed", total: $99 }
  + order.paid    → { status: "paid", payment_id: "..." }
  + order.shipped → { status: "shipped", tracking: "..." }
```

### Event Sourcing Benefits

```
Audit Trail (free!):
  Every change is recorded. You know exactly what happened and when.

Time Travel:
  Replay events up to any point in time.
  "What did this user's cart look like at 2pm on Tuesday?"

Replay for New Features:
  New service needs historical data?
  Replay all past events to build its own view.

Debugging:
  Reproduce any bug by replaying the exact sequence of events.
```

### Event Sourcing Challenges

```
Eventual consistency:     Current state requires replay (can be slow)
Schema evolution:         Old events must stay parseable as schema changes
Snapshot pattern:         Snapshot state periodically to avoid full replay

Snapshot every 100 events:
  [e1..e100 → snapshot S1] [e101..e200 → snapshot S2]
  To get current state: load latest snapshot + replay events since it
```

---

### 4. CQRS — Command Query Responsibility Segregation

Separate the read model from the write model.

```
Traditional Architecture (single model for reads and writes):
  ┌──────────────┐     ┌─────────────────┐
  │   Service    │────▶│    Database     │
  │  reads and   │◀────│  (same tables)  │
  │  writes here │     └─────────────────┘

CQRS Architecture:

  Write side:                           Read side:
  ┌──────────────┐     ┌──────────┐    ┌──────────────┐    ┌──────────────┐
  │  Commands    │────▶│  Write   │    │   Queries    │───▶│   Read DB    │
  │  (mutations) │     │   DB     │    │   (reads)    │    │  (optimized  │
  └──────────────┘     └────┬─────┘    └──────────────┘    │  for reads)  │
                            │ events                        └──────────────┘
                            └──────────────────────────────▶ (updated via events)
```

**Write DB:** Normalized, optimized for transactional writes. Could be PostgreSQL.
**Read DB:** Denormalized, optimized for query patterns. Could be Redis, Elasticsearch, or a read replica with materialized views.

### Why CQRS?

```
E-commerce example:
  Write: ORDER_PLACED → Orders table (normalized)
  Read:  "Show me my order history with product names, images, status"
         → Requires JOIN across 5 tables (slow)

  With CQRS:
  Write: ORDER_PLACED → Orders table
                │ event
                ▼
  Read model updated: order_history_view table
    (denormalized, pre-joined, fast to query)
  Read: single query to order_history_view (fast!)
```

---

## The Saga Pattern

When an operation spans multiple services, you need a way to ensure all succeed — or roll back gracefully. This is the Saga pattern.

### The Problem

```
Place Order requires:
1. Reserve inventory      (Inventory Service)
2. Charge payment         (Payment Service)
3. Schedule fulfillment   (Fulfillment Service)

If step 3 fails after steps 1 and 2 succeed:
→ Payment was charged but order won't ship
→ Need to compensate: refund payment, release inventory
```

### Choreography-Based Saga

Services react to events, no central coordinator:

```
Order Service
  → emit: "order.inventory_requested"
      └──▶ Inventory Service reserves items
            → emit: "inventory.reserved"
                └──▶ Payment Service charges card
                      → emit: "payment.completed"
                          └──▶ Fulfillment Service schedules
                                → emit: "order.confirmed"

Failure path (payment fails):
  Payment Service
    → emit: "payment.failed"
        └──▶ Inventory Service releases reservation
              → emit: "inventory.released"
                  └──▶ Order Service marks order failed
```

**Pros:** No single point of failure, services are autonomous
**Cons:** Hard to reason about the overall flow, difficult to debug

### Orchestration-Based Saga

A central Saga Orchestrator coordinates:

```
┌───────────────────────────────────────┐
│         Saga Orchestrator             │
│                                       │
│  1. Reserve Inventory ──────────────▶ Inventory Service
│     OK ──────────────────────────────◀
│                                       │
│  2. Charge Payment ─────────────────▶ Payment Service
│     FAILED ──────────────────────────◀
│                                       │
│  3. Compensate: Release Inventory ──▶ Inventory Service
│     OK ──────────────────────────────◀
│                                       │
│  → Order creation FAILED (cleaned up) │
└───────────────────────────────────────┘
```

**Pros:** Clear flow, easy to understand and debug, centralized error handling
**Cons:** Orchestrator can become a god object; single point of failure

---

## Idempotency in Event-Driven Systems

Events will be delivered multiple times (at-least-once delivery). Consumers must handle duplicates:

```
Event: { "event_id": "evt_abc", "order_id": "12345", "amount": 99 }

Without idempotency:
  event_abc received → charge $99
  event_abc received again → charge $99 again!  ← double charge!

With idempotency:
  event_abc received → check: "have I processed evt_abc?" → No → charge $99, record evt_abc
  event_abc received again → check: "have I processed evt_abc?" → Yes → skip!
```

Store processed event IDs in a database (with expiry to save space).

---

## Event Schema Evolution

Events must remain parseable as schemas change. Use a schema registry (Confluent Schema Registry) and follow these rules:

```
✅ Backward-compatible changes (old consumers can read new events):
  - Add optional fields with defaults
  - Add new event types

❌ Breaking changes (will break old consumers):
  - Remove existing fields
  - Change field types
  - Rename fields

Version strategy: Embed schema version in event
  { "schema_version": 2, "event": "order.placed", ... }
```

---

## Real-World Event-Driven Systems

### LinkedIn
LinkedIn uses Kafka for its "data backbone." Every action (profile view, job application, connection request) is an event that flows through Kafka to multiple downstream systems: feed ranking, analytics, notifications, recommendations.

### Uber
Uber's dispatch system is event-driven. Every GPS location update, trip status change, and driver availability change is an event. The matching algorithm reacts to these events in real time.

### Netflix
Netflix's studio content pipeline is event-driven. When a new video is uploaded: events trigger transcoding, quality checks, metadata extraction, and CDN distribution — all as separate services reacting to events.

---

## When to Use Event-Driven Architecture

| Use Case | EDA Appropriate? | Why |
|----------|----------------|-----|
| User action needs immediate response | Partially | Use sync for response, events for side effects |
| Audit trail / compliance required | Yes | Events are a natural audit log |
| Multiple services react to same action | Yes | Events decouple all consumers |
| Complex multi-step workflows | Yes (Saga) | Distributed transactions without 2PC |
| Simple CRUD app | No | Adds complexity without benefit |

---

## Key Takeaways

1. **Events describe what happened** — past tense, immutable facts about the world
2. **Event notification decouples producers from consumers** — add consumers without changing producers
3. **Event sourcing stores the log of events, not just current state** — enables audit trails, time travel, and replay
4. **CQRS separates write and read models** — optimize each independently
5. **Sagas handle distributed transactions** — choreography for simplicity, orchestration for control
6. **Idempotency is mandatory** — events will be delivered multiple times; handle it
7. **Schema evolution requires discipline** — add optional fields, never remove or rename

---

*Previous: [Distributed Consensus ←](01-distributed-consensus.md) · Next: [Design a URL Shortener →](03-design-url-shortener.md)*
