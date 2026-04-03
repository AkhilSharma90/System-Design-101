---
slug: consistent-hashing
title: Consistent Hashing
readTime: 15 min
orderIndex: 5
premium: false
---

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

With N=3 → N=4:   ~75% of all keys change servers
With N=10 → N=11:  ~91% of all keys change servers
```

In a cache layer, this means a ~75–90% cache miss rate every time you scale. Databases would need to physically migrate most of their data. This is catastrophic.

### Pause and think

If you cache 100 million items and adding one server causes 75 million cache misses — all hitting your database simultaneously — what happens to your database?

---

## The Hash Ring

Consistent hashing solves this by mapping both keys and servers onto a circular hash space (the "ring").

```
              0
           ╭────╮
        ╭──╯    ╰──╮
  A(45°)│            │
  270°  │            │  90°
        │   B(135°) │
         ╰──╮    ╭──╯
           ╰──────╯
          C(260°)  180°
```

**Step 1: Place servers on the ring**
Hash each server's identifier to get a position.

```
hash("Server A") = 45   → position 45°
hash("Server B") = 135  → position 135°
hash("Server C") = 260  → position 260°
```

**Step 2: Place keys on the ring**
```
hash("user:alice") = 80°
hash("user:bob")   = 200°
hash("user:carol") = 310°
```

**Step 3: Assign each key to the next server clockwise**
```
"user:alice" at 80°  → next clockwise: B at 135°  → Server B
"user:bob"   at 200° → next clockwise: C at 260°  → Server C
"user:carol" at 310° → next clockwise: A at 45°   → Server A (wraps)
```

---

## Adding a Server — Minimal Disruption

Add Server D at position 190°:

```
Before: "user:bob" at 200° → Server C at 260°
After:  "user:bob" at 200° → Server D at 190° ← Only this key moved
```

Only keys between the previous server position and D's position need to move. On average: **only 1/N keys** move when adding a server.

---

## Removing a Server — Minimal Disruption

Remove Server B at 135°:

```
"user:alice" at 80°
Was → Server B at 135°
Now → Server C at 260° (next clockwise after B is removed)
```

Only keys that were on B need to move. All others are unaffected. Again: **only 1/N keys** move.

---

## The Uneven Distribution Problem

With only a few physical servers, hash positions may cluster unevenly:

```
         0
      ╭──╯─╮
   ╭──╯A(10°)╰──╮
  │               │
  │  ← 90% of →  │   B and C are clustered together
  │    the ring   │   Server A gets most of the load
   ╰──╮B(340°)──╯
      ╰──C(350°)─╯
```

---

## Virtual Nodes (Vnodes) — The Fix

Each physical server is represented by **multiple positions** on the ring:

```
Physical servers: A, B, C

Virtual nodes (3 per server):
  Server A → A1(45°), A2(160°), A3(310°)
  Server B → B1(80°), B2(220°), B3(350°)
  Server C → C1(120°), C2(270°), C3(30°)
```

Now even if physical servers hash to clustered positions, their virtual nodes spread evenly. Load balances across all servers automatically.

**Benefits of vnodes:**
- Even load distribution regardless of hash clustering
- When a server is added, it takes a fraction of load from **all** existing servers (not just one neighbor)
- When a server fails, its load spreads across **all** remaining servers

**Cassandra** uses 256 vnodes per node by default.

---

## Consistent Hashing in Practice

### Amazon DynamoDB

DynamoDB uses consistent hashing to distribute data across storage nodes. Each partition has a hash range (a slice of the ring), and items are assigned to partitions based on their partition key.

### Apache Cassandra

Cassandra's ring architecture is built on consistent hashing with the `MURMUR3` hash function. 256 virtual nodes per physical node ensure even data distribution.

### Redis Cluster

Redis Cluster uses **16,384 hash slots** arranged as a ring. Each node handles a contiguous range of slots:

```
Node A: slots 0–5460
Node B: slots 5461–10922
Node C: slots 10923–16383

Adding Node D: Each existing node transfers ~1/4 of its slots to D
```

### CDN Cache Routing

CDNs use consistent hashing to route requests for the same URL to the same edge server — maximizing cache hit rates. When edge servers are added or removed, only a fraction of URLs need to be rerouted.

---

## Comparison: Hashing Strategies

| Strategy | Keys moved on change | Distribution | Complexity |
|----------|---------------------|--------------|------------|
| Modulo hashing | ~(N-1)/N — catastrophic | Even | Trivial |
| Consistent hashing | ~1/N — minimal | Good with vnodes | Low |
| Rendezvous hashing | ~1/N | Very even | O(N) per lookup |
| Jump consistent hashing | ~1/N | Very even | O(log N), no ring |

**Rendezvous hashing:** For each key, score every server with `hash(key + server_id)`. Assign to highest-scoring server. Simple, no ring, great distribution — but O(N) per lookup as servers grow.

**Jump consistent hashing:** A compact Google algorithm. Computes the bucket in O(log N) with no data structure — but only works with sequentially numbered buckets.

---

## Common Mistakes

| Mistake | Why it's wrong | Correct approach |
|---------|---------------|-----------------|
| Using modulo hashing for distributed systems | Adding one server rehashes 75%+ of keys | Use consistent hashing — only 1/N keys move |
| Too few virtual nodes | Uneven distribution across physical servers | Use 100-256 virtual nodes per server (Cassandra default: 256) |
| Not accounting for heterogeneous servers | A powerful server gets the same load as a weak one | Assign more virtual nodes to servers with more capacity |
| Ignoring replication in ring design | Data on a failed node is lost | Replicate to the next N nodes clockwise on the ring |

---

## Key Takeaways

1. **Naive modulo hashing moves ~75% of keys** when you add one server — catastrophic for caches
2. **Consistent hashing moves only ~1/N keys** — minimal disruption on topology changes
3. **The hash ring maps servers and keys to positions** — keys go to the next server clockwise
4. **Virtual nodes solve uneven distribution** — each physical server gets multiple ring positions
5. **Adding a node takes load from all existing nodes equally** — no single neighbor overwhelmed
6. **Cassandra (256 vnodes), DynamoDB, and Redis Cluster all use consistent hashing** — it's foundational infrastructure
7. **Rendezvous and jump hashing are simpler alternatives** for specific use cases
