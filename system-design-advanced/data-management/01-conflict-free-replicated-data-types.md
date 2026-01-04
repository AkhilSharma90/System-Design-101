---
slug: conflict-free-replicated-data-types
title: Conflict Free Replicated Data Types
readTime: 20 min
orderIndex: 1
premium: false
---



# CRDTs (Conflict-free Replicated Data Types): Building Apps That Don’t Panic When the Network Does

> You’re designing a distributed system where users edit state concurrently—while offline, across regions, with retries, reordering, and partitions. You want availability and low latency, but you also want replicas to converge without a central lock or a consensus round-trip for every update.

---

## [CHALLENGE] The “Three Coffee Shops” Problem (Why CRDTs Exist)

- Scenario or challenge
- Interactive question
- Explanation with analogy
- Real-world parallel
- Key insight box

### Scenario
You run a chain of three coffee shops: North, Central, and South. Each shop has its own POS system and local database because you want the cashier to keep working even if the internet dies.

Customers can:
- add items to their order
- remove items
- apply loyalty points
- update their name on the order

The shops sync data whenever the network allows.

Now the internet becomes unreliable:
- North goes offline for 20 minutes.
- Central keeps syncing.
- South receives updates out of order.

Yet you still want the system to end up with the same final order state everywhere.

### Interactive question (pause and think)
If North and South both modify the same order while partitioned, how do you merge them later?

1) Pick “last write wins” by timestamp
2) Run a consensus protocol for every field update
3) Build data types whose merges are deterministic and converge automatically

Hold your answer.

### Explanation with analogy
Think of the order like a shared whiteboard in a busy cafe. If multiple baristas write on it at the same time, you don’t want to call a manager to approve every marker stroke. You want rules like:
- “If two baristas both add a drink, both additions should appear.”
- “If someone writes a new customer name, we need a predictable tie-breaker.”

CRDTs are those rules: data types designed so that replicas can diverge temporarily but converge automatically when they exchange updates.

### Real-world parallel
- Collaborative editing (notes, docs)
- Shopping carts and wishlists
- Presence and counters (likes, views)
- Distributed configuration with eventual consistency

### Key insight
> CRDTs let you choose availability and local writes under partition, while still guaranteeing convergence, without requiring coordination for every update.

### Challenge question
Which of the three options (1/2/3) best matches the CRDT approach? (Don’t answer yet; we’ll come back.)

---

## [AGREEMENT] What “Conflict-free” Actually Means (and What It Doesn’t)

- Scenario or challenge
- Interactive question
- Explanation with analogy
- Real-world parallel
- Key insight box

### Scenario
Two cashiers update the same order total while offline:
- North applies a 10% discount.
- South adds a new item.

When they reconnect, you need a single final state.

### Interactive question (pause and think)
When distributed systems folks say “conflict-free,” do they mean:

A) There are no conflicts in the business sense
B) There are no conflicts at the replication protocol level because merges are well-defined
C) The network never partitions

Pause and think.

### Explanation with analogy
“Conflict-free” means the system can merge concurrent updates deterministically.

But business semantics can still be tricky:
- If two managers set “today’s special price” concurrently, you still need a rule.
- CRDTs give you a merge rule; whether that rule matches your domain is your responsibility.

### Real-world parallel
In a restaurant, two servers might both “update the table status” at the same time. The restaurant can adopt a rule:
- “Occupied beats available”
- or “Last manager update wins”

That rule is a policy. CRDTs are mechanisms to apply such policies safely under concurrency.

### Key insight
> CRDTs eliminate replication-level conflicts by defining merges that are associative, commutative, and idempotent.

### Challenge question
Which answer (A/B/C) is correct? Keep it in mind; we’ll reveal soon.

---

## [WARNING] The Core CRDT Contract (The Math You Actually Need)

- Scenario or challenge
- Interactive question
- Explanation with analogy
- Real-world parallel
- Key insight box

### Scenario
Replicas exchange state or operations in arbitrary order:
- messages can be duplicated
- delayed
- reordered
- lost (but eventually retransmitted)

### Interactive question (pause and think)
If two replicas merge updates in different orders, what property must the merge function have so they still converge?

- Property 1: Associativity
- Property 2: Commutativity
- Property 3: Idempotence

Pick which ones are necessary.

### Mental model explanation
Mental model: “Merge is like pouring coffee”

Imagine each replica has a cup of coffee representing its local state.
- When replicas sync, they pour their coffee into each other.
- You might pour multiple times (duplicates).
- You might pour in different orders.

For the final taste to be the same everywhere:
- Commutative: pouring A into B tastes the same as pouring B into A.
- Associative: if you pour in steps, grouping doesn’t matter.
- Idempotent: pouring the same cup twice doesn’t change the result after the first time.

These three properties are the “replication doesn’t care about network weirdness” toolkit.

### The formal-ish view (only what you need)
Most state-based CRDTs use a join-semilattice:
- There’s a partial order <= (“this state contains at least as much information as that state”).
- Merge is the least upper bound (call it JOIN).

Convergence comes from:
- monotonic growth in that partial order
- merge = JOIN

### Key insight
> CRDT convergence is a math guarantee: if all replicas eventually receive each other’s updates, they converge regardless of message order/duplication.

### Challenge question
Which of the three properties (1/2/3) are required? (We’ll reveal shortly.)

---

## [GAME] Decision Game — Are You Sure You Want a CRDT?

- Scenario or challenge
- Interactive question
- Explanation with analogy
- Real-world parallel
- Key insight box

### Scenario
You’re building a “flash sale” inventory service. Stock must never go negative.

### Which statement is true? (pause and think)
1) “CRDTs guarantee invariants like stock >= 0 without coordination.”
2) “CRDTs guarantee convergence, but invariants may require coordination or special designs.”
3) “CRDTs are just eventual consistency; there are no guarantees.”

### Explanation
CRDTs give you convergence. They do not automatically give you application invariants.

Some invariants are compatible with CRDTs (e.g., monotonic ones like “likes only increase”).

Others require coordination or advanced techniques:
- escrow / bounded counters
- transactions
- consensus for certain operations

### Real-world parallel
A restaurant can let each branch accept reservations locally (high availability), but ensuring you never overbook across branches might require coordination unless you partition capacity (escrow).

### Key insight
> CRDTs solve the “merge” problem; they don’t magically solve all correctness constraints.

### Challenge question
Which statement (1/2/3) is correct?

---

## [INVESTIGATE] Two Big Families — State-based vs Operation-based CRDTs

- Scenario or challenge
- Interactive question
- Explanation with analogy
- Real-world parallel
- Key insight box

### Scenario
North and South are syncing. You need to decide what to send:
- the entire state (or a compact summary)
- or the operations performed

### Interactive question (pause and think)
If network messages can be duplicated and reordered, which approach is easier to make correct?

A) State-based (CvRDT): send state, merge via JOIN
B) Operation-based (CmRDT): send operations, apply in any order

### Explanation with analogy
State-based: “Send the whole inventory list”
Like a cafe manager sending the entire current menu board photo to another branch. The receiver compares and takes the union.

Pros:
- conceptually simple
- tolerant of message duplication/reordering

Cons:
- can be bandwidth-heavy
- may need anti-entropy protocols

Operation-based: “Send the changes you made”
Like texting: “Added oat milk” or “Removed espresso shot.”

Pros:
- smaller messages
- can be low-latency

Cons:
- requires reliable causal delivery or extra metadata to ensure commutativity

### Comparison table

| Aspect | State-based (CvRDT) | Operation-based (CmRDT) |
|---|---|---|
| What you send | Full/partial state | Operations (deltas) |
| Merge/apply | JOIN/merge function | Apply operations |
| Network assumptions | Very weak (dup/reorder OK) | Often needs causal delivery or op IDs |
| Typical cost | More bandwidth, simpler correctness | Less bandwidth, more protocol complexity |
| Example | G-Counter with map of replica counts | RGA/WOOT-like text CRDTs |

[IMAGE: side-by-side diagram showing two replicas; left uses periodic state exchange with merge, right uses streaming ops with causal metadata]

### Key insight
> State-based CRDTs push complexity into the data structure; op-based CRDTs push complexity into delivery guarantees and metadata.

### Challenge question
Which approach would you choose for a mobile offline-first notes app? Why?

---

## [PUZZLE] The Greatest Hits — Common CRDT Types (and When to Use Them)

### Scenario
Your product team asks for:
- likes counter
- shopping cart
- collaborative checklist
- user profile fields (name, avatar)

You want to pick CRDTs that match each.

### Matching exercise (pause and think)
Match each requirement to a CRDT type:

1) “Count likes; only increments”
2) “Count likes; increments and decrements”
3) “Set of items where adds and removes can be concurrent”
4) “Single value, last update wins (with tie-break)”

Options:
A) G-Counter
B) PN-Counter
C) OR-Set (Observed-Remove Set)
D) LWW-Register

Pause and think. We’ll reveal answers after explaining.

---

## [PIPELINE] G-Counter — The Friendly Counter That Only Goes Up

### Scenario
Each cafe branch tracks “number of coffees sold today.” You never decrement.

### Interactive question (pause and think)
If each replica keeps a local integer and you sum them, what happens when messages duplicate?

### Explanation with analogy
Mental model: “Each branch has its own tally sheet”

Instead of a single shared counter, each branch keeps its own tally.
- North increments its own cell.
- South increments its own cell.

Total = sum of all cells.

State is a map: replica_id -> count.
Merge is element-wise max.

Why max?
- because counts only increase
- max is idempotent and commutative

[CODE: Python, implement a state-based G-Counter with merge and increment, demonstrate convergence under reordering/duplication]

```python
# Implementing a state-based G-Counter (CvRDT)
from __future__ import annotations

class GCounter:
    def __init__(self, replica_id: str):
        if not replica_id:
            raise ValueError("replica_id must be non-empty")
        self.replica_id, self.state = replica_id, {replica_id: 0}

    def increment(self, n: int = 1) -> None:
        if n < 0:
            raise ValueError("G-Counter cannot decrement")
        self.state[self.replica_id] = self.state.get(self.replica_id, 0) + n

    def merge(self, other: "GCounter") -> None:
        if not isinstance(other, GCounter):
            raise TypeError("can only merge with GCounter")
        for rid, cnt in other.state.items():
            if cnt < 0:
                raise ValueError("invalid negative component")
            self.state[rid] = max(self.state.get(rid, 0), cnt)

    def value(self) -> int:
        return sum(self.state.values())

# Usage example (duplication/reordering safe)
a, b = GCounter("north"), GCounter("south")
a.increment(2); b.increment(1)
a.merge(b); a.merge(b)  # duplicate message
b.merge(a)              # reordering doesn't matter
assert a.value() == b.value() == 3
```

### Failure scenarios
- Duplicate messages: max ignores repeats.
- Reordering: merge is commutative.
- Partition: each replica continues locally; later merges converge.

### Trade-offs
- state grows with number of replicas (or you need compaction / membership management)

### Key insight
> G-Counter turns a contentious shared integer into independent per-replica monotonic components.

### Challenge question
What breaks if you allow decrements in a G-Counter without changing the design?

---

## [INVESTIGATE] PN-Counter — When You Need Decrements Too

### Scenario
You track “active orders”:
- increment when order created
- decrement when completed/canceled

### Interactive question (pause and think)
How can you support decrements while still keeping monotonic state per replica?

### Explanation
PN-Counter = two G-Counters:
- P: increments
- N: decrements

Value = sum(P) - sum(N)

Merge each component with element-wise max.

[CODE: Go, implement PN-Counter and show merges]

```python
# Implementing a PN-Counter (two G-Counters) with safe merges
from __future__ import annotations

class PNCounter:
    def __init__(self, replica_id: str):
        if not replica_id:
            raise ValueError("replica_id must be non-empty")
        self.replica_id = replica_id
        self.p, self.n = {replica_id: 0}, {replica_id: 0}

    def inc(self, delta: int = 1) -> None:
        if delta < 0:
            raise ValueError("use dec() for negative updates")
        self.p[self.replica_id] = self.p.get(self.replica_id, 0) + delta

    def dec(self, delta: int = 1) -> None:
        if delta < 0:
            raise ValueError("delta must be >= 0")
        self.n[self.replica_id] = self.n.get(self.replica_id, 0) + delta

    def merge(self, other: "PNCounter") -> None:
        if not isinstance(other, PNCounter):
            raise TypeError("can only merge with PNCounter")
        for m, om in ((self.p, other.p), (self.n, other.n)):
            for rid, cnt in om.items():
                if cnt < 0:
                    raise ValueError("invalid negative component")
                m[rid] = max(m.get(rid, 0), cnt)

    def value(self) -> int:
        return sum(self.p.values()) - sum(self.n.values())

# Usage example
x, y = PNCounter("central"), PNCounter("south")
x.inc(3); y.inc(1); y.dec(2)
x.merge(y); y.merge(x)
assert x.value() == y.value() == 2
```

### Failure scenarios
Same as G-Counter.

### Trade-offs
- still grows with replica count
- doesn’t enforce invariants like “never below zero” by itself

### Key insight
> You can represent non-monotonic values as the difference of two monotonic ones.

### Challenge question
Can PN-Counter guarantee value >= 0 without coordination? Why or why not?

---

## [MISCONCEPTION] “CRDTs Are Just LWW Everywhere”

### Scenario
A team says: “We’ll just do last-write-wins with timestamps for everything. That’s a CRDT, right?”

### Interactive question (pause and think)
Is LWW a CRDT? If yes, when is it a bad idea?

### Explanation
LWW-Register can be modeled as a CRDT if:
- you define a total order on updates (timestamp + replica ID tie-break)
- merge picks the max element

But LWW is often a data loss policy:
- concurrent updates overwrite each other
- you lose intent (e.g., two items added concurrently to a cart)

LWW is acceptable for some fields:
- profile picture
- “current status message”

It’s dangerous for accumulative semantics:
- carts
- sets
- counters (unless you accept loss)

### Key insight
> LWW is a valid convergence strategy, but it encodes “discard one side” as your business rule.

### Challenge question
Name one domain field where LWW is correct, and one where it’s dangerous.

---

## [AGREEMENT] Sets — Where Conflicts Become Real

### Scenario
Customers can add/remove items from a shopping cart while offline.

- North adds “bagel”
- South removes “bagel”
- These happen concurrently

### Interactive question (pause and think)
What should the merged cart contain?

Options:
1) bagel present
2) bagel absent
3) depends on causality (“remove wins only if it saw the add”)

Pause and think.

### Explanation with analogy
Think of a cart as a clipboard that gets copied to each branch. If one branch removes “bagel” without ever seeing that another branch added it, should that removal delete the unseen future addition?

Many systems choose: Observed-Remove semantics.

### Key insight
> With sets, “conflict resolution” is really “what does remove mean under concurrency?”

### Challenge question
In your product, would users expect add-wins or observed-remove behavior for carts? Why?

---

## [INVESTIGATE] OR-Set (Observed-Remove Set) — “Remove What You’ve Seen”

### Scenario
You want cart semantics that match user intent:
- If I remove an item I see, it should be removed.
- But my removal shouldn’t erase someone else’s concurrent add I didn’t know about.

### Interactive question (pause and think)
How can a set remember which add a remove is targeting?

### Explanation
OR-Set assigns a unique tag to each add:
- Add(x) creates tag t and stores (x, t)
- Remove(x) removes only tags it has observed for x

Merge = union of add-tag pairs minus removed tags.

[IMAGE: timeline diagram showing concurrent add and remove of same element with tags; remove only deletes observed tags]

[CODE: TypeScript, implement a simple OR-Set with add tags and remove tombstones; show convergence]

```ts
// Implementing an OR-Set (Observed-Remove Set) with add-tags + tombstones
import { randomUUID } from "crypto";

type Tag = string;
export class ORSet<T> {
  private adds = new Map<T, Set<Tag>>();
  private tombstones = new Set<Tag>();

  add(value: T): Tag {
    const tag = randomUUID();
    (this.adds.get(value) ?? this.adds.set(value, new Set()).get(value)!).add(tag);
    return tag;
  }

  remove(value: T): void {
    const tags = this.adds.get(value);
    if (!tags) return; // removing unseen value is a no-op (observed-remove)
    for (const t of tags) this.tombstones.add(t);
  }

  merge(other: ORSet<T>): void {
    for (const [v, tags] of other.adds) {
      const mine = this.adds.get(v) ?? this.adds.set(v, new Set()).get(v)!;
      for (const t of tags) mine.add(t);
    }
    for (const t of other.tombstones) this.tombstones.add(t);
  }

  values(): Set<T> {
    const out = new Set<T>();
    for (const [v, tags] of this.adds) if ([...tags].some(t => !this.tombstones.has(t))) out.add(v);
    return out;
  }
}

// Usage example
const a = new ORSet<string>(), b = new ORSet<string>();
a.add("bagel"); b.remove("bagel"); b.add("bagel"); // concurrent remove + add
b.merge(a); a.merge(b);
console.log([...a.values()], [...b.values()]); // converges deterministically
```

### Failure scenarios
- duplicates: union behavior with sets
- reordering: tags ensure deterministic reconciliation

### Trade-offs
- metadata growth (tombstones)
- requires compaction / garbage collection with causal stability

### Key insight
> OR-Set encodes causality at the element level by tracking unique add instances.

### Challenge question
Why can’t a plain “add-wins set” without tags represent the difference between two concurrent adds of the same element?

---

## [WARNING] Tombstones and Garbage Collection (The Part Everyone Forgets)

### Scenario
Your OR-Set stores removed tags forever. Over months, metadata grows.

### Interactive question (pause and think)
Can you delete tombstones safely?

### Explanation
To remove tombstones, you need to know that all replicas have seen the remove (or at least won’t reintroduce the removed tag).

This is the concept of causal stability:
- if you know an operation is stable (delivered everywhere), you can compact metadata

This often requires extra machinery:
- version vectors
- anti-entropy sessions
- membership tracking

### Real-world parallel
A restaurant can throw away old paper receipts only after accounting confirms every branch has reported them.

### Key insight
> Many CRDTs trade coordination-free updates for metadata that must be managed with careful compaction.

### Challenge question
What could happen if you garbage-collect tombstones too early?

---

## [GAME] Decision Game — Shopping Cart Semantics

### Scenario
Two devices modify a cart concurrently:
- Device A removes item X
- Device B adds item X

### Which statement is true? (pause and think)
1) Add-wins is always correct because users prefer not to lose items.
2) Remove-wins is always correct because explicit removals should dominate.
3) There is no universally correct answer; you must choose semantics per domain.

### Explanation
“Correct” depends on intent:
- In a cart, add-wins might be less surprising.
- In an access control list, remove-wins might be safer.

CRDT design is about encoding these semantics explicitly.

### Key insight
> CRDTs don’t eliminate product decisions; they force you to make them explicit and consistent under concurrency.

### Challenge question
Pick a domain (cart, ACL, feature flags). Which semantics would you choose and why?

---

## [PUZZLE] Text Editing CRDTs — Why They’re Harder Than Sets

### Scenario
You’re building a collaborative note editor.

Two users concurrently insert characters at the same position.

### Interactive question (pause and think)
Why can’t we just use an OR-Set of characters?

### Explanation
Text has order. Sets don’t.

Text CRDTs (e.g., RGA, Logoot, LSEQ) assign identifiers to positions so that:
- concurrent inserts get unique positions
- the total order is deterministic

But there are trade-offs:
- identifier growth
- performance under heavy editing
- tombstone accumulation for deletes

[IMAGE: depiction of a sequence CRDT where characters have position IDs between neighbors; concurrent inserts create IDs that still sort]

[CODE: Pseudocode, show how a sequence CRDT assigns position IDs and merges concurrent inserts]

```ts
// Implementing the core idea of a sequence CRDT: stable position IDs + deterministic ordering
type Pos = number[]; // simplified "fractional" position (real CRDTs use denser encodings)

type Atom = { pos: Pos; site: string; ch: string; tombstone?: boolean };

export function comparePos(a: Atom, b: Atom): number {
  const n = Math.max(a.pos.length, b.pos.length);
  for (let i = 0; i < n; i++) {
    const da = a.pos[i] ?? 0, db = b.pos[i] ?? 0;
    if (da !== db) return da - db;
  }
  return a.site.localeCompare(b.site); // tie-break for concurrent inserts
}

export function allocBetween(left: Pos, right: Pos): Pos {
  // Production note: real systems avoid unbounded growth via base/bucket strategies.
  const l = left[0] ?? 0, r = right[0] ?? 1_000_000;
  if (r - l <= 1) throw new Error("no space between positions; rebalance needed");
  return [Math.floor((l + r) / 2)];
}

// Usage example: concurrent inserts at same spot get different (pos,site) ordering
const left: Atom = { pos: [100], site: "A", ch: "" }, right: Atom = { pos: [200], site: "A", ch: "" };
const x: Atom = { pos: allocBetween(left.pos, right.pos), site: "A", ch: "H" };
const y: Atom = { pos: x.pos, site: "B", ch: "i" }; // same pos, different site => deterministic order
console.log([x, y].sort(comparePos).map(a => a.ch).join(""));
```

### Key insight
> Sequence CRDTs replace “index positions” with stable identifiers so concurrent edits can be merged deterministically.

### Challenge question
What happens if your position identifiers grow without bound? What engineering techniques mitigate this?

---

## [INVESTIGATE] Delivery Guarantees — Causal vs Eventual vs “I Hope So”

### Scenario
You choose an op-based CRDT for efficiency. Your network can reorder messages.

### Interactive question (pause and think)
If operations commute, do you still need causal delivery?

### Explanation with analogy
Many op-based CRDTs require either:
- operations that are truly commutative under all reorderings, or
- causal delivery so that “remove” doesn’t arrive before “add” it depends on

Some designs embed enough metadata to relax delivery guarantees.

Mental model: “Kitchen tickets”
If the kitchen receives “cancel burger” before it receives “make burger,” it might ignore the cancel and still make the burger later.

Causal delivery ensures dependent instructions arrive in the right relationship.

### Key insight
> Op-based CRDTs often require causal ordering or additional metadata to be safe under arbitrary reordering.

### Challenge question
Would you rather pay for causal delivery in the network layer or in the CRDT metadata? Why?

---

## [WARNING] Failure Scenarios Walkthrough (CRDTs Under Stress)

### Scenario
Your replicas experience:
- partitions
- retries causing duplicates
- clock skew
- node restarts with lost local state

### Interactive question (pause and think)
Which of these failures do CRDTs handle naturally, and which require additional system design?

### Progressive reveal
Pause and think before reading.

#### Handles naturally (given correct implementation)
- Duplicate messages (idempotent merge)
- Reordering (commutative merge)
- Partitions (local writes continue; eventual convergence)

#### Needs extra design
- Clock skew: if you use timestamps (LWW), skew can cause surprising wins. Use hybrid logical clocks (HLC) or logical clocks.
- Node restarts with lost local state: state-based CRDTs assume state is durable; otherwise you “forget” increments and can violate monotonicity.
- Membership churn: replica IDs and vector sizes need management.

### Real-world parallel
If a branch loses its tally sheet in a fire, it can’t reconstruct how many coffees it sold unless it had backups or receipts.

### Key insight
> CRDTs are not a persistence layer. Durability and identity management still matter.

### Challenge question
What’s the worst outcome if a replica loses its local CRDT state and rejoins with zeros?

---

## [AGREEMENT] Anti-Entropy — How Replicas Actually Sync

### Scenario
You have 50 replicas. You can’t broadcast full state constantly.

### Interactive question (pause and think)
How do systems ensure all replicas eventually receive all updates?

### Explanation
Common approaches:
- Gossip / epidemic protocols: randomly sync with peers.
- State exchange with version vectors: send summaries, then missing deltas.
- Delta-state CRDTs: send compact deltas instead of full state.

[IMAGE: gossip anti-entropy diagram showing random peer sync rounds converging]

### Delta-state CRDTs (practical win)
Instead of sending full state, send only the “delta” since last sync.
- keeps state-based semantics
- reduces bandwidth

[CODE: Rust, sketch delta-state G-Counter where increment returns a delta and merge applies it]

```python
# Implementing a delta-state G-Counter: increment returns a delta you can ship over the network
from __future__ import annotations

class DeltaGCounter:
    def __init__(self, replica_id: str):
        if not replica_id:
            raise ValueError("replica_id must be non-empty")
        self.replica_id, self.state = replica_id, {replica_id: 0}

    def increment(self, n: int = 1) -> dict[str, int]:
        if n < 0:
            raise ValueError("cannot decrement")
        self.state[self.replica_id] = self.state.get(self.replica_id, 0) + n
        return {self.replica_id: self.state[self.replica_id]}  # delta = updated component

    def merge_delta(self, delta: dict[str, int]) -> None:
        if not isinstance(delta, dict):
            raise TypeError("delta must be a dict")
        for rid, cnt in delta.items():
            if not isinstance(rid, str) or not isinstance(cnt, int) or cnt < 0:
                raise ValueError("invalid delta")
            self.state[rid] = max(self.state.get(rid, 0), cnt)

    def value(self) -> int:
        return sum(self.state.values())

# Usage example: ship only deltas; duplicates are safe
n, s = DeltaGCounter("north"), DeltaGCounter("south")
d1 = n.increment(2); d2 = s.increment(1)
s.merge_delta(d1); s.merge_delta(d1)  # duplicate delta
n.merge_delta(d2)
assert n.value() == s.value() == 3
```

### Key insight
> Anti-entropy is the engine; CRDTs are the steering wheel. You need both for convergence in real deployments.

### Challenge question
What happens if anti-entropy never completes (e.g., a replica is permanently offline)? Does CRDT convergence still hold?

---

## [MISCONCEPTION] “Eventual Consistency Means No Guarantees”

### Scenario
An engineer says: “Eventual consistency is basically random; you can’t reason about it.”

### Interactive question (pause and think)
Is that true for CRDT-based eventual consistency?

### Explanation
CRDTs provide strong guarantees:
- Strong eventual consistency (SEC): if replicas receive the same set of updates, they converge to the same state.
- deterministic merge semantics

What’s not guaranteed:
- immediate agreement
- invariants that require coordination

### Key insight
> CRDTs turn eventual consistency from “best effort” into a precisely defined convergence model.

### Challenge question
Define SEC in your own words in one sentence.

---

## [INVESTIGATE] Trade-offs — What You Pay to Avoid Coordination

### Scenario
Your team is choosing between:
- CRDTs
- leader-based strong consistency

### Interactive question (pause and think)
What do you “spend” when you choose CRDTs?

### Explanation
Common costs:
- metadata overhead (vectors, tags, tombstones)
- weaker invariants unless you add coordination
- complexity in semantics (add-wins vs remove-wins)
- debuggability (harder to reason about intermediate states)

Benefits:
- local low-latency writes
- high availability under partitions
- natural offline support
- multi-region active-active

### Comparison table: CRDTs vs Strong Consistency

| Dimension | CRDT replication | Strong consistency (Raft/Paxos) |
|---|---|---|
| Writes under partition | continue | typically blocked or redirected |
| Latency | local | cross-quorum round trips |
| Convergence | deterministic | immediate agreement |
| Invariants | limited without coordination | easier to enforce |
| Metadata | can grow | typically smaller |
| Use cases | offline-first, collaboration, counters, carts | money transfers, inventory, strict ordering |

### Key insight
> CRDTs are a deliberate CAP trade: prioritize availability and convergence over immediate agreement and some invariants.

### Challenge question
Name a feature in your system that must be strongly consistent, and one that could be CRDT-based.

---

## [PUZZLE] Designing With CRDTs — Start From Semantics, Not Data Structures

### Scenario
You’re building a “team task board”:
- tasks can be added, moved between columns, renamed, assigned
- users may be offline

### Interactive prompt (pause and think)
For each field, decide the conflict policy:
- title
- description
- assignee
- column
- tags

### Explanation with analogy
A practical approach:
1) Identify which fields are commutative (can merge by union/addition).
2) Identify which fields are single-writer or can tolerate LWW.
3) For tricky invariants (e.g., “task must be in exactly one column”), consider:
   - modeling as a map with LWW for column
   - or using coordination for moves

Mental model: “Different kitchen stations”
Not every part of the order needs the same conflict policy:
- drinks station can independently add items
- cashier station sets payment status (single source of truth)

### Key insight
> CRDT architecture is usually hybrid: CRDT where it fits, coordination where it must.

### Challenge question
Which fields in the task board can be modeled as sets/counters, and which need registers?

---

## [GAME] Quiz — Identify the CRDT (Progressive Reveal)

### Question 1
You need a replicated value where concurrent updates should be merged by taking the maximum numeric value.

Pause and think: is this a CRDT? If yes, what’s the merge?

Answer (reveal): Yes, it can be a state-based CRDT if state monotonically increases and merge = max.

### Question 2
You need a replicated bank balance with deposits and withdrawals, and balance must never go negative.

Pause and think.

Answer (reveal): Plain PN-Counter won’t enforce non-negativity. You need coordination or bounded counter/escrow.

### Question 3
You need a replicated set where remove must always win over add, even if add is concurrent.

Pause and think.

Answer (reveal): Use a remove-wins set variant (e.g., with tombstones dominating adds) or design with causal metadata.

### Key insight
> “Is it a CRDT?” is really “Can you define a merge that is deterministic and matches your semantics under concurrency?”

### Challenge question
Invent a merge rule for “status” with values {online, away, offline} that behaves well under concurrency.

---

## [INVESTIGATE] Real-World Usage Patterns

### Scenario
You want confidence that CRDTs aren’t just academic.

### Where CRDTs show up
- Collaborative editors: sequence CRDTs or OT/CRDT hybrids
- Shopping cart / wishlist: OR-Set / add-wins sets
- Counters and analytics: G/PN counters, bounded counters
- Edge and mobile offline-first: local-first state sync
- Databases: some systems expose CRDT-like datatypes (varies by product)

### Interactive question (pause and think)
Why are CRDTs attractive in multi-region active-active deployments?

### Explanation
They avoid leader election and synchronous quorum writes for many operations.
- lower tail latency
- better partition tolerance

### Key insight
> CRDTs are a practical tool for active-active and offline-first systems, when semantics allow.

### Challenge question
Which of your system’s state is “user intent that can be merged,” versus “global invariant that must be serialized”? List two examples.

---

## [WARNING] Security and Abuse Considerations (Yes, Really)

### Scenario
Clients can submit CRDT operations directly (offline-first). A malicious client can:
- replay old operations
- forge replica IDs
- send huge tombstone sets

### Interactive question (pause and think)
What threats are unique to CRDT-based replication?

### Explanation
CRDTs assume replicas are honest participants unless you add controls.
Mitigations:
- authenticate replicas/clients
- limit operation sizes
- server-side validation
- per-user namespaces
- rate limiting
- signed operations (in some architectures)

### Key insight
> CRDT correctness is not the same as trust. Convergence doesn’t prevent malicious state changes.

### Challenge question
If clients can generate unique tags for OR-Set adds, how do you prevent tag-space abuse?

---

## [PUZZLE] Putting It Together — A Mini Offline-First Design

### Scenario
You’re building “CafeNotes”: a mobile app for baristas to keep shared notes per shift.
Requirements:
- edit notes offline
- merge across devices
- show “likes” on notes
- maintain a checklist of tasks

### Design exercise (pause and think)
Pick CRDTs:
- likes counter
- checklist items
- note text

### Suggested design
- likes: G-Counter (or PN if unlikes exist)
- checklist: OR-Set of task IDs + LWW register for task fields
- note text: sequence CRDT (RGA/Logoot/LSEQ) or a library-provided text CRDT

[IMAGE: architecture diagram: mobile clients with local CRDT state, sync service doing anti-entropy, conflict-free merges]

### Key insight
> Real apps compose multiple CRDTs: maps of registers, sets of IDs, counters, and sequences.

### Challenge question
Where would you still use strong consistency in this app (if anywhere)?

---

## [INVESTIGATE] Answer Reveal — Earlier Questions

### 1) The “Three Coffee Shops” Problem
Correct choice: 3) Build data types whose merges are deterministic and converge automatically.

### 2) “Conflict-free” meaning
Correct answer: B) No conflicts at the replication protocol level because merges are well-defined.

### 3) Merge properties
Required: Associativity + Commutativity + Idempotence (for state-based merge under duplication/reordering).

### 4) Decision game: invariants
Correct statement: 2) CRDTs guarantee convergence, but invariants may require coordination or special designs.

### 5) Matching exercise answers
1 -> A (G-Counter)
2 -> B (PN-Counter)
3 -> C (OR-Set)
4 -> D (LWW-Register)

---

## [CHALLENGE] Final Synthesis Challenge: Design a CRDT-Based Feature Like a Pro

### Scenario
You’re designing a multi-region, offline-capable “incident response board”:
- incidents can be created and tagged
- responders can assign themselves
- status transitions: open -> investigating -> mitigated -> resolved
- comments are appended
- you must not allow “resolved” to go back to “open”

### Your mission (pause and think)
1) For each field, choose a CRDT (or coordination) strategy:
   - tags
   - assignees
   - status
   - comments
2) Identify one place where CRDTs might violate an invariant.
3) Propose a mitigation.

### Progressive reveal (one possible solution)
- tags: OR-Set (add/remove)
- assignees: OR-Set (or add-wins set)
- comments: grow-only list / sequence CRDT (append-only is easiest)
- status: monotonic register with a lattice order (open < investigating < mitigated < resolved) and merge = max

Invariant risk:
- if you allow arbitrary status edits, someone could set status backward. Use a monotonic lattice where backward moves are ignored, or require coordination for certain transitions.

Mitigation:
- model status as a max-semilattice so it only moves forward
- or enforce via server-side validation + signed ops

### Key insight
> The best CRDT designs align your domain invariants with monotonic progress so the math works with your business logic.

---

## [GOODBYE] Closing: When CRDTs Should Be Your Default (and When They Shouldn’t)

Use CRDTs when:
- you need offline-first or active-active
- you can define merge semantics that preserve user intent
- you can tolerate eventual convergence

Avoid (or constrain) CRDTs when:
- strict global invariants dominate (money, inventory)
- you need total order of operations
- metadata growth is unacceptable without a compaction strategy

### Final challenge question
Pick one feature you’re building right now. Write down:
- the state
- the operations
- the concurrency cases
- the merge policy

If you can’t write a merge policy you’d be happy to explain to a teammate, you’re not ready for CRDTs yet.
