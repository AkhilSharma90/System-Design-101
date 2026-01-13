---
slug: optimistic-vs-pessimistic-locking
title: Optimistic vs Pessimistic Locking
readTime: 20 min
orderIndex: 3
premium: false
---




# Optimistic vs Pessimistic Locking

> You’re about to design concurrency control for a distributed system that handles money, inventory, bookings, or collaborative edits. You have two archetypal strategies:
>
> - **Pessimistic locking**: “Assume collisions will happen - reserve access up front.”
> - **Optimistic locking**: “Assume collisions are rare - detect conflicts at commit time.”
>
> This article turns those into interactive decision games, failure drills, and mental models you can reuse when you’re choosing between **row locks, compare-and-swap, leases, version checks, and distributed transactions**.

---

## Network + system assumptions (read once)
Distributed locking discussions get misleading unless we state assumptions:

- The network can **drop, delay, duplicate, and reorder** messages.
- Clocks are not perfectly synchronized; timeouts are **heuristics**, not truth.
- Processes can pause (GC, CPU steal), crash, restart, and lose local state.
- Partitions happen; during partitions you must choose between **availability** and **single-owner safety** (CAP implications).

---

##  Challenge 1: The last croissant problem (why locking exists)

- Scenario or challenge
  You run a busy coffee shop. There is **one last croissant** in the display case. Two baristas take orders at the same time:
  - Barista A sells it to Customer A.
  - Barista B sells it to Customer B.

- Interactive question (pause and think)
  If you do nothing special, what are the possible outcomes?
  1) One customer gets it, the other is refunded - no harm.
  2) Both customers are charged but only one croissant exists.
  3) The system “magically” prevents the double-sell.

  Pause for 10 seconds.

- Explanation with analogy (progressive reveal)
  The realistic outcomes are **(1)** and **(2)**. Without concurrency control, you can **oversell**.

- Real-world parallel
  Replace “croissant” with:
  - One seat on a flight
  - One inventory unit in a warehouse
  - One bank account balance
  - One rate-limit token

  In distributed systems, we need a way to ensure that **concurrent operations** don’t violate invariants.

-
  **Key insight:** Locking is about protecting invariants under concurrency. The “right” strategy depends on contention, latency, failure modes, and what you can tolerate: blocking, retries, or occasional aborts.

- Challenge question
  If customers mostly order coffee (no contention) and croissants rarely run out (rare contention), which style *sounds* better: optimistic or pessimistic?

---

##  Two ways to avoid double-selling

- Scenario or challenge
  You need a mental picture that generalizes from “croissant” to “distributed state.”

- Interactive question (pause and think)
  Which is more like your workload: a crowded checkout line (frequent collisions) or a mostly empty store (rare collisions)?

- Explanation with analogy
  Two mental pictures:

  1) **Pessimistic locking** (reserve first)
     - “Hold the croissant behind the counter while you ring it up.”
     - Others must wait.

  2) **Optimistic locking** (work first, validate later)
     - “Ring it up assuming it’s still there; at the end, check the case. If it’s gone, cancel and apologize.”
     - Others don’t wait, but some transactions may fail and must retry.

  [IMAGE: Side-by-side diagram. Left: Pessimistic path with lock acquisition -> critical section -> release. Right: Optimistic path with read version -> do work -> compare version on commit; if mismatch, abort/retry.]

- Real-world parallel
  - Pessimistic: restaurant host puts a “reserved” sign on a table.
  - Optimistic: two groups walk toward the same table; whoever sits first wins, the other relocates.

-
  **Key insight:** Pessimistic trades **waiting** for **certainty**. Optimistic trades **retries/aborts** for **parallelism**.

- Challenge question
  In a geo-distributed system where round trips are expensive, which cost hurts more: waiting for locks or retrying on conflicts?

---

##  Challenge 2: A distributed cart checkout with two services

- Scenario or challenge
  A user checks out a cart:
  - **Inventory service** decrements stock.
  - **Payment service** charges the card.

  You need the invariant: **don’t charge if you can’t reserve inventory**.

- Interactive question (pause and think)
  Where should locking happen?
  - A) In the database row for inventory.
  - B) In a distributed lock service (e.g., ZooKeeper/etcd/Consul).
  - C) In the application via optimistic version checks.
  - D) “No locks - just eventual consistency.”

  Pause for 15 seconds.

- Explanation with analogy (progressive reveal)
  It depends on architecture:
  - If inventory is stored in a single strongly consistent database shard: **A** can work (row lock / `SELECT ... FOR UPDATE`).
  - If inventory spans multiple nodes or stores: you may need **B** (distributed lock / lease) or **C** (optimistic concurrency with versioning) plus compensation.
  - **D** can work only if your business invariant is weaker (e.g., allow oversell/backorder).

  **Production insight:** even if you “lock inventory”, you still need **idempotency** for payment and order creation because timeouts create ambiguous outcomes.

- Real-world parallel
  This is like a delivery service that must coordinate between:
  - Warehouse picking (inventory)
  - Payment processing

  If the warehouse can’t pick, charging is a bad customer experience.

-
  **Key insight:** In distributed systems, “locking” is not just a DB feature - it can be a **protocol** spanning services.

- Challenge question
  If the payment provider is external and slow, do you want to hold an inventory lock while waiting for payment?

---

##  “Optimistic locking means no locking”

- Scenario or challenge
  Teams hear “optimistic” and assume “no synchronization.”

- Interactive question (pause and think)
  If no one ever blocks, how can the system prevent two writers from committing incompatible updates?

- Explanation with analogy
  Optimistic locking is usually:
  - **Conflict detection** (version check, compare-and-swap)
  - Implemented with **atomic commit-time primitives** (CAS / conditional update)

  Example: `UPDATE ... WHERE version = ?` is “optimistic,” but the database still uses internal latching/locking to apply the update atomically.

- Real-world parallel
  Two people can both fill out the last reservation form, but the host accepts only the one with the latest ticket/sequence.

-
  **Key insight:** Optimistic locking avoids **long-held locks**, not **all synchronization**.

- Challenge question
  What is the “lock” in optimistic locking? (Hint: it’s the **commit-time atomicity**.)

---

##  Pessimistic locking in distributed systems

- Scenario or challenge
  Reserving the last table at a restaurant: the host says, “I’ll put your name on it and hold the table.” Others must wait.

- Interactive question (pause and think)
  In distributed locking, which component must be strongly consistent?
  - A) Clients
  - B) Lock service / consensus group
  - C) Cache layer

- Explanation with analogy (progressive reveal)
  **Answer: B.** If the lock service isn’t strongly consistent, you can get **split-brain locks** (two owners).

  [IMAGE: Sequence diagram of distributed lock acquisition using etcd: client -> etcd put lock key with lease -> success; other client blocks/watches key; release or lease expiry.]

- Real-world parallel
  A restaurant’s reservation book must be authoritative. If two hosts keep separate books that don’t sync reliably, you’ll double-book tables.

-
  **Key insight:** Distributed pessimistic locking requires a strongly consistent coordinator (or an equivalent single-writer mechanism).

- Challenge question
  If your lock coordinator is a Raft cluster, what happens during a network partition - can clients still acquire locks?

  **Clarification (CAP):** a correctly implemented Raft-based lock service will typically **refuse** lock acquisition if it cannot reach a quorum. That preserves **consistency** (no split-brain) at the cost of **availability**.

---

##  Failure drill: The lock holder crashes

- Scenario or challenge
  A worker acquires a lock on `inventory:sku123`, then crashes mid-operation.

- Interactive question (pause and think)
  How does the system avoid deadlock (lock never released)?
  - A) Locks live forever; humans fix it.
  - B) Use leases / TTLs.
  - C) Rely on TCP disconnect.
  - D) Store locks in Redis without replication.

- Explanation with analogy (progressive reveal)
  **Answer: B.** The standard answer is **leases**.
  - The lock has an expiry time.
  - The holder must renew.
  - If it dies, the lock eventually expires.

  TCP disconnect can help but isn’t sufficient in partitions: TCP may stay open or fail late.

  **Production insight:** leases solve “stuck locks” but introduce “lock loss” (a paused process can lose its lease). To be safe you often need **fencing tokens** (covered later).

- Real-world parallel
  A restaurant holds your table for **10 minutes**. If you don’t show up, they give it away.

-
  **Key insight:** Pessimistic locks in distributed systems must be time-bounded to tolerate crashes and partitions.

- Challenge question
  What’s the trade-off of short leases vs long leases?

---

##  Which statement is true? (Pessimistic edition)

- Scenario or challenge
  You must decide whether to block or to abort.

- Interactive question (pause and think)
  Pick the true statement(s):

  1) “Pessimistic locking guarantees progress.”
  2) “Pessimistic locking reduces wasted work under high contention.”
  3) “Pessimistic locking is always safer in distributed systems.”
  4) “Pessimistic locking can amplify tail latency.”

  Pause.

- Explanation with analogy (progressive reveal)
  Answers:
  - **(2) True**: high contention -> fewer retries, more waiting.
  - **(4) True**: queued lock waiters + coordinator latency -> long tails.
  - **(1) False**: you can deadlock, starve, or stall during partitions.
  - **(3) False**: it can be safer for invariants, but safety depends on correct lock implementation and failure handling.

- Real-world parallel
  A single host controlling a waitlist reduces double-booking but can create a long line.

-
  **Key insight:** Pessimistic locking often trades throughput for predictable correctness - but not necessarily predictable latency.

- Challenge question
  In a system with strict SLOs (p99 latency), when might you prefer abort/retry over waiting?

---

##  Optimistic locking in distributed systems

- Scenario or challenge
  Two people editing the same Google Doc line: you type a sentence, someone else edits the same line. The system merges or asks you to resolve.

- Interactive question (pause and think)
  Which of these is a classic optimistic locking pattern?
  - A) `SELECT ... FOR UPDATE`
  - B) `UPDATE item SET qty=qty-1 WHERE id=? AND version=?`
  - C) `SETNX lockkey`

- Explanation with analogy (progressive reveal)
  **Answer: B.** It updates only if the version matches.

  [IMAGE: Timeline diagram showing two clients read version v=7; client A commits to v=8; client B tries commit with v=7 fails and must retry.]

- Real-world parallel
  Two customers fill out a form; the system accepts the first one that reaches the clerk and rejects the stale one.

-
  **Key insight:** Optimistic locking is “validate on write.” Pessimistic locking is “block before write.”

- Challenge question
  If conflicts are frequent, what happens to throughput under optimistic locking?

---

##  “Optimistic locking is only for databases”

- Scenario or challenge
  You see it in JPA or SQL and assume it’s a DB-only technique.

- Interactive question (pause and think)
  Where else have you seen: “If version matches, apply; else reject”?

- Explanation with analogy
  Optimistic concurrency is everywhere:
  - HTTP caching with **ETags** (`If-Match`)
  - Object stores with **conditional writes** (e.g., “only put if generation matches”)
  - Coordinating config changes with **revision numbers**
  - CRDT/OT systems (a more advanced form: merge instead of abort)

- Real-world parallel
  A delivery app updates your address only if you are editing the latest version of the order.

-
  **Key insight:** Optimistic concurrency is a protocol pattern, not a DB feature.

- Challenge question
  Where in your stack could you add version checks to prevent lost updates?

---

##  Failure drill: Retry storms and thundering herds

- Scenario or challenge
  A popular SKU goes viral. Thousands of clients attempt to decrement stock using optimistic locking.

- Interactive question (pause and think)
  What failure mode can occur?
  - A) Deadlock
  - B) Retry storm causing overload
  - C) Perfect scaling

- Explanation with analogy (progressive reveal)
  **Answer: B.** Under high contention, optimistic locking can cause:
  - Many failed updates
  - Hot partitions / hot keys
  - Cascading retries
  - Amplified load on the primary/leader

  **Production mitigations:**
  - Exponential backoff + jitter; cap retries.
  - Server-side rate limiting / admission control.
  - Queueing (serialize at the hotspot) or sharding the counter.
  - Redesign: tokens/reservations (see flash sale exercise).

- Real-world parallel
  It’s like telling everyone, “Just walk to the counter and if the croissant is gone, go back and try again.” The crowd blocks the entrance.

-
  **Key insight:** Optimistic locking shifts contention from waiting to wasted work.

- Challenge question
  Which is easier to control: waiting queues (pessimistic) or retry loops (optimistic)?

---

##  Comparison table: Optimistic vs Pessimistic (distributed focus)

| Dimension | Pessimistic locking | Optimistic locking |
|---|---|---|
| Assumption | Conflicts likely | Conflicts rare |
| Coordination | Up front (acquire lock) | At commit (validate version/CAS) |
| Latency profile | Can block; tail latency from lock waits | Fast on success; spikes under conflict due to retries |
| Failure handling | Needs leases/heartbeats; risk of lock loss | Needs retry/backoff; risk of livelock under contention |
| Partition behavior | Often stops granting locks to preserve safety (C over A) | Reads can continue; writes depend on store’s consistency + leader/quorum |
| Implementation | Lock manager/leader/DB locks | Version fields, CAS, MVCC, conditional writes |
| Good for | High contention, strong invariants, short critical sections | Read-heavy, low contention, high latency networks |
| Bad for | Long operations, external calls while holding lock | Hot keys, bursty contention, expensive retries |

- Challenge question
  Which dimension matters most for your system: tail latency, throughput, or simplicity?

---

##  Challenge 3: Don’t hold locks across the payment gateway

- Scenario or challenge
  Your checkout flow:
  1) Lock inventory row
  2) Call external payment provider (takes 2-6 seconds)
  3) Update inventory and release lock

- Interactive question (pause and think)
  What’s the worst thing about this design?
  - A) It’s slow.
  - B) It’s expensive.
  - C) It turns your inventory into a global bottleneck.
  - D) All of the above.

- Explanation with analogy (progressive reveal)
  **Answer: D.** Holding locks across slow, failure-prone external calls is a classic distributed systems foot-gun.

  **Production pattern:** split into two phases:
  - **Reserve inventory** quickly (short DB transaction / short lease).
  - **Charge payment** outside the lock.
  - **Confirm** reservation or **release/expire** it.

  This is a saga/reservation workflow, not a single lock.

- Real-world parallel
  It’s like a host holding the only table while you step outside to call your friend to decide what to order.

-
  **Key insight:** In distributed systems, time is the enemy of locks.

- Challenge question
  If you must guarantee “no charge without inventory,” what compensation or idempotency mechanisms do you need?

---

##  Implementation patterns (with code markers)

### Pattern A: Pessimistic locking with SQL row locks (and what the demo is / isn’t)

- Scenario or challenge
  You have a single strongly consistent database for inventory.

- Interactive question (pause and think)
  What happens to other writers while you hold a row lock?

- Explanation with analogy
  Others wait in line; you are holding the “croissant behind the counter.”

- Real-world parallel
  A single clerk handles one customer at a time for a specific ticket.

-
  **Key insight:** Keep pessimistic critical sections short; do not call external systems while holding locks.

> **Important clarification:** the following TCP lock server is a *teaching demo* to illustrate blocking waiters. It is **not** a correct distributed lock implementation (no persistence, no fencing, no quorum, no crash recovery).

```python
# Demonstration-only: blocking mutex semantics similar to a DB row lock.
# Not production-safe.
import socket, threading

HOST, PORT = "127.0.0.1", 5055
locks, cv = set(), threading.Condition()

def handle(conn: socket.socket):
    key = None
    try:
        key = conn.recv(128).decode("utf-8").strip()  # client sends resource key
        if not key:
            conn.sendall(b"ERR empty-key\n"); return
        with cv:
            while key in locks:  # blocks like SELECT ... FOR UPDATE
                cv.wait(timeout=5)
            locks.add(key)
        conn.sendall(b"OK locked\n")
        conn.recv(1)  # wait for client to signal commit/rollback
    except (OSError, UnicodeDecodeError):
        pass
    finally:
        with cv:
            if key in locks:
                locks.remove(key)
                cv.notify_all()
        try:
            conn.close()
        except OSError:
            pass

# Usage example: run server; clients connect, send "inventory:sku123\n", do work, then send 1 byte to release.
```

- Challenge question
  What happens if your app server dies mid-transaction? (Hint: DB will roll back and release locks - but what about side effects outside DB?)

---

### Pattern B: Optimistic locking with version columns

- Scenario or challenge
  You want high read concurrency and can tolerate retries.

- Interactive question (pause and think)
  If two writers race, how does the loser learn it lost?

- Explanation with analogy
  The loser’s commit sees “someone updated this since you read it,” like arriving at the counter with a stale ticket.

- Real-world parallel
  A reservation form is accepted only if the table is still free.

-
  **Key insight:** Optimistic locking needs a retry policy (backoff, jitter, max attempts) and good UX for conflict errors.

```javascript
// Demonstration-only: optimistic concurrency with a version field + retry/backoff.
// In production, the atomic conditional write must be provided by the datastore.
const store = new Map([['sku123', { qty: 1, version: 0 }]]);

async function decrementWithOptimism(id, n, maxAttempts = 5) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const cur = store.get(id);
    if (!cur) throw new Error('not_found');
    if (cur.qty < n) throw new Error('insufficient_stock');

    const expected = cur.version;
    const next = { qty: cur.qty - n, version: expected + 1 };

    // Simulated conditional write: only commit if version unchanged.
    if (store.get(id)?.version === expected) {
      store.set(id, next);
      return next;
    }

    const backoffMs = Math.min(200, 20 * attempt) + Math.floor(Math.random() * 20);
    await new Promise(r => setTimeout(r, backoffMs));
  }
  throw new Error('conflict_retry_exhausted');
}

// Usage example: await decrementWithOptimism('sku123', 1)
```

- Challenge question
  How many retries is too many before you fail fast and surface a “please retry” to the user?

---

### Pattern C: Conditional writes in a key-value store (CAS)

- Scenario or challenge
  You store state in a KV store with compare-and-swap support.

- Interactive question (pause and think)
  Why is “read then write” dangerous without a compare primitive?

- Explanation with analogy
  Without compare, you can overwrite someone else’s update (“lost update”). Compare makes it a single atomic decision.

- Real-world parallel
  The host checks the reservation book at the moment of writing your name, not 2 minutes earlier.

-
  **Key insight:** If your store supports transactional compare (e.g., etcd Txn), you can avoid extra coordination round trips.

> **Important clarification:** the following single-threaded server demonstrates the *API shape* of CAS. Real CAS correctness in distributed systems requires a strongly consistent store (single leader/quorum) or a linearizable primitive.

```python
# Demonstration-only: CAS semantics via a single-writer event loop.
# Not replicated; not durable.
import socket, selectors

sel, kv = selectors.DefaultSelector(), {"cfg": ("v1", 1)}  # key -> (value, rev)

def serve(host="127.0.0.1", port=6060):
    ls = socket.socket()
    ls.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    ls.bind((host, port))
    ls.listen()
    ls.setblocking(False)
    sel.register(ls, selectors.EVENT_READ)

    while True:
        for key, _ in sel.select(timeout=1):
            if key.fileobj is ls:
                c, _ = ls.accept()
                c.setblocking(False)
                sel.register(c, selectors.EVENT_READ)
            else:
                c = key.fileobj
                try:
                    line = c.recv(256).decode().strip()
                    op, k, exp, newv = line.split(" ", 3)
                    val, rev = kv.get(k, (None, 0))
                    if op != "CAS":
                        raise ValueError("bad_op")
                    if str(rev) != exp:
                        c.sendall(f"CONFLICT {rev}\n".encode())
                        continue
                    kv[k] = (newv, rev + 1)
                    c.sendall(f"OK {rev+1}\n".encode())
                except Exception:
                    try:
                        c.sendall(b"ERR\n")
                    except OSError:
                        pass
                    try:
                        sel.unregister(c)
                    except Exception:
                        pass
                    c.close()

# Usage example: client sends "CAS cfg 1 v2\n"; server replies OK/CONFLICT.
```

- Challenge question
  If your store is only eventually consistent, does CAS still work across replicas?

  **Answer:** not in the linearizable sense. You might get “successful” CAS on two replicas during a partition (split brain). For correctness you need a **single-writer leader** or **quorum/consensus** providing linearizable compare-and-set.

---

### Pattern D: Distributed pessimistic locks with leases (and fencing tokens)

- Scenario or challenge
  You must coordinate access to a shared resource not protected by a single DB (e.g., singleton job, migration, cross-store invariant).

- Interactive question (pause and think)
  What’s the minimum requirement for correctness if locks can be lost?

- Explanation with analogy
  You need a way to prevent a previous holder from continuing after losing ownership.

- Real-world parallel
  A reservation ticket expires; you can’t show up later and claim the table.

-
  **Key insight:** Leases prevent deadlocks; fencing tokens prevent stale owners from corrupting state.

```python
# Demonstration-only: lease-based lock with fencing tokens (in-memory).
# In production, the lock service must be strongly consistent and tokens must be enforced by the protected resource.
import threading, time

class LeaseLock:
    def __init__(self):
        self._mu = threading.Lock()
        self._owner = None
        self._exp = 0.0
        self._token = 0

    def acquire(self, owner: str, ttl_s: float = 5.0):
        now = time.monotonic()
        with self._mu:
            if self._owner and now < self._exp:  # someone holds a valid lease
                raise TimeoutError("lock_busy")
            self._token += 1
            self._owner = owner
            self._exp = now + ttl_s
            return self._token, self._exp

    def renew(self, owner: str, ttl_s: float = 5.0):
        now = time.monotonic()
        with self._mu:
            if self._owner != owner or now >= self._exp:
                raise PermissionError("lost_lock")
            self._exp = now + ttl_s
            return self._exp

# Usage example: token, exp = lock.acquire("worker-1"); periodically lock.renew("worker-1"); stop work on lost_lock.
```

- Challenge question
  What should your worker do if it loses the lock mid-operation?

  **Production answer:** stop making side effects immediately; if possible, roll back/compensate; and ensure downstream writes are protected by **fencing token checks** so stale workers are rejected.

---

##  Progressive reveal: The consistency triangle behind your choice

- Scenario or challenge
  You want both perfect correctness and perfect availability.

- Interactive question (pause and think)
  Why is distributed locking hard?

- Explanation with analogy (progressive reveal)
  Because you’re negotiating between:
  - Consistency: only one writer/owner
  - Availability: system keeps working during failures
  - Partition tolerance: network splits happen

  In partitions, you must choose:
  - Stop granting locks (preserve safety) -> lower availability
  - Grant locks anyway (preserve availability) -> risk split-brain and invariant violation

  [IMAGE: CAP-style diagram annotated with “lock service chooses C over A during partitions” and “optimistic validation depends on store consistency model”.]

- Real-world parallel
  Two restaurant hosts on opposite sides of a building with no communication: do they keep seating (availability) or stop seating to avoid double-booking (consistency)?

-
  **Key insight:** Locking is coordination, and coordination is where distributed systems pay latency and availability costs.

- Challenge question
  If your product can tolerate occasional oversell but not downtime, how does that change your locking choice?

---

##  Which statement is true? (Optimistic edition)

- Scenario or challenge
  You’re considering “no waiting, just retry.”

- Interactive question (pause and think)
  Pick the true statement(s):
  1) “Optimistic locking always increases throughput.”
  2) “Optimistic locking requires a way to detect write-write conflicts.”
  3) “Optimistic locking works best when conflicts are rare and retries are cheap.”
  4) “Optimistic locking is incompatible with geo-replication.”

- Explanation with analogy (progressive reveal)
  Answers:
  - **(2) True**: you need versions/CAS/conditional writes.
  - **(3) True**: that’s the core assumption.
  - **(1) False**: under contention, throughput can collapse.
  - **(4) False**: it can work with geo-replication, but depends on whether you have a single-writer leader, synchronous replication, or conflict resolution.

  **Clarification:** for invariants like “counter must not go below zero”, multi-leader geo writes are hard without coordination; many systems choose **single-writer per key/shard**.

- Real-world parallel
  Letting everyone walk up to the counter works if the store is empty; it fails if it’s a stampede.

-
  **Key insight:** Optimistic locking is a bet: “conflicts are rare enough that retries are cheaper than waiting.”

- Challenge question
  What makes retries “cheap” in your system: fast storage, small payloads, or idempotent operations?

---

##  Matching exercise: Choose the right tool

- Scenario or challenge
  You must map workload shape to coordination strategy.

- Interactive question (pause and think)
  Match the scenario (left) to the best default approach (right). Pause before the reveal.

  Scenarios:
  A) Updating a user profile (name, avatar) in a read-heavy app
  B) Reserving seats for a concert that’s about to sell out
  C) Running a once-per-day billing job (must not run twice)
  D) Collaborative editing with frequent concurrent edits
  E) Decrementing a rate-limit counter per API call

  Approaches:
  1) Pessimistic lock (row lock / distributed mutex)
  2) Optimistic lock (version check)
  3) Lease-based leader/singleton
  4) Merge-based concurrency (OT/CRDT)
  5) Token bucket / sharded counters / approximate techniques

- Explanation with analogy (progressive reveal)
  One reasonable mapping:
  A->2, B->1, C->3, D->4, E->5

- Real-world parallel
  - Profile updates: most people don’t edit the same profile simultaneously.
  - Concert seats: everyone collides.
  - Billing job: you need a single “manager on duty.”
  - Collaborative editing: you need merge semantics.
  - Rate limits: approximate/sharded is often acceptable.

-
  **Key insight:** The “best” approach depends on whether you can merge, retry, wait, or approximate.

- Challenge question
  Which of your system’s invariants can be approximated (rate limits) and which cannot (money)?

---

##  Trade-offs under failure: partitions, timeouts, and ambiguity

- Scenario or challenge
  You attempt an update. The database commits, but the client times out and retries.

- Interactive question (pause and think)
  What do you need regardless of optimistic vs pessimistic locking?
  - A) Idempotency keys
  - B) Bigger timeouts
  - C) A faster network

- Explanation with analogy (progressive reveal)
  **Answer: A.** You need idempotency to handle ambiguous outcomes.

  **Production insight:** idempotency must be enforced at the *side-effect boundary* (e.g., payment charge, order creation). If you only dedupe at the edge but not at the DB/payment provider, you can still double-charge.

- Real-world parallel
  You place a delivery order; your phone loses signal after you hit “Pay.” You try again. Without an idempotency key, you might pay twice.

-
  **Key insight:** Locking controls concurrency, but idempotency controls retries under uncertainty.

- Challenge question
  Where should idempotency be enforced: API gateway, service layer, or database?

---

##  “Distributed locks make everything consistent”

- Scenario or challenge
  You add a lock and expect multi-service atomicity.

- Interactive question (pause and think)
  If Service A uses the lock but Service B doesn’t, is the invariant protected?

- Explanation with analogy
  A lock only protects what all participants agree to protect.
  - If Service A uses the lock but Service B doesn’t, invariants still break.
  - If the lock is lost (lease expiry) and the client continues, you can get overlap.
  - If you need atomicity across services/stores, you’re in distributed transaction territory (2PC) or sagas.

- Real-world parallel
  One host uses the reservation book; the other seats walk-ins without checking it.

-
  **Key insight:** Locks are coordination agreements with failure modes.

- Challenge question
  What’s the scope of your lock - one row, one shard, one service, or the whole business invariant?

---

##  Leases, fencing tokens, and “lock loss” correctness

- Scenario or challenge
  Worker W1 acquires a lease-based lock and starts processing. A network hiccup delays renewals; the lease expires. Worker W2 acquires the lock and starts processing too. W1 comes back and continues, unaware it lost the lock.

- Interactive question (pause and think)
  How do you prevent W1 from corrupting state after losing the lock?

- Explanation with analogy (progressive reveal)
  Use fencing tokens:
  - Each lock acquisition returns a monotonically increasing token `t`.
  - Any write to the protected resource must include `t`.
  - The resource rejects writes with tokens lower than the latest seen.

  [IMAGE: Diagram showing W1 token=41, lease expires; W2 token=42; W1 tries write with 41 and is rejected by resource that tracks max token.]

- Real-world parallel
  A restaurant gives numbered tickets. Only the highest ticket number is allowed to claim the reservation.

-
  **Key insight:** Leases prevent deadlocks, but fencing tokens prevent stale owners from causing damage.

- Challenge question
  Where do you store and enforce the “max token seen” - in the database row, the service, or a proxy?

  **Production answer:** enforce it at the *resource that is being protected* (often the DB row or a single-writer service). If enforcement is only in the lock client, it’s not safe.

---

##  Challenge 4: Inventory at scale - one SKU, many regions

- Scenario or challenge
  You run an e-commerce platform with multiple regions. Inventory for a hot SKU is shared globally.

- Interactive question (pause and think)
  Which approach is most realistic?
  - A) Global distributed lock per SKU
  - B) Single-writer leader per SKU (or per shard)
  - C) Multi-leader updates with conflict resolution
  - D) Allow oversell and reconcile

- Explanation with analogy (progressive reveal)
  Often **B** or **D**:
  - Global lock (A) is usually too slow and fragile at scale.
  - Single-writer per shard (B) is common: route all writes for a SKU to one leader.
  - Multi-leader with conflict resolution (C) is hard for decrementing counters because merges are non-trivial if you must never go below zero.
  - Oversell + reconcile (D) is a business choice.

  **Production insight:** another common approach is **regional reservations** (allocate each region a quota, periodically rebalance). This reduces cross-region coordination.

- Real-world parallel
  One store manager controls the last-item decisions; other branches forward requests.

-
  **Key insight:** For counters and decrements, you often end up with single-writer or reservation tokens rather than pure optimistic multi-writer.

- Challenge question
  If you must support writes in every region with low latency, what invariant would you relax to avoid global coordination?

---

##  Hybrid strategies (because reality is messy)

- Scenario or challenge
  Pure optimistic vs pure pessimistic is rarely the whole story.

- Interactive question (pause and think)
  Where can you reduce coordination scope: by key, by shard, by time, or by semantics?

- Explanation with analogy
  Hybrid strategies:
  1) Optimistic reads + pessimistic writes
     - Read without locks
     - Lock only at the moment of commit for a short time

  2) Reservation systems (soft locks)
     - Create a reservation record with TTL
     - Confirm later

  3) Escalation
     - Start optimistic; if conflicts exceed threshold, switch to pessimistic temporarily

  4) Partitioned ownership
     - Assign ownership (leader) per key range
     - Within owner, use local locks or optimistic versioning

  [IMAGE: Architecture diagram: request router -> shard leader -> local DB; shows ownership boundaries reducing coordination scope.]

- Real-world parallel
  A restaurant uses walk-in seating (optimistic) on quiet days and a reservation list (pessimistic) on busy nights.

-
  **Key insight:** The best systems minimize the scope and duration of coordination.

- Challenge question
  What’s the smallest “unit” you can lock - order, SKU, user, or shard?

---

##  Exercise: Design a locking strategy for a flash sale

- Scenario or challenge
  You have:
  - 10,000 units of an item
  - 1,000,000 users refreshing at once
  - Requirement: do not sell more than 10,000
  - SLO: p99 < 300ms

- Interactive question (pause and think)
  Pick a strategy:
  1) Pessimistic lock on a single inventory row
  2) Optimistic version check on a single inventory row
  3) Pre-mint 10,000 tokens (claim tokens) + idempotent checkout
  4) Allow oversell and cancel later

  Write down your choice and why.

- Explanation with analogy (progressive reveal)
  A typical industry answer is **(3) Pre-mint tokens**:
  - You turn “decrement a hot counter” into “claim a unique token.”
  - Token claim can be done with partitioning, queues, or atomic pop.
  - Checkout becomes idempotent and can be processed asynchronously.

  But:
  - It adds complexity.
  - You need token expiry and reconciliation.

  **Failure scenarios to plan for:**
  - Token claimed but payment fails -> token must be released/expired.
  - Client times out after token claim -> idempotency key must return the same token/decision.
  - Token service partition -> decide whether to stop sales (safety) or risk oversell (availability).

- Real-world parallel
  Hand out 10,000 numbered wristbands at the door; only wristband holders can buy.

-
  **Key insight:** When contention is extreme, neither naive pessimistic nor naive optimistic locking scales well. You redesign the invariant.

- Challenge question
  What’s the failure mode if token expiry is too short? Too long?

---

##  Observability: How to tell you chose wrong

- Scenario or challenge
  You deployed your locking strategy. Now production traffic arrives.

- Interactive question (pause and think)
  What would you alert on first: lock wait time or conflict rate?

- Explanation with analogy
  Signals you should be measuring:

  If you chose pessimistic locking:
  - Lock wait time distribution (p50/p95/p99)
  - Deadlock counts (DB) / lock acquisition timeouts
  - Lease renewal failures
  - Coordinator (Raft) latency and leader changes

  If you chose optimistic locking:
  - Conflict rate (% of failed conditional writes)
  - Retry counts per request
  - Hot key metrics (skew)
  - CPU wasted on aborted work

  [IMAGE: Dashboard mockup showing lock wait histogram vs optimistic conflict rate chart.]

- Real-world parallel
  - Pessimistic: watch how long people wait in line.
  - Optimistic: watch how many people get turned away and come back.

-
  **Key insight:** Your locking strategy is a control loop: measure contention, tune backoff/leases, and redesign hotspots.

- Challenge question
  What threshold indicates trouble sooner in your system: “conflict rate > X%” or “lock wait p99 > Y ms”? Why?

---

##  “Deadlocks only happen with pessimistic locking”

- Scenario or challenge
  You avoided locks and feel safe.

- Interactive question (pause and think)
  If everyone keeps retrying and no one makes progress, what is that called?

- Explanation with analogy
  - Classic deadlocks are associated with pessimistic locks.
  - Optimistic systems can have **livelock**: everyone retries and no one makes progress.

- Real-world parallel
  In a narrow hallway, two people keep stepping aside in the same direction repeatedly.

-
  **Key insight:** Pessimistic -> deadlock risk. Optimistic -> livelock/retry storm risk.

- Challenge question
  What’s your plan for livelock: backoff, jitter, queueing, or switching strategy?

---

##  Distributed transactions vs locking (where the boundary is)

- Scenario or challenge
  You want atomic updates across:
  - A SQL DB (orders)
  - A Kafka topic (events)
  - A Redis cache

- Interactive question (pause and think)
  Can a lock make this atomic?

- Explanation with analogy (progressive reveal)
  Not by itself.
  - A lock can serialize access, but atomicity across heterogeneous systems requires:
    - 2PC (rare in microservices due to complexity and availability cost), or
    - Transactional outbox + idempotent consumers, or
    - Sagas with compensations

  **Production insight:** if you need “DB update + event publish” reliably, the transactional outbox is usually the pragmatic choice.

- Real-world parallel
  Reserving a table doesn’t automatically reserve a taxi and charge your card; you still need coordination across parties.

-
  **Key insight:** Locking is about concurrency. Transactions are about atomicity. They overlap but aren’t the same.

- Challenge question
  If you use an outbox, where would optimistic vs pessimistic locking apply?

---

##  Quiz: Spot the bug (progressive reveal)

- Scenario or challenge
  You implement optimistic locking for account withdrawal:
  - Read balance and version
  - If balance >= amount, write new balance with version check

- Interactive question (pause and think)
  What’s missing for correctness in a distributed environment?

- Explanation with analogy (progressive reveal)
  Two common missing pieces:
  1) Idempotency: client retries can double-withdraw.
  2) Invariant scope: if there are multiple accounts or cross-account transfers, a single-row version check may not protect the full invariant.

```python
# Demonstration-only: optimistic withdrawal with idempotency + conditional update (in-memory).
# In production, idempotency must be persisted and the conditional update must be atomic at the datastore.
import threading

_accounts = {"acct1": {"bal": 100, "ver": 0}}
_seen = set()  # idempotency keys
_mu = threading.Lock()

def withdraw(acct: str, amount: int, idem_key: str):
    if amount <= 0:
        raise ValueError("amount_must_be_positive")
    with _mu:  # protects idempotency + update as one atomic decision (demo)
        if idem_key in _seen:
            return {"status": "ok", "idempotent": True}
        row = _accounts.get(acct)
        if not row:
            raise KeyError("account_not_found")
        if row["bal"] < amount:
            raise RuntimeError("insufficient_funds")
        expected = row["ver"]
        if _accounts[acct]["ver"] != expected:
            raise RuntimeError("write_conflict")
        _accounts[acct] = {"bal": row["bal"] - amount, "ver": expected + 1}
        _seen.add(idem_key)
        return {"status": "ok", "new_balance": _accounts[acct]["bal"], "version": expected + 1}

# Usage example: withdraw("acct1", 10, "req-123")
```

- Real-world parallel
  A customer calls twice because the line cut out; without a “same order” identifier, you charge twice.

-
  **Key insight:** Optimistic locking protects against concurrent writers, not duplicate requests or multi-entity invariants.

- Challenge question
  How would you implement transfers between two accounts without deadlocks or invariant violations?

  **Production answer (sketch):**
  - Prefer a single DB transaction that locks rows in a consistent order (pessimistic) *or* uses serializable isolation.
  - If accounts are sharded, you need a saga/2PC-like approach or redesign (e.g., ledger with append-only entries + async settlement).

---

##  Practical checklist: choosing optimistic vs pessimistic

- Scenario or challenge
  You must choose a default strategy and defend it in design review.

- Interactive question (pause and think)
  Answer these in order:
  1) Is the resource a hotspot (high contention)?
  2) Can operations be merged (CRDT) or compensated (saga)?
  3) How expensive is a retry (CPU, I/O, user experience)?
  4) How expensive is waiting (SLO impact, lock coordinator latency)?
  5) What failures matter most: crash, partition, timeout ambiguity?
  6) Can you enforce fencing tokens if using leases?

- Explanation with analogy
  You’re deciding whether to:
  - make people wait in line (pessimistic),
  - let them try and sometimes bounce (optimistic),
  - or change the process (tokens/merge).

-
  **Key insight:** Pick the strategy that fails in the way you can tolerate.

- Challenge question
  Which failure is worse for your product: occasional “please retry” errors or occasional oversells?

---

##  Final synthesis challenge: Design the “shared whiteboard” service

- Scenario or challenge
  You’re building a shared whiteboard:
  - Users can move shapes around
  - 100 users can edit the same board
  - Must feel real-time
  - Must not “teleport” shapes unpredictably
  - Offline edits should sync later

- Interactive question (pause and think)
  Design concurrency control for shape position updates.

  Choose one approach and justify:
  1) Pessimistic locking per shape
  2) Optimistic locking with versions and retries
  3) CRDT/OT merge-based approach
  4) Hybrid: optimistic for most updates, occasional locks for conflicts

  Also answer:
  - What’s your conflict resolution rule?
  - What do you do during partitions?
  - What metrics tell you it’s working?

- Explanation with analogy (progressive reveal)
  One strong solution outline:
  - Use merge-based semantics for positions (e.g., last-writer-wins with timestamps, or vector clocks if you need causality), possibly with constraints.
  - Use optimistic versioning for metadata that must not conflict (e.g., shape deletion).
  - Ensure idempotent operations and stable IDs.
  - Provide UI-level conflict hints rather than hard failures.

  **Production insight:** for “dragging”, you often want *intent-based* operations (deltas) and smoothing on the client, not strict locking.

- Real-world parallel
  A shared dry-erase board in a meeting room: people can write simultaneously, but you need conventions for overwriting and erasing.

-
  **Key insight:** The most scalable “locking” strategy is often: don’t lock - change the data type and semantics so concurrent updates can safely merge.

- Final challenge question
  What invariant in your system is truly non-negotiable - and how much coordination are you willing to pay to protect it?
