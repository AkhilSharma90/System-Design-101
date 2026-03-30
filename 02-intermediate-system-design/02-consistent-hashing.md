# Consistent Hashing

Consistent hashing is one of the most elegant algorithms in distributed systems. It solves a fundamental problem: how do you distribute data across N servers in a way that minimizes disruption when servers are added or removed? You'll find it inside Redis Cluster, Amazon DynamoDB, Apache Cassandra, CDNs, and load balancers.

---

## The Problem with Naive Hashing

The simplest approach to distributing data across N servers is modulo hashing:

```
server = hash(key) % N
```

This works beautifully — until you need to add or remove a server.

```
Before (3 servers): hash("user:alice") % 3 = 2  → Server 2
After  (4 servers): hash("user:alice") % 4 = 1  → Server 1  ← MOVED!

With N=3 → N=4:  ~75% of all keys change servers
With N=10 → N=11: ~91% of all keys change servers
```

In a cache layer, this means a ~75–90% cache miss rate every time you scale. Databases would need to physically migrate most of their data. This is catastrophic.

---

## The Hash Ring

Consistent hashing solves this by mapping both keys and servers onto a circular hash space (the "ring").

```
         0°
         │
    270° ─┼─ 90°     Hash space wraps around: 0 ... 2^32-1
         │
        180°

Visualized as a ring (values 0–360 for simplicity):

              0
           ╭────╮
        ╭──╯    ╰──╮
       │            │
  270  │   RING     │  90
       │            │
        ╰──╮    ╭──╯
           ╰────╯
             180
```

**Step 1: Place servers on the ring**

Hash each server's identifier to get a position on the ring.

```
hash("Server A") = 45   → Server A sits at position 45°
hash("Server B") = 135  → Server B sits at position 135°
hash("Server C") = 260  → Server C sits at position 260°

              0
           ╭────╮
        ╭──╯    ╰──╮
    A(45)│            │
  270   │            │90
        │  B(135)    │
         ╰──╮    ╭──╯
           ╰──────╯
          C(260)  180
```

**Step 2: Place keys on the ring**

Hash each key the same way.

```
hash("user:alice") = 80   → position 80°
hash("user:bob")   = 200  → position 200°
hash("user:carol") = 310  → position 310°
```

**Step 3: Assign key to the next server clockwise**

```
"user:alice" at 80° → next server clockwise is B at 135°  → Server B
"user:bob"   at 200° → next server clockwise is C at 260° → Server C
"user:carol" at 310° → next server clockwise is A at 45°  → Server A
                        (wraps around the ring)
```

---

## Adding a Server — Minimal Disruption

Add Server D at position 190°.

```
Before:               After adding D at 190°:
"user:bob" at 200°    "user:bob" at 200°
→ Server C at 260°    → Server D at 190° ← MOVED to D

Only "user:bob" moved. All other keys unaffected.
```

Only keys between the previous server's position and D's position need to move. On average: only **1/N keys** move when adding a server.

---

## Removing a Server — Minimal Disruption

Remove Server B at 135°.

```
"user:alice" at 80°
Was → Server B at 135°
Now → Server C at 260° (next clockwise after B is removed)

Only keys that were on B need to move. All others stay.
```

Again: only **1/N keys** are affected.

---

## The Uneven Distribution Problem

With only a few physical servers, the hash function may place them unevenly on the ring:

```
         0
      ╭──╯─╮
   ╭──╯     ╰──╮
  │  A(10°)     │
  │              │
  │ ←  90% of  → │
  │    the ring   │   ← B and C are clustered together
  │              │
   ╰──╮B(340°)──╯
      ╰──C(350°)─╯
```

Server A handles 90% of all keys — terrible distribution.

---

## Virtual Nodes (Vnodes)

The solution: each physical server is represented by **multiple points** on the ring.

```
Physical: Server A, Server B, Server C

Virtual nodes (3 vnodes per server):
  Server A → A1(45°), A2(160°), A3(310°)
  Server B → B1(80°), B2(220°), B3(350°)
  Server C → C1(120°), C2(270°), C3(30°)

             0
           ╭────╮
        ╭──╯C3  ╰──╮
      A1│    (30°)  │B1(80°)
  A3(310│            │
  310°)  │ C1(120°)  │
         ╰──╮    ╭──╯
           B2╰──╯C2
         (220°) (270°)
```

Now even if physical servers are added unevenly, their vnodes are distributed across the ring. Distribution is much more uniform.

**Benefits of vnodes:**
- Even load distribution
- When a server is added, it takes a small fraction of load from *all* existing servers (not just one neighbor)
- When a server fails, its load spreads across all remaining servers

**Cassandra** uses 256 vnodes per node by default.

---

## Consistent Hashing in Practice

### Amazon DynamoDB
DynamoDB uses consistent hashing to distribute data across storage nodes. Each partition has a hash range (a slice of the ring), and items are assigned to partitions based on their partition key.

### Apache Cassandra
Cassandra's ring architecture is built on consistent hashing. The `MURMUR3` hash function maps partition keys to token positions. Virtual nodes (256 per node) ensure even data distribution.

### Redis Cluster
Redis Cluster uses **hash slots** — 16,384 slots arranged in a ring. Each node handles a contiguous range of slots. When nodes are added/removed, slots (and their data) are migrated.

```
Node A: slots 0–5460
Node B: slots 5461–10922
Node C: slots 10923–16383

Adding Node D: Each existing node transfers ~1/4 of its slots to D
```

### CDN Cache Routing
CDNs use consistent hashing to route requests for the same URL to the same edge server (maximizing cache hits). When edge servers are added or removed, only a small fraction of URLs are rerouted.

---

## Comparison: Hashing Strategies

| Strategy | Redistribution on change | Distribution | Complexity |
|----------|--------------------------|--------------|------------|
| Modulo hashing | ~(N-1)/N of keys | Even | Trivial |
| Consistent hashing | ~1/N of keys | Good (vnodes) | Low |
| Rendezvous hashing | ~1/N of keys | Very even | Low |
| Jump consistent hashing | ~1/N of keys | Very even | Low (no ring) |

### Rendezvous Hashing (Highest Random Weight)
For each key, score every server using `hash(key + server)`. Assign key to the highest-scoring server. Simple, no ring, great distribution — but O(N) per lookup.

### Jump Consistent Hashing
A compact algorithm by Google. Given a key and N buckets, computes the bucket in O(log N) with no data structure. Only works with sequentially numbered buckets (can't handle arbitrary server addition/removal).

---

## Key Takeaways

1. **Naive modulo hashing moves ~75% of keys when servers change** — catastrophic for caches
2. **Consistent hashing moves only ~1/N keys** — minimal disruption on topology changes
3. **The hash ring maps both servers and keys to positions** — keys go to the next server clockwise
4. **Virtual nodes solve uneven distribution** — each physical server gets multiple ring positions
5. **Adding a node takes load from all existing nodes equally** — no single neighbor overwhelmed
6. **Cassandra, DynamoDB, and Redis Cluster all use consistent hashing** — it's foundational
7. **Rendezvous and jump hashing are simpler alternatives** — useful in specific scenarios

---

*Previous: [Database Sharding ←](01-database-sharding.md) · Next: [Message Queues →](03-message-queues.md)*
