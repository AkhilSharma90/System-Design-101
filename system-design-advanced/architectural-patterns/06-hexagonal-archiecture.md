---
slug: hexagonal-architecture
title: Hexagonal Archiecture
readTime: 20 min
orderIndex: 6
premium: false
---







 # Hexagonal Architecture (Ports & Adapters) for Distributed Systems — An Interactive Guide

> **Audience:** senior engineers building services in distributed environments (microservices, event-driven systems, multi-tenant SaaS, hybrid cloud).
> **Goal:** make Hexagonal Architecture *practical* under network failures, evolving dependencies, and production constraints.

---

## :dart: Challenge: Your service is “simple”… until it isn’t

You run **Order Service** for a delivery platform.

Today it:
- Accepts HTTP requests from a web app
- Calls Payment Provider A
- Writes to Postgres
- Publishes `OrderPlaced` events to Kafka

Tomorrow, product asks for:
- A second payment provider (regional)
- A CLI tool for batch imports
- A background worker that retries failed payments
- A gRPC API for mobile
- A new event bus for one region

And ops adds:
- Circuit breakers for flaky providers
- Chaos testing
- Multi-region failover

### Pause & think
If your domain logic is currently tangled with HTTP handlers, ORM entities, Kafka producers, and payment SDK calls…

**What changes will be hardest?**
1) Adding another API protocol (HTTP -> gRPC)
2) Swapping payment provider
3) Changing DB schema
4) Adding retries and idempotency

> Don’t answer yet—hold your intuition.

### The core problem
In distributed systems, dependencies change and fail. If your **business rules** are coupled to **delivery mechanisms** (HTTP, messaging) and **infrastructure** (DB, external providers), every change becomes a risky refactor.

**Hexagonal Architecture** (a.k.a. **Ports and Adapters**) is a disciplined way to:
- Keep the domain model insulated from I/O and frameworks
- Make dependencies replaceable
- Make failure handling explicit
- Enable multiple inputs/outputs without rewriting business logic

**Key insight:** In a distributed system, *network calls are not “implementation details”*. They are sources of latency, partial failure, retries, and inconsistency—Hexagonal Architecture helps you surface and manage that complexity.

> **Key insight box:**
> Hexagonal Architecture isn’t about “pretty folders.” It’s about **dependency direction**: the domain depends on *abstractions* (ports), while infrastructure depends on the domain.

---

## :handshake: Section: The restaurant analogy (the fastest mental model)

### Scenario
Imagine your service is a restaurant.
- The **kitchen** = domain logic (rules, invariants)
- **Waiters** taking orders = inbound adapters (HTTP controllers, gRPC handlers, consumers)
- **Suppliers** delivering ingredients = outbound adapters (DB, payment gateway, email, Kafka)

The kitchen shouldn’t care *who* brought the order (waiter, phone call, app). It also shouldn’t care *which* supplier delivered tomatoes—only that tomatoes arrive with a certain contract.

### Interactive question (pause & think)
If the restaurant switches from phone orders to app orders, should the chef rewrite recipes?

- A) Yes, the chef must learn the app’s API
- B) No, the chef only needs the order ticket format

**Answer (progressive reveal):**

<details>
<summary>Reveal</summary>
**B.** The chef shouldn’t know the app’s API. That’s the waiter’s job.
</details>

### Real-world parallel
In services:
- Domain logic shouldn’t know HTTP headers, JSON shapes, Kafka partitioning, or ORM sessions.
- It should operate on domain concepts: `Order`, `PaymentAuthorization`, `InventoryReservation`.

> **Key insight box:**
> **Ports** are the “order ticket format” and “supplier purchase order format.”
> **Adapters** translate between the outside world and those formats.

---

## :rotating_light: Section: Common Misconception — “Hexagonal means no frameworks”

### Misconception
> “If we use Hexagonal Architecture, we can’t use Spring, Django, Rails, or ORM libraries.”

### Reality
You can use any framework. Hexagonal says:
- Framework code should live in **adapters**
- The **domain** should be framework-agnostic

### Why it matters in distributed systems
Frameworks are great until:
- You need to run the same domain logic in a **worker** and an **API**
- You need to test failure behavior without starting Kafka/Postgres
- You need to swap a dependency (payment provider, event bus)

Hexagonal doesn’t ban frameworks—it **contains** them.

> **Key insight box:**
> Hexagonal Architecture is a *boundary management strategy*, not a framework boycott.

---

## :mag: Section: What “Ports” and “Adapters” actually are (with distributed systems twist)

### Scenario
Your Order Service has:
- inbound: HTTP `POST /orders`, Kafka `CartCheckedOut` events
- outbound: Postgres, Payment API, Kafka `OrderPlaced` events

### Ports
A **port** is an interface (or abstract contract) that describes:
- What the domain *needs* from the outside (outbound port)
- What the outside can *ask* the domain to do (inbound port)

In distributed systems, ports are where you encode:
- Idempotency expectations
- Timeouts and cancellation semantics
- Error taxonomy (retryable vs non-retryable)
- Consistency requirements

### Adapters
An **adapter** implements a port by translating:
- protocol <-> domain model
- SDK <-> domain model
- DB rows <-> aggregates

Adapters are where you handle:
- serialization/deserialization
- retries, circuit breakers
- authentication
- schema evolution
- backpressure

### Interactive: classify these
Match each item to **Port** or **Adapter**.

1) `PaymentGateway.authorize(paymentRequest)`
2) `StripePaymentAdapter` using Stripe SDK
3) HTTP controller parsing JSON into `PlaceOrderCommand`
4) `PlaceOrderUseCase` interface

Pause & think.

<details>
<summary>Reveal</summary>
1) **Port** (outbound)
2) **Adapter** (outbound)
3) **Adapter** (inbound)
4) **Port** (inbound)
</details>

> **Key insight box:**
> Ports are *stable contracts*. Adapters are *replaceable translators*.

---

## :jigsaw: Section: The hexagon diagram (and what it hides)

Hexagonal Architecture is often drawn as a hexagon with:
- domain in the center
- ports on the edges
- adapters outside

[IMAGE: A hexagon diagram labeled “Domain” in center; left side inbound adapters (HTTP, gRPC, Kafka consumer, CLI); right side outbound adapters (Postgres repository, Kafka producer, Payment provider, Email). Arrows show dependencies pointing inward from adapters to ports and domain. Include “dependency inversion” note.]

### Pause & think
In real distributed systems, what’s missing from the simple hexagon picture?

- A) Observability (logs/metrics/traces)
- B) Failure modes (timeouts, retries)
- C) Data consistency patterns (outbox, sagas)
- D) All of the above

<details>
<summary>Reveal</summary>
**D.** The basic diagram is conceptual; production systems require explicit patterns for these concerns.
</details>

### The distributed-systems upgrade
For distributed systems, treat “adapters” as *policy-bearing components*:
- outbound adapters enforce timeouts, retries, circuit breakers
- inbound adapters enforce idempotency keys, rate limits, auth

> **Key insight box:**
> In distributed systems, adapters are not “dumb glue.” They are where you implement *resilience policy*.

---

## :video_game: Section: Decision game — Which dependency direction is correct?

### Scenario
You have a domain use case `PlaceOrder` that needs to:
- persist an order
- call payment provider
- publish an event

Which structure is *hexagonal*?

**Option 1**
- `PlaceOrderService` imports `StripeSDK`, `KafkaProducer`, `OrderJpaRepository`

**Option 2**
- `PlaceOrderService` depends on `PaymentPort`, `OrderRepositoryPort`, `EventPublisherPort`
- Adapters implement those ports using Stripe/Kafka/JPA

Pause & think.

<details>
<summary>Reveal</summary>
**Option 2** is hexagonal. The domain depends on ports (abstractions), not concrete infrastructure.
</details>

### Why Option 1 fails harder in distributed systems
Because network and infrastructure concerns leak into business logic:
- retry loops inside domain methods
- HTTP status mapping inside domain
- Kafka partition keys computed deep in aggregates

> **Key insight box:**
> If your domain imports an SDK for something that can fail over the network, you’ve coupled your business rules to failure mechanics.

---

## :potable_water: Section: Inbound side — Ports for use cases (not controllers)

### Scenario
Your current code:
- Controller validates JSON
- Controller calls repository
- Controller calls payment
- Controller returns response

That’s a “transaction script” living at the edge.

### The hexagonal move
Define an **inbound port** per use case (application service boundary):
- `PlaceOrder`
- `CancelOrder`
- `GetOrder`

The controller becomes an adapter that:
- maps HTTP request -> command
- calls the port
- maps result -> HTTP response

### Think about it
If you add a Kafka consumer that triggers `PlaceOrder` from `CartCheckedOut`, what changes?

- A) You rewrite business logic for Kafka
- B) You add another inbound adapter that calls the same port

<details>
<summary>Reveal</summary>
**B.** Same use case, different adapter.
</details>

[CODE: JavaScript, demonstrate an inbound port interface `PlaceOrderUseCase`, a command DTO, and two adapters: HTTP controller and Kafka consumer calling the same port.]

```javascript
// Implementing inbound port + two inbound adapters (HTTP + Kafka) calling same use case
'use strict';

class PlaceOrderUseCase {
  async execute(cmd, ctx) { throw new Error('Not implemented'); }
}

function httpPlaceOrderHandler(useCase) {
  return async (req, res) => {
    try {
      const cmd = { userId: req.body.userId, items: req.body.items, requestId: req.headers['x-request-id'] };
      const result = await useCase.execute(cmd, { deadlineMs: 1500 });
      res.status(201).json(result);
    } catch (err) {
      res.status(502).json({ error: 'PLACE_ORDER_FAILED', detail: err.message });
    }
  };
}

async function kafkaCartCheckedOutConsumer(useCase, msg) {
  try {
    const cmd = { userId: msg.userId, items: msg.items, requestId: msg.eventId };
    await useCase.execute(cmd, { deadlineMs: 1500 });
  } catch (err) {
    console.error('consumer failed; send to DLQ', { eventId: msg.eventId, err: err.message });
  }
}

// Usage: wire httpPlaceOrderHandler(useCase) and kafkaCartCheckedOutConsumer(useCase, msg)
```

> **Key insight box:**
> Inbound adapters multiply; inbound ports stay stable.

---

## :mag: Section: Outbound side — Ports for dependencies (DB, messaging, providers)

### Scenario
Your domain needs:
- store/retrieve orders
- authorize/charge payment
- publish events

In hexagonal architecture, the domain defines *what it needs*:
- `OrderRepositoryPort`
- `PaymentPort`
- `EventPublisherPort`

Adapters implement those ports:
- Postgres adapter (SQL/ORM)
- Stripe adapter (HTTP SDK)
- Kafka adapter (producer)

### Pause & think: where do retries belong?
Retries for payment authorization should live in:
- A) Domain use case
- B) Payment adapter
- C) HTTP controller

<details>
<summary>Reveal</summary>
**B** most often. Retries are a policy for interacting with an unreliable dependency.

But: the domain may still need to decide *whether* to retry (business policy) vs adapter deciding *how* (technical policy). We’ll split that later.
</details>

> **Key insight box:**
> Distributed systems force you to distinguish **business policy** (should we retry?) from **resilience mechanism** (how do we retry?).

---

## :jigsaw: Section: The “two-layer” mental model — Domain vs Application vs Adapters

### Mental model
Hexagonal is easiest when you separate:

1) **Domain layer** (pure):
   - entities/aggregates
   - value objects
   - domain services
   - invariants

2) **Application layer** (use cases):
   - orchestrates domain objects
   - manages transactions
   - calls outbound ports
   - defines inbound ports (use case interfaces)

3) **Adapters/Infrastructure**:
   - HTTP/gRPC endpoints
   - Kafka consumers/producers
   - DB repositories
   - external clients

### Restaurant mapping
- Domain: recipes and food safety rules
- Application: expediter coordinating dishes, timing, plating
- Adapters: waiters, delivery apps, suppliers

### Interactive exercise: spot the leak
Which is a sign your domain layer is leaking infrastructure?

1) Domain entity has `@JsonProperty` annotations
2) Use case takes `HttpServletRequest`
3) Repository adapter imports domain types
4) Domain throws `PaymentGatewayTimeoutException`

Pause & think.

<details>
<summary>Reveal</summary>
1) leak (serialization concerns)
2) leak (transport concerns)
3) fine (adapters depend inward)
4) likely leak (infrastructure failure types leaking into domain)
</details>

> **Key insight box:**
> Domain should speak in domain terms. Failure types should be mapped at the boundary.

---

## :dart: Section: Distributed systems reality — failures are part of the contract

### Scenario
Payment provider sometimes:
- times out
- returns 500
- returns 409 duplicate charge
- succeeds but your network drops before you receive response

### Pause & think
Which statement is true?

A) “Hexagonal architecture eliminates failures by isolating them.”
B) “Hexagonal architecture makes failures explicit by isolating where they are handled.”

<details>
<summary>Reveal</summary>
**B.** It doesn’t remove failures; it makes your codebase honest about them.
</details>

### Failure taxonomy as port design
Design ports so failure modes are explicit and testable:
- return `Result<T, Error>`
- or throw typed domain-level errors
- include idempotency keys
- include timeout/cancellation semantics

[CODE: JavaScript, demonstrate a `PaymentPort.authorize()` returning a sealed result with retryable/non-retryable errors, plus idempotency key.]

```javascript
// Implementing PaymentPort with explicit failure contract (retryable vs non-retryable)
'use strict';

class PaymentPort {
  /** @returns {Promise<{ok:true, authId:string}|{ok:false, retryable:boolean, code:string, message:string}>} */
  async authorize({ amountCents, currency, idempotencyKey }, ctx) { throw new Error('Not implemented'); }
}

class HttpPaymentAdapter extends PaymentPort {
  constructor(fetchFn, baseUrl) { super(); this.fetch = fetchFn; this.baseUrl = baseUrl; }
  async authorize(req, ctx = {}) {
    const controller = new AbortController();
    const t = setTimeout(() => controller.abort(), ctx.deadlineMs ?? 1500);
    try {
      const r = await this.fetch(`${this.baseUrl}/authorize`, {
        method: 'POST', signal: controller.signal,
        headers: { 'content-type': 'application/json', 'idempotency-key': req.idempotencyKey },
        body: JSON.stringify({ amountCents: req.amountCents, currency: req.currency })
      });
      if (r.status === 409) return { ok: false, retryable: false, code: 'DUPLICATE', message: 'Duplicate charge' };
      if (!r.ok) return { ok: false, retryable: r.status >= 500, code: `HTTP_${r.status}`, message: await r.text() };
      const body = await r.json();
      return { ok: true, authId: body.authorizationId };
    } catch (e) {
      return { ok: false, retryable: true, code: 'TIMEOUT_OR_NETWORK', message: e.message };
    } finally { clearTimeout(t); }
  }
}

// Usage: await new HttpPaymentAdapter(fetch, 'https://pay').authorize({amountCents:1000,currency:'USD',idempotencyKey:'req-1'},{deadlineMs:800})
```

> **Key insight box:**
> In distributed systems, a port is not just “a method signature.” It’s a **failure contract**.

---

## :handshake: Section: Timeouts, cancellation, and backpressure (ports must carry them)

### Scenario
Your `PlaceOrder` use case calls Payment and Inventory. Under load, requests pile up.

If your port methods don’t accept context/deadlines, your adapters may:
- block threads
- keep retrying after the client gave up
- overload downstream services

### Pause & think
Where should deadlines live?

- A) Only in HTTP layer
- B) Only in adapters
- C) Propagated from inbound adapter -> use case -> outbound adapters

<details>
<summary>Reveal</summary>
**C.** Deadline propagation is end-to-end.
</details>

### Practical patterns
- Pass a `Context` / `CancellationToken` / `CoroutineContext`
- Include `timeoutMs` in command
- Use structured concurrency where possible

[CODE: Python, show deadline propagation from inbound adapter to use case to outbound port using sockets with timeouts.]

```python
# Implementing deadline propagation across ports using socket timeouts
import socket
import time

class PaymentPort:
    def authorize(self, amount_cents: int, idem_key: str, deadline_s: float) -> str:
        raise NotImplementedError

class TcpPaymentAdapter(PaymentPort):
    def __init__(self, host: str, port: int):
        self.host, self.port = host, port

    def authorize(self, amount_cents: int, idem_key: str, deadline_s: float) -> str:
        timeout = max(0.05, deadline_s - time.time())  # propagate remaining budget
        try:
            with socket.create_connection((self.host, self.port), timeout=timeout) as s:
                s.settimeout(timeout)
                s.sendall(f"AUTH {amount_cents} {idem_key}\n".encode())
                return s.recv(1024).decode().strip()
        except (OSError, socket.timeout) as e:
            raise TimeoutError(f"payment call failed within deadline: {e}")

def place_order(use_payment: PaymentPort, amount_cents: int, request_id: str, deadline_ms: int):
    deadline_s = time.time() + (deadline_ms / 1000.0)
    return use_payment.authorize(amount_cents, request_id, deadline_s)

# Usage: place_order(TcpPaymentAdapter('127.0.0.1', 9000), 1200, 'req-123', 800)
```

> **Key insight box:**
> Hexagonal boundaries help you *propagate* distributed-systems concerns without coupling to specific transports.

---

## :rotating_light: Section: Common Misconception — “Ports are just interfaces, so we’re done”

### Misconception
> “We created interfaces for repositories. That’s hexagonal.”

### Reality
Interfaces alone don’t buy much unless:
- dependency direction is correct
- adapters are outside
- ports represent use cases and dependency contracts
- failure semantics are modeled

### Distributed systems angle
If your `PaymentPort` exposes Stripe-specific fields (like `paymentIntentId`) everywhere, you’ve created a “Stripe-shaped domain.”

> **Key insight box:**
> Ports should be **domain-shaped**, not vendor-shaped.

---

## :mag: Section: Event-driven systems — inbound adapters aren’t just HTTP

### Scenario
You consume `CartCheckedOut` events and create orders.

Where does deserialization and schema evolution live?
- In the consumer adapter.

Where does deduplication/idempotency live?
- Split: adapter enforces *delivery semantics*; domain enforces *business semantics*.

### Pause & think
Kafka delivers at-least-once. Your consumer may see duplicates.

Which statement is true?

A) “Hexagonal architecture guarantees exactly-once processing.”
B) “Hexagonal architecture lets you implement idempotency in one place and reuse it across transports.”

<details>
<summary>Reveal</summary>
**B.** The architecture helps structure the solution; it doesn’t change Kafka semantics.
</details>

### Pattern: Idempotency as an outbound port
A robust approach:
- Use case receives an idempotency key (`eventId` or `requestId`)
- Use case uses an `IdempotencyStorePort` (backed by DB/Redis)
- If already processed, return prior result

[CODE: Python, show an idempotent `PlaceOrder` use case using an `IdempotencyStorePort` and `OrderRepositoryPort`.]

```python
# Implementing idempotent use case with an IdempotencyStorePort
from dataclasses import dataclass

class IdempotencyStorePort:
    def get(self, key: str): raise NotImplementedError
    def put(self, key: str, value: dict): raise NotImplementedError

class OrderRepositoryPort:
    def save(self, order: dict) -> str: raise NotImplementedError

@dataclass(frozen=True)
class PlaceOrderCommand:
    user_id: str
    items: list
    request_id: str

class PlaceOrderUseCase:
    def __init__(self, idem: IdempotencyStorePort, repo: OrderRepositoryPort):
        self.idem, self.repo = idem, repo

    def execute(self, cmd: PlaceOrderCommand) -> dict:
        cached = self.idem.get(cmd.request_id)
        if cached is not None:  # duplicate delivery -> return prior result
            return cached
        if not cmd.items:
            raise ValueError("order must contain at least one item")
        order_id = self.repo.save({"userId": cmd.user_id, "items": cmd.items})
        result = {"orderId": order_id, "status": "PLACED"}
        self.idem.put(cmd.request_id, result)
        return result

# Usage: PlaceOrderUseCase(idem_store, order_repo).execute(PlaceOrderCommand('u1',[{'sku':'x','qty':1}],'evt-9'))
```

> **Key insight box:**
> In event-driven systems, idempotency is a first-class dependency—model it as a port.

---

## :jigsaw: Section: Outbox pattern — where it fits in hexagonal

### Scenario
You need to:
- save order in DB
- publish `OrderPlaced` event

If you do DB then Kafka directly, you risk:
- DB commit succeeds, Kafka publish fails -> missing event
- Kafka publish succeeds, DB commit fails -> phantom event

### Pause & think
Which is the safer approach?

A) “Try/catch and retry Kafka publish”
B) “Transactional outbox: write event to DB in same transaction, publish asynchronously”

<details>
<summary>Reveal</summary>
**B.** Outbox provides atomicity with the database.
</details>

### Hexagonal placement
- Domain/use case calls `EventOutboxPort.append(event)`
- DB adapter writes to `outbox` table transactionally
- Separate publisher adapter reads outbox and publishes to Kafka

[IMAGE: Sequence diagram: PlaceOrderUseCase -> DB transaction writes Order + Outbox row; OutboxPublisher polls table -> Kafka publish; marks outbox row sent. Show failure points and retries.]

[CODE: Python, show outbox publisher loop with idempotent publish using event id (socket-based publisher stub).]

```python
# Implementing outbox publisher loop (poll -> publish -> mark sent) with socket networking
import json
import socket
import time

class OutboxStorePort:
    def fetch_batch(self, limit: int): raise NotImplementedError
    def mark_sent(self, event_id: str): raise NotImplementedError

def publish_event(host: str, port: int, event: dict, timeout_s: float = 1.0) -> None:
    payload = (json.dumps(event) + "\n").encode()
    try:
        with socket.create_connection((host, port), timeout=timeout_s) as s:
            s.settimeout(timeout_s)
            s.sendall(payload)
            ack = s.recv(16).decode().strip()
            if ack != "OK":
                raise RuntimeError(f"broker rejected event: {ack}")
    except OSError as e:
        raise ConnectionError(f"publish failed: {e}")

def outbox_publisher_loop(store: OutboxStorePort, broker_host: str, broker_port: int, stop_flag):
    while not stop_flag.is_set():
        for row in store.fetch_batch(limit=50):
            event = {"id": row["id"], "type": row["type"], "payload": row["payload"]}
            try:
                publish_event(broker_host, broker_port, event)
                store.mark_sent(row["id"])  # idempotent: safe to call once per event id
            except Exception as e:
                time.sleep(0.2)  # backoff; keep event in outbox for retry
        time.sleep(0.5)

# Usage: run outbox_publisher_loop(outbox_store, '127.0.0.1', 9099, stop_event)
```

> **Key insight box:**
> Hexagonal architecture doesn’t replace distributed-systems patterns like Outbox—it gives them a clean home.

---

## :video_game: Section: Matching exercise — pick the right port for the job

Match the distributed-systems concern to the port/adapter location.

| Concern | Best home |
|---|---|
| JSON schema validation | Inbound adapter |
| Retry with exponential backoff for Payment API | Outbound adapter |
| Business rule: “don’t retry payment after 24h” | Use case / domain policy |
| Event deduplication based on `eventId` | Use case + idempotency port |
| Mapping domain error to HTTP 409 | Inbound adapter |
| Circuit breaker state | Outbound adapter |

Pause & think—then check.

<details>
<summary>Reveal</summary>
All rows as shown are correct.
</details>

> **Key insight box:**
> Put **protocol translation** at the edges, **business policy** in the center, and **resilience mechanics** near outbound calls.

---

## :potable_water: Section: Trade-offs — what you pay for hexagonal

### Scenario
Your team is under pressure to ship. Someone says:
> “This is too much ceremony. We can just put everything in controllers.”

### The real trade-offs
Hexagonal Architecture increases:
- number of types (ports, adapters, DTOs)
- indirection
- upfront design effort

But decreases:
- coupling
- change cost
- integration-test dependence
- blast radius of dependency swaps

### Comparison table

| Dimension | Layered MVC (typical) | Hexagonal (ports/adapters) |
|---|---|---|
| Multiple inbound protocols | Often duplicated logic | Reuse use cases via multiple adapters |
| Swapping DB/provider | High refactor risk | Adapter replacement (if port is stable) |
| Testing business logic | Often needs DB/framework | Pure unit tests around ports |
| Failure modeling | Often ad hoc | Explicit at port boundaries |
| Cognitive load | Lower initially | Lower over time (if disciplined) |

### Pause & think
When is hexagonal *not* worth it?

- A) One-off scripts
- B) Short-lived prototypes
- C) Tiny services with no external dependencies
- D) All of the above

<details>
<summary>Reveal</summary>
**D.** But beware: distributed systems tend to grow dependencies quickly.
</details>

> **Key insight box:**
> Hexagonal Architecture is an investment. It pays off when dependencies and change rates are high.

---

## :mag: Section: Real-world usage patterns (microservices, modular monoliths, edge services)

### Pattern 1: Microservice with multiple adapters
- Inbound: HTTP + Kafka consumer
- Outbound: Postgres + Kafka + Payment provider

Hexagonal helps you:
- keep one domain model
- reuse use cases
- test without spinning infra

### Pattern 2: Modular monolith
Each module is a “hexagon” with:
- internal ports
- adapters between modules

Distributed-systems twist:
- modules may later become services; ports become service contracts.

### Pattern 3: Edge service / BFF
Mostly adapters:
- orchestrates calls to downstream services
- little domain logic

Hexagonal still helps:
- isolate protocol translation
- standardize resilience and observability

> **Key insight box:**
> Hexagonal is not tied to microservices. It’s a modularity technique that scales from monolith to distributed.

---

## :jigsaw: Section: Observability as an adapter concern (but with domain-friendly hooks)

### Scenario
You need traces across:
- inbound HTTP
- use case execution
- outbound payment call
- DB write

### Pause & think
Where should you create spans and record metrics?

- A) Only in domain
- B) Only in adapters
- C) Adapters create/propagate context; domain emits semantic events

<details>
<summary>Reveal</summary>
**C.** Domain shouldn’t import OpenTelemetry, but it can expose meaningful events.
</details>

### Practical approach
- Inbound adapter starts trace/span and passes context
- Outbound adapters create child spans for network calls
- Domain/use case logs semantic events via a `DomainEventsPort` or returns structured outcomes

[CODE: Python, show an adapter wrapping a call to `PlaceOrderUseCase` and outbound adapters creating spans (minimal tracer shim).]

```python
# Implementing observability at adapters: trace wrapper + outbound span around socket call
import time
import socket

class Tracer:
    def span(self, name: str):
        start = time.time()
        class _Span:
            def __enter__(self_inner): return self_inner
            def __exit__(self_inner, exc_type, exc, tb):
                dur_ms = int((time.time() - start) * 1000)
                print(f"span={name} durationMs={dur_ms} error={bool(exc)}")
        return _Span()

def http_adapter(place_order_use_case, tracer: Tracer):
    def handler(req_json: dict):
        with tracer.span("http.place_order"):
            return place_order_use_case.execute(req_json)  # domain/app stays telemetry-free
    return handler

class PaymentAdapter:
    def __init__(self, tracer: Tracer, host: str, port: int):
        self.tracer, self.host, self.port = tracer, host, port

    def authorize(self, payload: str) -> str:
        with self.tracer.span("payment.authorize"):
            try:
                with socket.create_connection((self.host, self.port), timeout=1.0) as s:
                    s.sendall((payload + "\n").encode())
                    return s.recv(128).decode().strip()
            except OSError as e:
                raise ConnectionError(f"payment network error: {e}")

# Usage: handler = http_adapter(use_case, Tracer()); handler({'userId':'u1','items':[1]})
```

> **Key insight box:**
> Keep telemetry libraries at the edge; keep semantics in the center.

---

## :rotating_light: Section: Common Misconception — “Domain events = Kafka events”

### Misconception
> “If the domain emits `OrderPlaced`, that must be the Kafka message.”

### Reality
- **Domain events** are internal facts for your model.
- **Integration events** are messages for other services.

They can align, but they often differ:
- integration events need versioning, backward compatibility, PII filtering
- domain events may be richer and not stable externally

### Hexagonal placement
- Domain raises domain events
- Application layer decides which integration events to publish
- Outbound adapter publishes to Kafka

> **Key insight box:**
> Don’t let external contracts dictate your internal model.

---

## :dart: Section: Designing ports for evolution (versioning, compatibility, migrations)

### Scenario
Payment Provider A returns `riskScore`. Provider B doesn’t.

If your port requires `riskScore`, you can’t adopt Provider B.

### Pause & think
What’s better?

A) Put every provider field into the port contract
B) Keep port minimal; expose domain-level concepts; allow optional metadata

<details>
<summary>Reveal</summary>
**B.** Ports should be stable and domain-focused.
</details>

### Techniques
- Use domain value objects instead of vendor DTOs
- Add optional `metadata: Map<String,String>` only when necessary
- Prefer additive changes
- Create separate ports for separate use cases (authorization vs capture)

> **Key insight box:**
> Port design is API design—treat it with the same rigor as public service APIs.

---

## :video_game: Section: Quiz — Identify the adapter boundary violations

### Code smell scenarios
Which one violates hexagonal boundaries most?

1) `Order` entity has `toJson()`
2) `PaymentAdapter` returns `StripePaymentIntent`
3) `PlaceOrderUseCase` returns a domain `OrderId`
4) Kafka producer adapter imports `OrderPlacedIntegrationEvent`

Pause & think.

<details>
<summary>Reveal</summary>
**2** is the biggest violation: the outbound adapter is leaking vendor types through the port into the application/domain.

**1** is also a violation (serialization in domain), but it’s often easier to fix.
</details>

> **Key insight box:**
> Vendor types should not cross port boundaries.

---

## :jigsaw: Section: A full walk-through — Place Order in a flaky world

We’ll build a mental implementation with:
- inbound HTTP adapter
- `PlaceOrderUseCase` (application)
- domain model with invariants
- outbound ports: repository, payment, outbox, idempotency
- adapters: Postgres, Stripe, Kafka outbox publisher

### Step 1: Define the inbound port
- `placeOrder(cmd, ctx) -> result`

### Step 2: Define outbound ports
- `OrderRepositoryPort`
- `PaymentPort`
- `OutboxPort`
- `IdempotencyStorePort`

### Step 3: Implement the use case
Orchestration responsibilities:
- validate business rules
- enforce idempotency
- call payment with idempotency key
- store order + outbox event transactionally

### Step 4: Implement adapters
- HTTP adapter maps request/response
- Payment adapter handles retries/timeouts/circuit breaker
- Repository adapter handles mapping to tables
- Outbox publisher handles eventual publish

[CODE: Python, show simplified but complete skeleton: ports, use case implementation, HTTP adapter, payment adapter stub, repository adapter stub, outbox append.]

```python
# Implementing hexagonal skeleton: ports + use case + HTTP adapter wiring
from dataclasses import dataclass

class OrderRepositoryPort:
    def save(self, order: dict) -> str: raise NotImplementedError

class PaymentPort:
    def authorize(self, amount_cents: int, idem_key: str) -> str: raise NotImplementedError

class OutboxPort:
    def append(self, event: dict) -> None: raise NotImplementedError

@dataclass(frozen=True)
class PlaceOrderCommand:
    user_id: str
    items: list
    request_id: str

class PlaceOrderUseCase:
    def __init__(self, repo: OrderRepositoryPort, pay: PaymentPort, outbox: OutboxPort):
        self.repo, self.pay, self.outbox = repo, pay, outbox

    def execute(self, cmd: PlaceOrderCommand) -> dict:
        if not cmd.items:
            raise ValueError("empty order")
        auth_id = self.pay.authorize(amount_cents=1000, idem_key=cmd.request_id)
        order_id = self.repo.save({"userId": cmd.user_id, "items": cmd.items, "authId": auth_id})
        self.outbox.append({"type": "OrderPlaced", "orderId": order_id, "requestId": cmd.request_id})
        return {"orderId": order_id, "status": "PLACED"}

def http_adapter(use_case: PlaceOrderUseCase, req_json: dict, headers: dict) -> tuple[int, dict]:
    try:
        cmd = PlaceOrderCommand(req_json["userId"], req_json["items"], headers.get("x-request-id", ""))
        return 201, use_case.execute(cmd)
    except (KeyError, ValueError) as e:
        return 400, {"error": "BAD_REQUEST", "detail": str(e)}
    except Exception as e:
        return 502, {"error": "UPSTREAM_FAILURE", "detail": str(e)}

# Usage: status, body = http_adapter(use_case, {"userId":"u1","items":[1]}, {"x-request-id":"r1"})
```

### Pause & think: where does the transaction live?

- A) In the repository adapter
- B) In the use case
- C) In an application-layer transaction boundary (unit-of-work) invoked by adapter

<details>
<summary>Reveal</summary>
**C** is common: the inbound adapter (or a framework integration adapter) starts a transaction around the use case call.

But some teams keep transaction handling in the use case. The key is: the domain entities should not manage transactions.
</details>

> **Key insight box:**
> Transactions are infrastructure. Keep them out of the domain; coordinate them at the application boundary.

---

## :potable_water: Section: Testing strategy — unit tests that simulate distributed failures

### Scenario
You want to test:
- payment timeout
- duplicate event delivery
- DB transient failure

Hexagonal makes this easy because ports are mockable.

### Pause & think
What should be the default test pyramid for a hexagonal service?

A) Mostly end-to-end tests with real Kafka/Postgres
B) Mostly unit tests for use cases + a few contract/integration tests per adapter

<details>
<summary>Reveal</summary>
**B.** Use cases can be tested with fake ports; adapters need integration tests.
</details>

### Suggested test matrix

| Layer | Test type | What to validate |
|---|---|---|
| Domain | unit/property tests | invariants, state transitions |
| Use case (application) | unit tests with fakes | orchestration, error mapping |
| Adapters | integration tests | DB schema mapping, SDK behavior |
| System | end-to-end | wiring, configuration, deployment |

[CODE: Python, show a unit test where PaymentPort fake times out, verifying use case returns retryable error and writes nothing.]

```python
# Implementing deterministic failure test using fake ports (no Docker required)
class FakeRepo:
    def __init__(self): self.saved = 0
    def save(self, order: dict) -> str:
        self.saved += 1
        return "o-1"

class TimeoutPayment:
    def authorize(self, amount_cents: int, idem_key: str) -> str:
        raise TimeoutError("payment timeout")

class FakeOutbox:
    def __init__(self): self.events = []
    def append(self, event: dict) -> None:
        self.events.append(event)

def test_place_order_payment_timeout_does_not_persist_or_emit():
    repo, pay, outbox = FakeRepo(), TimeoutPayment(), FakeOutbox()
    uc = PlaceOrderUseCase(repo, pay, outbox)
    try:
        uc.execute(PlaceOrderCommand("u1", [1], "req-1"))
        assert False, "expected TimeoutError"
    except TimeoutError:
        pass
    assert repo.saved == 0, "should not save order when payment fails"
    assert outbox.events == [], "should not append outbox event when payment fails"

# Usage: run test_place_order_payment_timeout_does_not_persist_or_emit() in your test runner
```

> **Key insight box:**
> Hexagonal architecture turns “distributed failure tests” into deterministic unit tests—when modeled correctly.

---

## :mag: Section: Handling consistency and sagas with ports

### Scenario
Placing an order requires:
- reserve inventory (Inventory Service)
- authorize payment (Payment Provider)
- confirm order

This is a distributed transaction.

### Pause & think
Which is true?

A) Hexagonal architecture solves distributed transactions by design
B) Hexagonal architecture helps you implement sagas by keeping orchestration in the application layer and side-effects behind ports

<details>
<summary>Reveal</summary>
**B.** You still need patterns like sagas/compensation.
</details>

### Where saga logic lives
- Application layer use case orchestrates steps
- Outbound ports represent steps
- Adapters implement calls
- Compensation is domain/application policy

[IMAGE: Saga flow diagram with steps and compensations: ReserveInventory -> AuthorizePayment -> ConfirmOrder; compensation arrows for failures.]

[CODE: JavaScript, show saga orchestration in `PlaceOrderUseCase` with compensation on failure.]

```javascript
// Implementing saga orchestration in application layer with compensation
'use strict';

async function placeOrderSaga({ inventoryPort, paymentPort, orderRepo }, cmd) {
  let reservationId = null;
  try {
    reservationId = await inventoryPort.reserve(cmd.items, { requestId: cmd.requestId });
    const auth = await paymentPort.authorize({ amountCents: cmd.amountCents, idempotencyKey: cmd.requestId });
    if (!auth.ok) throw Object.assign(new Error(auth.message), { retryable: auth.retryable });
    const orderId = await orderRepo.save({ userId: cmd.userId, items: cmd.items, reservationId, authId: auth.authId });
    return { ok: true, orderId };
  } catch (err) {
    // Compensation should be best-effort and safe to retry.
    if (reservationId) {
      try { await inventoryPort.release(reservationId, { requestId: cmd.requestId }); }
      catch (e) { console.error('compensation failed; enqueue for retry', e.message); }
    }
    return { ok: false, retryable: !!err.retryable, error: err.message };
  }
}

// Usage: await placeOrderSaga({inventoryPort, paymentPort, orderRepo}, {userId:'u1',items:[...],amountCents:1000,requestId:'r1'})
```

> **Key insight box:**
> Ports make saga steps explicit and mockable; adapters keep network chaos out of the core.

---

## :rotating_light: Section: Common Misconception — “Hexagonal = Clean Architecture = DDD = same thing”

### Reality
They overlap but differ:
- **Hexagonal**: focus on ports/adapters and dependency direction
- **Clean Architecture**: similar, emphasizes concentric circles and use cases
- **DDD**: modeling approach (bounded contexts, aggregates)

You can use Hexagonal without DDD, and DDD without Hexagonal.

Distributed systems note:
- DDD helps define service boundaries
- Hexagonal helps implement each service cleanly

> **Key insight box:**
> Use DDD to decide *what* services should be; use Hexagonal to decide *how* each service integrates with the world.

---

## :video_game: Section: “Which statement is true?” — trade-off edition

1) “Ports should expose CRUD methods for every entity.”
2) “Ports should model capabilities needed by use cases.”

Pause & think.

<details>
<summary>Reveal</summary>
**2**. CRUD ports often leak persistence concerns and encourage anemic domains.
</details>

3) “Adapters can depend on domain types.”
4) “Domain can depend on adapter types.”

<details>
<summary>Reveal</summary>
**3** is true; **4** breaks dependency direction.
</details>

> **Key insight box:**
> Ports align to **use cases**, not tables.

---

## :jigsaw: Section: Practical folder/module layout (language-agnostic)

A common layout:

- `domain/`
  - `model/` (entities, value objects)
  - `policy/` (domain services)
- `application/`
  - `ports/in/` (use case interfaces)
  - `ports/out/` (dependency interfaces)
  - `usecases/` (implementations)
- `adapters/`
  - `in/http/`
  - `in/kafka/`
  - `out/postgres/`
  - `out/kafka/`
  - `out/payment/`
- `bootstrap/` (wiring/DI)

### Pause & think
Where does configuration live (URLs, credentials, topic names)?

- A) Domain
- B) Application
- C) Bootstrap/adapters

<details>
<summary>Reveal</summary>
**C.** Configuration is infrastructure.
</details>

> **Key insight box:**
> If your domain knows topic names, you’ve built a Kafka-shaped domain.

---

## :dart: Final synthesis challenge: Design a hexagon under pressure

### Scenario
You are building **Notification Service**.

Requirements:
- Inbound: HTTP API `POST /notify`, plus Kafka `UserSignedUp` events
- Outbound: Email provider (sometimes down), SMS provider (rate-limited), Postgres for audit log
- Must be idempotent (events can be duplicated)
- Must support “dry-run” mode for testing campaigns
- Needs observability and dead-lettering for poison messages

### Your task (pause & think)
1) List **3 inbound ports** (use cases).
2) List **4 outbound ports** (dependencies).
3) Decide which concerns belong in adapters vs use cases:
   - retries
   - rate limiting
   - idempotency
   - message schema evolution
   - mapping errors to HTTP

Write your answer as if you were explaining to a teammate.

---

### Progressive reveal: one possible solution

<details>
<summary>Reveal a reference design</summary>

#### Inbound ports
- `SendNotificationUseCase` (for HTTP)
- `HandleUserSignedUpUseCase` (for Kafka)
- `PreviewNotificationUseCase` (dry-run)

#### Outbound ports
- `EmailSenderPort`
- `SmsSenderPort`
- `AuditLogPort`
- `IdempotencyStorePort`

#### Concern placement
- Retries: outbound adapters (email/sms) with policy knobs from application
- Rate limiting: outbound adapter (SMS client) + application policy (when to fallback)
- Idempotency: application/use case using `IdempotencyStorePort`
- Schema evolution: inbound Kafka adapter (versioned deserialization)
- HTTP error mapping: inbound HTTP adapter

#### Failure strategy
- Non-retryable errors -> dead-letter topic with reason
- Retryable errors -> exponential backoff + bounded retries, then DLQ
- Provider outage -> fallback (email->sms) if business allows

</details>

> **Key insight box:**
> A good hexagon design reads like an operational plan: clear contracts, clear failure handling, replaceable dependencies.

---

## :wave: Closing: What to do next in your codebase

### Challenge questions
1) Identify one place where domain imports an infrastructure dependency. How would you extract a port?
2) Pick one flaky outbound call. Can you move retry/circuit breaker logic into an adapter?
3) Add a second inbound adapter (CLI or consumer) that reuses an existing use case. What breaks?

### Practical next step
Start with one use case (e.g., `PlaceOrder`).
- Define an inbound port
- Define outbound ports for the dependencies it truly needs
- Move translation and resilience into adapters
- Add tests that simulate timeouts and duplicates

You’ll feel the architecture “click” when:
- domain tests run without Docker
- you can add a new adapter without touching domain logic
- failure handling becomes explicit and consistent
