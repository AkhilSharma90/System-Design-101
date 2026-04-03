---
slug: cap-theorem
title: CAP Theorem
readTime: 15 min
orderIndex: 7
premium: false
---

# CAP Theorem

The CAP theorem is one of the most important concepts in distributed systems. It defines the fundamental trade-off that every distributed database must make — understanding it will inform every database architecture decision you make.

---

## What the CAP Theorem Says

Proposed by Eric Brewer in 2000 and formally proved in 2002:

> **A distributed system can only guarantee two of the following three properties simultaneously:**

```
                    Consistency
                         ▲
                        / \
                       /   \
                      /     \
                     /       \
                    /  Choose \
                   /   2 of 3  \
                  /─────────────\
   Availability ◄─────────────────► Partition
                                    Tolerance
```

### The Three Properties

**Consistency (C)**
Every read receives the most recent write or an error. All nodes see the same data at the same time. If you write a value to node A, any subsequent read from node B returns that same value.

**Availability (A)**
Every request receives a non-error response — though it might not contain the most recent write. The system is always operational.

**Partition Tolerance (P)**
The system continues to operate even when network messages between nodes are dropped or delayed (a network partition).

---

## Why You Can't Have All Three

A network partition is not optional — it *will* happen. Networks fail, packets get dropped, servers get isolated. In any real distributed system, you must tolerate partitions.

This means the real choice is: **when a partition occurs, do you sacrifice Consistency or Availability?**

```
   Network Partition Scenario:

   ┌─────────────┐        X X X        ┌─────────────┐
   │   Node A    │  ─── PARTITION ─── │   Node B    │
   │  (writes)   │        X X X        │  (reads)    │
   └─────────────┘                     └─────────────┘

   Option 1 — Sacrifice Availability (CP):
   Node B rejects reads until partition heals → Consistent but unavailable

   Option 2 — Sacrifice Consistency (AP):
   Node B serves stale data → Available but inconsistent
```

---

## CP Systems — Consistency over Availability

CP systems prioritize returning correct data. When a partition occurs, they refuse to serve requests rather than risk returning stale data.

### Pause and think

If your payment system serves a stale balance during a network partition, what's the worst case? (A customer double-spends.) This is why financial systems choose CP.

**Examples of CP systems:**

| System | Use Case |
|--------|----------|
| HBase | Big data, batch analytics |
| ZooKeeper | Distributed coordination, config management |
| etcd | Kubernetes config store |
| MongoDB (strong consistency) | General purpose with consistency guarantees |
| CockroachDB | ACID transactions across nodes |

**When to choose CP:** Financial transactions, inventory management, auth tokens — anything where stale reads cause real harm.

---

## AP Systems — Availability over Consistency

AP systems prioritize staying online. When a partition occurs, they continue serving requests but may return stale data.

**Examples of AP systems:**

| System | Use Case |
|--------|----------|
| Cassandra | Time-series data, high write throughput |
| DynamoDB (default) | E-commerce, gaming, IoT |
| CouchDB | Offline-first applications |
| Riak | High availability key-value storage |
| DNS | Domain name resolution |

**When to choose AP:** Shopping carts, social media counts, user sessions, analytics — anything where brief inconsistency is tolerable.

---

## Why "CA" Doesn't Really Exist

A system that doesn't tolerate partitions is just a single-node system. The moment you have multiple nodes communicating over a network, partitions *will* happen.

Traditional relational databases (PostgreSQL, MySQL) on a single node are effectively CA — but only because they're not distributed. The moment you add replication, you're back to choosing between C and A.

---

## Consistency Models — The Spectrum

CAP treats consistency as binary, but real systems offer a spectrum:

```
Strong                                            Eventual
Consistency ◄────────────────────────────────► Consistency

   │              │              │              │
   ▼              ▼              ▼              ▼
Linearizable   Sequential    Causal        Eventual
Consistency    Consistency   Consistency   Consistency

(Slowest,                                 (Fastest,
 Safest)                                   Most Scalable)
```

**Linearizable:** Operations appear instantaneous and globally ordered. A read always returns the latest write. Most expensive.

**Sequential:** All nodes see operations in the same order, but not necessarily real-time.

**Causal:** Causally related operations are seen in the same order everywhere. Unrelated operations may differ.

**Eventual:** Given no new writes, all nodes will *eventually* converge. The most common model for AP systems.

---

## PACELC: A More Complete Model

CAP only describes behavior during partitions. PACELC extends it to normal operation:

> **P**artition → choose **A**vailability vs **C**onsistency
> **E**lse (no partition) → choose **L**atency vs **C**onsistency

| System | During Partition | Normal Operation |
|--------|-----------------|-----------------|
| DynamoDB, Cassandra | Availability (PA) | Latency (EL) |
| Google Spanner, VoltDB | Consistency (PC) | Consistency (EC) |

This is why even without failures, Spanner is slower than Cassandra — it pays for synchronous replication in latency.

---

## Tunable Consistency: The Practical Approach

Many modern systems let you tune consistency per operation:

### Cassandra's Quorum Math

```
Write to N=3 nodes. Choose consistency level:

ONE     → Write acknowledged by 1 node    (fastest)
QUORUM  → Write acknowledged by 2 nodes   (balanced)
ALL     → Write acknowledged by 3 nodes   (safest)
```

**Rule:** If W + R > N, at least one node overlaps → Strong consistency

```
N=3, W=2, R=2:  W + R = 4 > 3  →  Strongly consistent ✓
N=3, W=1, R=1:  W + R = 2 < 3  →  Eventually consistent (but fast)
```

---

## Real-World Examples

**Amazon Shopping Cart (AP)**
Amazon prioritizes availability — you can always add items even during partial outages. Occasionally two sessions merge on checkout (resolved by union of items). They explicitly chose AP for carts.

**Google Spanner (CP)**
Uses atomic clocks (TrueTime) to achieve strong consistency across datacenters globally. Higher latency, but used for Google Ads billing where accuracy is non-negotiable.

**DNS (AP)**
DNS record changes propagate gradually across the internet via TTL. You can always resolve a domain, but you might get a stale IP for minutes to hours after a change.

---

## Common Misconceptions

**"You can pick any two of C, A, P"**
You cannot opt out of P in real networks. The practical choice is always C vs A during partitions.

**"Eventual consistency means data loss"**
No. It means data *convergence* takes time, not that data is lost. All writes are persisted; they propagate over time.

**"CP systems are always the right choice"**
Not at all. For analytics, social features, and content delivery, AP systems are the right choice and perform dramatically better.

---

## Decision Framework

```
Can stale reads cause financial loss or data corruption?
          │
      YES ▼                              NO ▼
   Choose CP                   Is 100% uptime more important
   (ZooKeeper, etcd,            than perfect accuracy?
    CockroachDB)                         │
                               YES ▼     │      NO ▼
                           Choose AP     │   Evaluate per
                           (Cassandra,   │   operation with
                            DynamoDB)    │   tunable consistency
```

---

## Common Mistakes

| Mistake | Why it's wrong | Correct approach |
|---------|---------------|-----------------|
| "We'll build a CA system" | CA doesn't exist in distributed systems — partitions are inevitable | Accept that you must choose C or A during partitions |
| Choosing CP for everything | Unnecessary consistency kills performance and availability | Match consistency to the use case — social feeds don't need CP |
| Ignoring PACELC | CAP only covers partition behavior, not normal operation | Consider latency vs consistency trade-offs during normal operation too |
| Confusing eventual consistency with data loss | Eventually consistent doesn't mean data disappears | All writes are persisted; they just take time to propagate to all replicas |

---

## Key Takeaways

1. **Partition tolerance is mandatory** in real distributed systems — choose C or A, not C+A+P
2. **CP systems return errors during partitions** — they never serve stale data
3. **AP systems serve stale data during partitions** — they never go offline
4. **Consistency is a spectrum** — from linearizable (strongest) to eventual (most scalable)
5. **PACELC extends CAP** — latency vs consistency matters even without partitions
6. **Tunable consistency** (Cassandra, DynamoDB) lets you choose the trade-off per operation
7. **Match the model to the use case** — financial data needs CP, social feeds can use AP
