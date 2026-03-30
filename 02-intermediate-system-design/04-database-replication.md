# Database Replication

Replication means keeping copies of your data on multiple machines. It's the foundation of high availability, read scalability, and disaster recovery. Understanding replication — and its trade-offs — is essential for designing reliable systems.

---

## Why Replicate?

```
┌─────────────────────────────────────────────────────┐
│           Reasons to Replicate                      │
│                                                     │
│  High Availability:  If primary dies, replica       │
│                      takes over — zero downtime     │
│                                                     │
│  Read Scalability:   Route reads to replicas,       │
│                      offload primary for writes     │
│                                                     │
│  Geographic Locality: Put replicas near users       │
│                        for low-latency reads        │
│                                                     │
│  Disaster Recovery:  Replica in a different region  │
│                      survives datacenter failure    │
└─────────────────────────────────────────────────────┘
```

---

## Single-Leader Replication (Leader-Follower)

The most common model. One node accepts all writes (the leader/primary). Followers replicate the leader's data and serve reads.

```
         Writes
           │
           ▼
    ┌─────────────┐      Replication log
    │   Leader    │──────────────────────────▶ ┌────────────┐
    │  (Primary)  │                            │ Follower 1 │
    └─────────────┘──────────────────────────▶ └────────────┘
           │             Replication log
           └───────────────────────────────▶   ┌────────────┐
                                               │ Follower 2 │
    Reads ──────────────────────────────────▶  └────────────┘
    (any follower or leader)
```

**How replication happens:**
1. Leader writes to its log (WAL — Write-Ahead Log or binlog)
2. Followers stream the log and apply changes in the same order
3. Followers serve reads from their local copy

### Synchronous vs Asynchronous Replication

**Synchronous:**
```
Client ──write──▶ Leader ──wait for ack──▶ Follower
Client ◀──"done"── Leader (only after follower confirms)

Pro:  If leader dies, follower has all data
Con:  Writes are slower — must wait for follower
Con:  If follower is down, leader blocks
```

**Asynchronous:**
```
Client ──write──▶ Leader ──"done"──▶ Client
                      │
                      └──(send to follower eventually)──▶ Follower

Pro:  Writes are fast — don't wait for follower
Con:  If leader dies before follower catches up, data lost
Con:  Replication lag — follower may be seconds/minutes behind
```

**Semi-synchronous (common in MySQL):**
At least one follower must acknowledge, then the write completes. Balances durability and performance.

---

## Replication Lag

Asynchronous replication means followers lag behind the leader. This creates subtle bugs:

### Read-After-Write Inconsistency

```
t=0: User Alice updates her profile bio
t=0: Write goes to leader
t=1: Alice refreshes her profile page
t=1: Request routed to Follower 1
t=1: Follower 1 hasn't received update yet → shows old bio
t=2: Alice thinks her update was lost → refreshes again
```

**Solutions:**
- Route reads of a user's own data to the leader
- Remember timestamp of last write; read from replica only if replica is caught up to that timestamp
- Read from leader for a short window after a write

### Monotonic Read Consistency

Without it, a user can see "time go backwards":

```
User reads post #42 from Follower 1 (up-to-date)  → sees 5 comments
User refreshes, routed to Follower 2 (lagging)    → sees 3 comments
```

**Solution:** Route a given user's reads to the same follower consistently (using a hash of user_id).

---

## Multi-Leader Replication

Multiple nodes accept writes. Leaders replicate to each other and to followers.

```
   Datacenter 1                    Datacenter 2
┌──────────────────┐            ┌──────────────────┐
│  ┌────────────┐  │            │  ┌────────────┐  │
│  │  Leader 1  │  │◄──────────▶│  │  Leader 2  │  │
│  └─────┬──────┘  │            │  └─────┬──────┘  │
│        │         │            │        │         │
│  ┌─────▼──────┐  │            │  ┌─────▼──────┐  │
│  │ Followers  │  │            │  │ Followers  │  │
│  └────────────┘  │            │  └────────────┘  │
└──────────────────┘            └──────────────────┘
```

**Advantages:**
- Writes can happen in the nearest datacenter (low latency)
- Datacenter failure doesn't stop writes

**The big problem: Write Conflicts**

```
t=0: User A changes title to "Hello" on Leader 1
t=0: User B changes title to "World" on Leader 2 (simultaneously)
t=1: Leaders replicate to each other
t=1: Conflict! Same record has two different values.
```

### Conflict Resolution Strategies

| Strategy | How | Problem |
|----------|-----|---------|
| Last Write Wins (LWW) | Keep the value with the most recent timestamp | Timestamps are unreliable; data loss |
| First Write Wins | Keep the first write; reject the second | User B's change is silently discarded |
| Merge | Combine both values (works for CRDTs) | Complex, domain-specific logic |
| Custom resolver | Application code decides | Complex, requires business logic |
| Avoid conflicts | Route same user's writes to same leader | Reduces performance benefit |

---

## Leaderless Replication (Dynamo-Style)

No designated leader. Any replica can accept reads and writes. Popularized by Amazon's Dynamo paper; used in Cassandra, Riak, DynamoDB.

```
                ┌──────────┐
                │ Client   │
                └────┬─────┘
           ┌─────────┼─────────┐
           ▼         ▼         ▼
     ┌──────────┐ ┌──────┐ ┌──────────┐
     │ Node A   │ │Node B│ │ Node C   │
     │ (accept) │ │(down)│ │ (accept) │
     └──────────┘ └──────┘ └──────────┘

Write to any N nodes. Read from any N nodes.
Use quorum to determine latest value.
```

### Quorum Reads and Writes

With N replicas, W write acknowledgments required, R read acknowledgments required:

**Strong consistency:** W + R > N

```
Example: N=3, W=2, R=2

Write: ──▶ 2 of 3 nodes ack
Read:  ──▶ read from 2 of 3 nodes, return latest version

At least 1 node overlaps → guaranteed to see latest write
```

**High availability (eventual consistency):** W=1, R=1

```
Write: just 1 node acks (very fast)
Read:  read from just 1 node (very fast)
Risk:  may read stale data if replicas haven't converged
```

**Tunable consistency** — DynamoDB and Cassandra let you choose W and R per operation.

### Sloppy Quorums and Hinted Handoff

During a network partition, fewer than N nodes may be reachable. A sloppy quorum allows writing to "wrong" nodes temporarily:

```
Normal quorum unavailable (nodes A, B, C but A is down):
  Write to B and D instead (D is outside normal quorum)
  D stores the write with hint: "this belongs to A"

When A recovers:
  D sends the hinted write to A (hinted handoff)
  Eventually all nodes are consistent
```

### Read Repair and Anti-Entropy

**Read repair:** When reading from multiple replicas, if they disagree, update the stale replicas in the background.

**Anti-entropy:** A background process continuously compares replicas and syncs differences using Merkle trees.

---

## Replication Topologies

```
Star (Single Leader):     Circular:         All-to-All:

      L                   A──▶B──▶C         A◄──▶B
     /|\                  ▲         │        ▲   /│\
    F F F                 └─────────┘        │  / │ \
                                             ▼ ▼  ▼  ▼
                                              C◄──▶D
```

| Topology | Fault Tolerance | Replication Lag |
|----------|----------------|-----------------|
| Star (single leader) | Medium | Low |
| Circular | Low (break = stop) | Medium |
| All-to-all | High | Can arrive out of order |

---

## Handling Leader Failure: Failover

When the leader dies:

```
Step 1: Detect failure (via heartbeat timeout)
Step 2: Choose new leader (node with most up-to-date log)
Step 3: Reconfigure system to route writes to new leader
Step 4: Old leader (if it recovers) must recognize it's a follower
```

**Pitfalls:**
- **Split-brain:** Two nodes both think they're leader → both accept conflicting writes. Solution: fencing tokens (only leader with highest token is valid).
- **Data loss:** Async replication means new leader may be missing some recent writes. Accept loss or use synchronous replication.
- **False positives:** Network glitch causes healthy leader to be demoted. Solution: longer timeouts, multiple health check sources.

---

## Real-World Implementations

| System | Replication Model | Notes |
|--------|-----------------|-------|
| MySQL | Single-leader (async or semi-sync) | Binlog replication |
| PostgreSQL | Single-leader (streaming replication) | WAL shipping or streaming |
| MongoDB | Single-leader (replica sets, 3+ nodes) | Automatic failover with Raft |
| Cassandra | Leaderless | Tunable consistency, gossip protocol |
| DynamoDB | Leaderless | Paxos-based multi-master |
| Redis | Single-leader | Asynchronous, Redis Sentinel for failover |

---

## Key Takeaways

1. **Single-leader replication is the simplest model** — one leader for writes, multiple followers for reads
2. **Synchronous replication guarantees durability** — at the cost of write latency
3. **Replication lag causes subtle consistency bugs** — read-after-write inconsistency is the most common
4. **Multi-leader enables multi-region writes** — but write conflicts are hard to resolve correctly
5. **Leaderless replication (Dynamo-style) maximizes availability** — any node accepts reads and writes
6. **W + R > N guarantees strong consistency** — tune W and R based on your consistency vs latency needs
7. **Failover is complex** — split-brain, data loss, and cascading failures are real risks

---

*Previous: [Message Queues ←](03-message-queues.md) · Next: [Rate Limiting →](05-rate-limiting.md)*
