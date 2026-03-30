# CAP Theorem

The CAP theorem is one of the most important concepts in distributed systems. It defines the fundamental trade-off that every distributed database must make — and understanding it will inform every database choice you make for the rest of your career.

---

## What the CAP Theorem Says

Proposed by Eric Brewer in 2000 and formally proved in 2002, the CAP theorem states:

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

```
┌─────────────────────────────────────────────────────┐
│                 CP System Behavior                   │
│                                                     │
│   Client ──▶ Node B (isolated)                      │
│                    │                                │
│                    ▼                                │
│           "I can't reach the leader.                │
│            Returning error rather than              │
│            potentially stale data."                 │
│                    │                                │
│                    ▼                                │
│           Client receives 503 Service Unavailable   │
└─────────────────────────────────────────────────────┘
```

**Examples of CP systems:**
| System | Use Case |
|--------|----------|
| HBase | Big data, batch analytics |
| ZooKeeper | Distributed coordination, config management |
| etcd | Kubernetes config store |
| MongoDB (default) | General purpose (with strong consistency setting) |
| CockroachDB | ACID transactions across nodes |

**When to choose CP:** Financial transactions, inventory management, anything where reading stale data causes real harm.

---

## AP Systems — Availability over Consistency

AP systems prioritize staying online. When a partition occurs, they continue serving requests but may return stale data.

```
┌─────────────────────────────────────────────────────┐
│                 AP System Behavior                   │
│                                                     │
│   Client ──▶ Node B (isolated)                      │
│                    │                                │
│                    ▼                                │
│           "I can't reach other nodes,               │
│            but I'll serve what I have.              │
│            It might be slightly stale."             │
│                    │                                │
│                    ▼                                │
│           Client receives data (possibly stale)     │
└─────────────────────────────────────────────────────┘
```

**Examples of AP systems:**
| System | Use Case |
|--------|----------|
| Cassandra | Time-series data, high write throughput |
| DynamoDB (default) | E-commerce, gaming, IoT |
| CouchDB | Offline-first applications |
| Riak | High availability key-value storage |
| DNS | Domain name resolution |

**When to choose AP:** Shopping carts, social media likes/counts, user sessions, anything where brief inconsistency is tolerable.

---

## Why "CA" Doesn't Really Exist

You'll sometimes see systems listed as "CA" — but this is misleading. A system that doesn't tolerate partitions is just a single-node system. The moment you have multiple nodes communicating over a network, you *will* experience partitions.

Traditional relational databases (PostgreSQL, MySQL) running on a single node are effectively CA — but only because they're not distributed. The moment you add replication or clustering, you're back to choosing between C and A.

---

## Consistency Models (Beyond Binary)

CAP treats consistency as binary, but real systems offer a spectrum:

```
Strong                                            Eventual
Consistency ◄────────────────────────────────► Consistency

   │              │              │              │
   ▼              ▼              ▼              ▼
Linearizable   Sequential    Causal        Eventual
Consistency    Consistency   Consistency   Consistency

(Slowest,                                 (Fastest,
 Safest)                                   Riskiest)
```

### Strong (Linearizable) Consistency
Operations appear instantaneous and in a global order. Reading immediately after a write always returns the new value. Most expensive.

### Sequential Consistency
All nodes see operations in the same order, but not necessarily in real-time order.

### Causal Consistency
Operations that are causally related are seen in the same order by all nodes. Unrelated operations may be seen in different orders.

### Eventual Consistency
Given no new updates, all nodes will *eventually* converge to the same value. The most common model for AP systems.

---

## PACELC: A Better Model

CAP only describes behavior during partitions. PACELC extends it to cover normal operation too:

> If there's a **P**artition, choose between **A**vailability and **C**onsistency.
> **E**lse (no partition), choose between **L**atency and **C**onsistency.

```
┌──────────────────────────────────────────────────────┐
│                   PACELC Trade-offs                  │
│                                                      │
│  During partition:     During normal operation:      │
│  ┌─────────────────┐   ┌─────────────────────────┐  │
│  │  PA/EL systems  │   │ Low latency, but         │  │
│  │  (e.g. Dynamo,  │   │ possibly stale reads     │  │
│  │   Cassandra)    │   └─────────────────────────┘  │
│  └─────────────────┘                                 │
│  ┌─────────────────┐   ┌─────────────────────────┐  │
│  │  PC/EC systems  │   │ Always consistent, but   │  │
│  │  (e.g. Spanner, │   │ higher latency           │  │
│  │   VoltDB)       │   └─────────────────────────┘  │
│  └─────────────────┘                                 │
└──────────────────────────────────────────────────────┘
```

---

## Tunable Consistency: The Practical Approach

Many modern systems let you tune consistency per operation. DynamoDB and Cassandra are the best examples:

### Cassandra's Tunable Consistency

```
Write to N=3 nodes. Choose consistency level:

ANY      → Write to 1 node (fastest, least safe)
ONE      → Write acknowledged by 1 node
QUORUM   → Write acknowledged by ⌈N/2⌉+1 = 2 nodes
ALL      → Write acknowledged by all 3 nodes (slowest, safest)
```

**Quorum reads + Quorum writes = Strong consistency**
If W + R > N, at least one node overlaps between reads and writes.

```
N=3, W=2, R=2: W + R = 4 > 3 → Strongly consistent
N=3, W=1, R=1: W + R = 2 < 3 → Eventually consistent (but fast)
```

---

## Choosing CP vs AP: Decision Framework

```
Is data corruption or financial loss possible from stale reads?
          │
      YES ▼                          NO ▼
   ┌──────────────┐            ┌──────────────────────┐
   │  Choose CP   │            │  Is 100% uptime      │
   │              │            │  more important than  │
   │ Examples:    │            │  perfect accuracy?   │
   │ - Payments   │            └──────────┬───────────┘
   │ - Inventory  │                       │
   │ - Auth tokens│              YES ▼    │    NO ▼
   └──────────────┘         ┌──────────┐  │  ┌──────────┐
                            │Choose AP │  │  │ Evaluate │
                            │          │  │  │ per-use  │
                            │Examples: │  │  │ case     │
                            │- Carts   │  │  └──────────┘
                            │- Feeds   │  │
                            │- Counters│  │
                            └──────────┘  │
```

---

## Real-World Examples

### Amazon Shopping Cart (AP)
Amazon prioritizes availability for their cart — you can always add items. Occasionally two sessions merge on checkout (the cart conflict resolves by union of items). Users can add to cart even during partial outages.

### Google Spanner (CP)
Google's globally distributed SQL database uses atomic clocks (TrueTime) to achieve strong consistency across datacenters. Pays the price in latency but used for mission-critical data (Google Ads billing).

### DNS (AP)
DNS favors availability and uses eventual consistency. DNS record changes propagate gradually across the internet (TTL-based). You can always resolve a name, but you might get a stale IP for a short window.

---

## Common Misconceptions

**"You can pick any two of C, A, P"**
You can't opt out of P in real networks. The choice is C vs A during partitions.

**"Eventual consistency means data loss"**
No. It means data *convergence* takes time, not that data is lost. All writes are persisted; they just take time to propagate.

**"CP systems are always better"**
Not at all. For many use cases (analytics, social features, content), AP systems are the right choice and perform dramatically better.

---

## Key Takeaways

1. **In distributed systems, partition tolerance is mandatory** — you choose between C and A
2. **CP systems return errors during partitions** — they never serve stale data
3. **AP systems serve stale data during partitions** — they never go offline
4. **Consistency is a spectrum** — from linearizable to eventual
5. **Many systems offer tunable consistency** — Cassandra, DynamoDB let you choose per-operation
6. **PACELC extends CAP** — the latency vs consistency trade-off matters even without partitions
7. **Match the model to the use case** — financial data needs CP, social feeds can use AP

---

*Previous: [Latency and Throughput ←](04-latency-and-throughput.md) · Next: [Load Balancing →](06-load-balancing.md)*
