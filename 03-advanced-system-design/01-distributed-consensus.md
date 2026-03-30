# Distributed Consensus

Distributed consensus is the problem of getting multiple nodes in a distributed system to agree on a single value — even when some nodes fail or messages are delayed. It sounds simple. It is one of the hardest problems in computer science.

---

## Why Consensus is Hard

In a distributed system, you can't rely on:
- **Reliable message delivery** — messages can be lost, delayed, or reordered
- **Synchronized clocks** — you can't know the exact time across nodes
- **Reliable nodes** — servers crash, restart, and may be slow

### The FLP Impossibility Result

In 1985, Fischer, Lynch, and Paterson proved:

> *It is impossible for a deterministic distributed system to reach consensus in an asynchronous network if even one process can fail.*

This means you must make trade-offs. Real systems get around FLP by:
- Using **timeouts** (assuming a node that hasn't responded in X time has failed)
- Using **randomization** (probabilistically breaking ties)
- Using **partial synchrony** (assuming messages arrive within some unknown but finite time)

---

## What is Consensus Useful For?

Consensus underpins critical distributed systems:

```
Leader Election:       Which node is the primary for this shard?
Distributed Lock:      Only one process holds the lock at a time
Configuration Store:   All nodes see the same config (etcd, ZooKeeper)
Database Replication:  All replicas apply changes in the same order
Atomic Broadcast:      All nodes receive all messages in the same order
```

---

## Paxos (the Original)

Paxos was the first practical consensus algorithm, published by Leslie Lamport in 1989 (a paper so confusing it took years to be accepted). It defines three roles:

- **Proposer** — initiates the consensus (proposes a value)
- **Acceptor** — votes on proposals
- **Learner** — learns the decided value

### Paxos in Two Phases

**Phase 1: Prepare**
```
Proposer → "Prepare(n=5)" → all Acceptors
                                │
Acceptors respond:
  If n > highest promised n:
    → "Promise(n=5, last_accepted=(3, 'X'))"
  Else:
    → "Nack" (I already promised a higher n)
```

**Phase 2: Accept**
```
Proposer (if majority promised):
  → "Accept(n=5, value='X')" → all Acceptors

Acceptors respond:
  If n >= highest promised n:
    → "Accepted"
  Else:
    → "Nack"

If majority accepted → value is chosen!
```

**Why Paxos is hard to understand:**
- Multiple concurrent proposers create complex livelock scenarios
- Multi-Paxos (choosing a sequence of values) requires additional optimizations
- The original paper didn't fully describe the practical implementation

---

## Raft (The Understandable Algorithm)

Raft was designed in 2014 with one explicit goal: be easier to understand than Paxos. It decomposes consensus into three sub-problems: leader election, log replication, and safety.

### Core Idea: Strong Leader

```
All writes go through the leader.
Leader replicates to followers.
Leader tells followers what to commit.
```

```
┌──────────────────────────────────────────────────────┐
│                 Raft Cluster (3 nodes)               │
│                                                      │
│   ┌──────────┐                                       │
│   │  Node A  │ ← LEADER (handles all writes)        │
│   │(term=5)  │                                       │
│   └─────┬────┘                                       │
│         │ AppendEntries (replicate log)              │
│    ┌────┴────────────┐                               │
│    ▼                 ▼                               │
│ ┌──────────┐   ┌──────────┐                          │
│ │  Node B  │   │  Node C  │ ← FOLLOWERS              │
│ │(term=5)  │   │(term=5)  │                          │
│ └──────────┘   └──────────┘                          │
└──────────────────────────────────────────────────────┘
```

### Term Numbers

Raft divides time into **terms** (numbered epochs). Each term begins with an election.

```
Term 1:   Leader A elected  → serves normally
Term 2:   A crashes         → election → Leader B elected
Term 3:   B crashes         → election → Leader C elected
```

Terms are used to detect stale messages — ignore any message from a lower term.

---

### Leader Election Step by Step

**State 1: All followers (initial or after leader failure)**

```
Follower starts an election when:
  No heartbeat received from leader within election timeout (150–300ms)
```

**State 2: Candidate**

```
Node B times out, increments term to 6, votes for itself:
  B → "RequestVote(term=6, candidate=B, lastLogIndex=42, lastLogTerm=5)"
      → to all other nodes
```

**State 3: Voting**

```
Node A (dead), Node C receives RequestVote:
  C checks:
    term 6 > my term 5?  YES → update term to 6
    B's log is at least as current as mine?  YES
  C → "VoteGranted(term=6)"

B receives majority of votes (itself + C = 2/3) → becomes LEADER for term 6
B immediately sends heartbeats to prevent new elections
```

**Split vote:** If two candidates start elections simultaneously, neither may win. Both wait a random timeout and try again. Random timeouts prevent infinite split votes.

---

### Log Replication

```
Write flow:
1. Client sends command to leader: "SET x=5"
2. Leader appends to its log (uncommitted): [... | SET x=5 (term=6, index=43)]
3. Leader sends AppendEntries to all followers
4. Followers append to their logs, respond "ok"
5. Once MAJORITY respond: leader commits (applies to state machine)
6. Leader responds to client: "ok"
7. Next heartbeat tells followers the committed index → they apply too

         Leader Log:        Follower B Log:    Follower C Log:
Index:   [1][2][3][4][5]    [1][2][3][4][5]    [1][2][3][4]
          committed ↑         committed ↑         ↑ lagging

Follower C will catch up on next AppendEntries
```

### Safety Guarantee

A leader is only elected if its log is **at least as up-to-date** as the majority. This ensures:
- Once a value is committed, it survives leader changes
- No committed log entry is ever overwritten

---

## Split-Brain Problem

If the network partitions, you may get two leaders:

```
         Network Partition

   Partition A              Partition B
 ┌────────────────┐       ┌────────────────┐
 │ A(Leader,t=5) │       │ B (starts      │
 │ B (follows A) │       │  election,t=6) │
 └────────────────┘       └────────────────┘

  Partition A: 2 nodes (majority!) → A continues
  Partition B: 1 node → B cannot win election (needs 2/3)
  No split-brain! Raft's quorum requirement prevents it.
```

If the partition is 2+2 (even split), neither side has a majority. Both stall rather than proceed with conflicting decisions.

---

## ZooKeeper and etcd

**ZooKeeper:** Apache's distributed coordination service, uses ZAB (ZooKeeper Atomic Broadcast) — similar to Paxos. Used by Kafka (older versions), HBase, Hadoop.

**etcd:** The key-value store for Kubernetes cluster state. Uses Raft. Stores all cluster config: pods, services, configmaps.

```
kubectl create deployment nginx ...
→ API server writes to etcd
→ etcd replicates via Raft to all etcd nodes (3 or 5 in production)
→ Scheduler/kubelet watch etcd for changes
→ System converges to desired state
```

---

## Performance Implications

Consensus requires at least one round-trip to a majority of nodes before returning a result. This creates a latency floor:

```
Datacenter latency (same region): ~1–2ms per round trip
→ Consensus latency: ~2–4ms per write (2 round trips in Raft)

Cross-datacenter (500km): ~10ms per round trip
→ Consensus latency: ~20–40ms per write

Google Spanner (global): ~14ms average
```

**Optimization: Pipelining**
Don't wait for one round to complete before sending the next. Overlap multiple rounds in flight.

**Optimization: Multi-Paxos / Single Leader**
Elect a leader once, skip Phase 1 for subsequent operations. Only re-elect if the leader fails.

---

## When to Use Consensus

| Use Case | Need Consensus? | Example |
|----------|----------------|---------|
| Storing user data | No (replicas are fine) | PostgreSQL with async replication |
| Leader election | Yes | etcd/ZooKeeper election |
| Distributed lock | Yes | etcd's mutex |
| Config store | Yes | etcd, Consul |
| Strongly consistent DB | Yes | CockroachDB, Spanner, TiKV |
| Financial ledger | Yes | Distributed transaction log |

---

## Key Takeaways

1. **Consensus is fundamentally hard** — the FLP result proves you can't solve it without trade-offs
2. **Paxos was first; Raft was designed to be understandable** — Raft is dominant in new systems
3. **Raft uses strong leadership** — one node handles all writes per term, simplifying reasoning
4. **Leader election uses randomized timeouts** — prevents infinite split votes
5. **A write is committed when a majority acknowledge it** — this is the quorum requirement
6. **Consensus adds latency** — at least one round-trip to a majority before returning
7. **etcd (Kubernetes), ZooKeeper (Kafka), and CockroachDB all use consensus** — it's everywhere at scale

---

*Next: [Event-Driven Architecture →](02-event-driven-architecture.md)*
