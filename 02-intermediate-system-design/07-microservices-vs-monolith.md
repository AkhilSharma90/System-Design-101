# Microservices vs Monolith

The microservices vs monolith debate is one of the most charged in software engineering — and one of the most misunderstood. Neither is universally better. The right architecture depends on your team size, product maturity, and scaling requirements. This article gives you the frameworks to choose wisely.

---

## The Monolith

A monolithic application packages all functionality into a single deployable unit.

```
┌────────────────────────────────────────────────────┐
│                  Monolithic App                    │
│                                                    │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  User      │  │  Order     │  │  Payment    │  │
│  │  Module    │  │  Module    │  │  Module     │  │
│  └────────────┘  └────────────┘  └─────────────┘  │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  Product   │  │  Search    │  │  Notification│  │
│  │  Module    │  │  Module    │  │  Module     │  │
│  └────────────┘  └────────────┘  └─────────────┘  │
│                                                    │
│  Single process. Shared memory. Single database.   │
└────────────────────────────────────────────────────┘
              │
              ▼
         ┌──────────┐
         │ Database │
         └──────────┘
```

### Advantages of a Monolith

- **Simple to develop** — no network calls between modules; in-process function calls
- **Easy to test** — run the whole system locally with one command
- **Easy to deploy** — one artifact, one deployment pipeline
- **No distributed system complexity** — no service discovery, no network failures between components
- **Transactions are simple** — ACID transactions span the whole database
- **Easier to debug** — one log file, one process, one trace

### When Monoliths Fail

```
Team grows to 50+ engineers → everyone touching the same codebase
Build time: 20 minutes → deploys slow down
One team's bug → takes down the entire product
Checkout module needs 10x more CPU → must scale the entire app
```

---

## Microservices

Microservices decompose the application into small, independent services. Each service owns its data, its deployment, and its scaling.

```
                          API Gateway
                              │
         ┌────────────────────┼──────────────────┐
         ▼                    ▼                  ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  User        │    │  Order       │    │  Payment     │
│  Service     │    │  Service     │    │  Service     │
│              │    │              │    │              │
│  ┌────────┐  │    │  ┌────────┐  │    │  ┌────────┐  │
│  │ Users  │  │    │  │ Orders │  │    │  │ Txns   │  │
│  │  DB    │  │    │  │  DB    │  │    │  │  DB    │  │
│  └────────┘  │    │  └────────┘  │    │  └────────┘  │
└──────────────┘    └──────────────┘    └──────────────┘
         │                    │                  │
         └────────────────────┼──────────────────┘
                              │
                    ┌──────────────────┐
                    │  Message Bus     │  (async communication)
                    │  (Kafka)         │
                    └──────────────────┘
```

### Advantages of Microservices

- **Independent deployment** — teams ship without coordinating with other teams
- **Independent scaling** — scale the checkout service 10x without scaling everything
- **Technology freedom** — each service can use the right language/database
- **Fault isolation** — a bug in the notification service doesn't bring down payments
- **Team autonomy** — Conway's Law: architecture mirrors your org structure

### Disadvantages of Microservices

- **Distributed system complexity** — network calls fail, latency is variable
- **Data consistency** — cross-service transactions require Sagas, not simple ACID
- **Operational overhead** — dozens of services to deploy, monitor, and debug
- **Service discovery** — services need to find each other dynamically
- **Debugging is hard** — a request spans 5 services; which one failed?

---

## The Spectrum: Not Binary

```
Monolith ←────────────────────────────────────────────▶ Microservices

  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Big Ball │  │  Modular     │  │  Macro-      │  │  Micro-      │
  │ of Mud   │  │  Monolith    │  │  services    │  │  services    │
  │          │  │              │  │  (3–5 svcs)  │  │  (50+ svcs)  │
  └──────────┘  └──────────────┘  └──────────────┘  └──────────────┘
     Tangled       Clean             Right for         Right for
     code, no    modules but        medium teams      large orgs
     boundaries   1 deploy
```

**The modular monolith is often the best starting point** — clean module boundaries with the simplicity of one deployment. Modules can become services later if needed.

---

## Service Communication

### Synchronous (REST / gRPC)

```
Order Service ──HTTP POST /payments/charge──▶ Payment Service
              ◀──────200 OK {transaction_id}───

Pros:  Simple, immediate response, easy to reason about
Cons:  Coupling — if Payment Service is down, Order Service fails
       Cascading failures — one slow service slows everything
```

### Asynchronous (Message Queue)

```
Order Service ──publish "order.created" ──▶ Kafka ──▶ Payment Service
              (returns 202 Accepted immediately)       (processes async)

Pros:  Decoupled — services can fail independently
       Absorbs traffic spikes
Cons:  Eventual consistency — harder to debug and monitor
```

### When to Use Each

| Scenario | Synchronous | Asynchronous |
|----------|-------------|-------------|
| User needs immediate response | ✅ | ❌ |
| Critical path (checkout) | ✅ | Sometimes |
| Notification, email, analytics | ❌ | ✅ |
| Fire-and-forget side effects | ❌ | ✅ |
| Multiple downstream consumers | ❌ | ✅ |

---

## Data Management: Database per Service

A key principle of microservices: **each service owns its own database**. No service reads another service's database directly.

```
❌ Wrong (shared database):
  ┌──────────────┐     ┌──────────────┐
  │ Order Service│     │  User Service│
  └──────┬───────┘     └──────┬───────┘
         │ SQL JOIN            │
         └─────────────────────┘
                   │
            ┌──────────┐
            │ Shared DB│  ← Tight coupling, schema changes break both
            └──────────┘

✅ Right (database per service):
  ┌──────────────┐     ┌──────────────┐
  │ Order Service│     │  User Service│
  │  ┌────────┐  │     │  ┌────────┐  │
  │  │ Orders │  │     │  │ Users  │  │
  │  │  DB    │  │     │  │  DB    │  │
  │  └────────┘  │     │  └────────┘  │
  └──────────────┘     └──────────────┘
  Order needs user data? Call User Service API.
```

---

## API Gateway Pattern

The single entry point for all client requests. Handles cross-cutting concerns:

```
                     ┌──────────────────────────────┐
Client ──────────────▶│         API Gateway          │
                     │  - Authentication/Authorization│
                     │  - Rate Limiting               │
                     │  - SSL Termination             │
                     │  - Request Routing             │
                     │  - Load Balancing              │
                     │  - Circuit Breaking            │
                     │  - Request Aggregation         │
                     └─────────────────┬────────────┘
                                       │
              ┌────────────────────────┼──────────────┐
              ▼                        ▼              ▼
       User Service              Order Service   Payment Service
```

---

## Distributed Transactions: The Saga Pattern

Without a shared database, how do you handle operations that span multiple services?

**Problem:**
```
Place Order requires:
1. Reserve inventory (Inventory Service)
2. Charge payment (Payment Service)
3. Create order record (Order Service)

All must succeed or all must be undone.
```

### Choreography-Based Saga

Services react to each other's events. No central coordinator.

```
Order Service
  → emits: "order.payment_requested"
      └──▶ Payment Service charges card
            → emits: "payment.completed"
                └──▶ Inventory Service reserves items
                      → emits: "inventory.reserved"
                          └──▶ Order Service marks order confirmed

On failure:
  Payment Service fails
    → emits: "payment.failed"
        └──▶ Order Service cancels order (compensating transaction)
```

### Orchestration-Based Saga

A central Saga Orchestrator coordinates the transaction:

```
┌──────────────────┐
│  Saga Orchestrator│
│                  │
│ 1. Call Payment  │──▶ Payment Service
│ 2. Call Inventory│──▶ Inventory Service
│ 3. Call Order    │──▶ Order Service
│                  │
│ If step 2 fails: │
│ Compensate step 1│──▶ Refund payment
└──────────────────┘
```

---

## The Strangler Fig Pattern

Migrating from monolith to microservices without a big-bang rewrite:

```
Phase 1: New feature → build as service (not in monolith)
Phase 2: Extract high-value module from monolith → service
Phase 3: Another module → service
...
Phase N: Monolith hollowed out → decommission it

┌──────────────────────────────────┐
│ Monolith                         │  Phase 1
│  (most logic)                    │  ┌──────────┐
│                    ──────────────┼──▶ New Svc  │
└──────────────────────────────────┘  └──────────┘

┌─────────────────────┐
│ Monolith            │  Phase 2
│  (less logic)       │  ┌──────────┐  ┌──────────┐
│            ─────────┼──▶ Svc 1   │  │ Svc 2    │
└─────────────────────┘  └──────────┘  └──────────┘
```

---

## Conway's Law

> *"Organizations which design systems are constrained to produce designs which are copies of the communication structures of those organizations."* — Melvin Conway

In practice:
- 5-person startup → monolith is right
- 50-person company → modular monolith or a few services
- 500-person company → services aligned to team ownership

**Inverse Conway Maneuver:** Design your org structure to produce the architecture you want.

---

## The Monolith-First Recommendation

Martin Fowler's advice: **start with a monolith, extract services when needed**.

Why:
1. You can't know service boundaries without building the product first
2. Wrong service boundaries are harder to fix than a monolith refactor
3. The overhead of microservices at early stage slows velocity

**Extract to a service when:**
- The module has a dramatically different scaling requirement
- Different teams need to own it independently
- It needs a different tech stack (e.g., ML service in Python, main app in Go)
- It has clear, stable API boundaries

---

## Real-World Lessons

**Amazon (2001):** Broke monolith into services because teams were stepping on each other. Today: 1000+ services.

**Amazon Prime Video (2023):** Moved their video monitoring system *from* microservices *back to a monolith*. Cost dropped 90%. Sometimes the simplest architecture wins.

**Shopify:** Keeps a modular monolith (Rails) as their core. Only extracts services for truly different scaling needs. Serves billions in GMV.

---

## Key Takeaways

1. **Monoliths are simpler to build, deploy, and debug** — start here unless you have a strong reason not to
2. **Microservices enable independent scaling and team autonomy** — the benefit grows with org size
3. **The modular monolith is often the best of both worlds** — clean architecture without operational overhead
4. **Database per service prevents tight coupling** — services own their data and communicate via APIs
5. **Sagas handle distributed transactions** — choreography for simplicity, orchestration for control
6. **Use the Strangler Fig pattern** — migrate incrementally, don't rewrite everything at once
7. **Conway's Law is real** — your architecture will reflect your org structure whether you plan it or not

---

*Previous: [CDN and Edge Computing ←](06-cdn-and-edge-computing.md) · Next: [Search Systems →](08-search-systems.md)*
