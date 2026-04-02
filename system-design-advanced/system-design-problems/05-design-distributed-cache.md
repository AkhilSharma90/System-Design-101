---
slug: design-distributed-cache
title: Design a Distributed Cache
readTime: 25 min
orderIndex: 5
premium: false
---

# Design a Distributed Cache

Design a distributed in-memory cache system like Redis Cluster or Memcached — capable of handling millions of operations per second with sub-millisecond latency.

---

## Step 1: Requirements

### Functional Requirements
- `get(key)` → returns value or null
- `set(key, value, ttl)` → stores value with optional TTL
- `delete(key)` → removes key
- High throughput: 1 million operations/second
- Low latency: < 1ms p99

### Non-Functional Requirements
- High availability: nodes fail without downtime
- Horizontal scalability: add nodes to increase capacity
- Durability: optional (caches are typically ephemeral)
- Consistency: eventual consistency acceptable for most use cases

---

## Step 2: Estimation

```
Operations:  1M ops/sec total (70% reads, 30% writes)
Memory:      100 GB total cache size
Nodes:       Assuming 10 GB usable memory per node → 10 nodes initially
Network:     Avg value size 1 KB → 1M × 1 KB = 1 GB/sec bandwidth (distributed across nodes)
```

---

## Step 3: Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Distributed Cache Cluster                  │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Cache    │  │ Cache    │  │ Cache    │  │ Cache    │   │
│  │ Node 1   │  │ Node 2   │  │ Node 3   │  │ Node 4   │   │
│  │          │  │          │  │          │  │          │   │
│  │ Primary  │  │ Primary  │  │ Primary  │  │ Primary  │   │
│  │ Slots    │  │ Slots    │  │ Slots    │  │ Slots    │   │
│  │ 0-4095   │  │4096-8191 │  │8192-12287│  │12288-    │   │
│  │          │  │          │  │          │  │  16383   │   │
│  │+Replica  │  │+Replica  │  │+Replica  │  │+Replica  │   │
│  │(Node 3's │  │(Node 4's │  │(Node 1's │  │(Node 2's │   │
│  │ slots)   │  │  slots)  │  │  slots)  │  │  slots)  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                             │
│  Each key hashed to one of 16,384 slots (Redis Cluster)     │
│  Each node owns ~4,096 slots as primary + ~4,096 as replica │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 4: Data Partitioning

Use consistent hashing to distribute keys across nodes. (See [Consistent Hashing](../02-intermediate-system-design/02-consistent-hashing.md) for details.)

**Redis Cluster's approach — hash slots:**

```
slot = CRC16(key) % 16384

Key "user:42"  → CRC16("user:42") % 16384 = 7843 → Node 2
Key "order:99" → CRC16("order:99") % 16384 = 1205 → Node 1

Cluster config stored in each node (gossip protocol):
  Node 1 → slots 0-4095
  Node 2 → slots 4096-8191
  ...

Client caches this mapping locally → routes directly to correct node
```

**Hash tags for co-location:**
```
"user:{42}:profile" and "user:{42}:settings" → same slot (key within {})
→ Guarantees they land on same node → multi-key operations work
```

---

## Step 5: Eviction Policies

When the cache is full, what gets removed?

```
Memory: 10 GB, all used.
New write arrives: must evict something.

noeviction:  Return error. Client must handle.
allkeys-lru: Remove least recently used key across ALL keys.
volatile-lru: Remove LRU key from those WITH TTL set.
allkeys-lfu:  Remove least frequently used key.
allkeys-random: Remove a random key.
volatile-ttl:  Remove key with soonest expiration.
```

**LRU Implementation (with O(1) eviction):**

```
Doubly Linked List + HashMap

HashMap: key → node in DLL
DLL:     most_recent ← [node] ↔ [node] ↔ [node] → least_recent

Access key: move its node to head (O(1))
Eviction:   remove tail node (O(1))
Lookup:     HashMap for O(1) access to any node
```

**Production recommendation:** `allkeys-lru` for general caches. `volatile-lru` if some keys should never be evicted (set no TTL on them).

---

## Step 6: Replication for High Availability

Each primary node has a replica:

```javascript
┌──────────────────────────────────────────────────────────┐
│  Normal operation (Node 1 = Primary, Node 5 = Replica)  │
│                                                          │
│  Client ──write──▶ Node 1 (Primary)                     │
│                         │ async replication               │
│                         └──────────────▶ Node 5 (Replica)│
│                                                          │
│  Client ──read──▶ Node 1 (Primary)  [or Node 5 Replica]  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  Node 1 fails:                                           │
│                                                          │
│  Cluster detects Node 1 down (multiple nodes confirm)   │
│  Node 5 promoted to Primary for slots 0-4095            │
│  Cluster config updated via gossip                       │
│  New replica assigned for Node 5's slots                 │
│                                                          │
│  Failover time: ~10-15 seconds                          │
└──────────────────────────────────────────────────────────┘
```

**Trade-off:** Replication is asynchronous → if Node 1 crashes before replicating its last writes, those writes are lost. Acceptable for a cache (cache miss just means a DB read).

---

## Step 7: Write Policies

### Write-Through
```
App ──write──▶ Cache ──write──▶ DB
               both succeed before returning

Pro:  Cache always consistent with DB
Con:  Higher write latency (two writes)
```

### Write-Behind (Write-Back)
```
App ──write──▶ Cache ──return "ok"
                 │ (async, batched)
                 └──write──▶ DB

Pro:  Very fast writes
Con:  Risk of data loss if cache node fails before flushing to DB
```

### Write-Around
```
App ──write──▶ DB (directly, skip cache)
App ──read──▶ Cache MISS ──▶ DB ──▶ Cache

Pro:  Cache not polluted with write-heavy data that isn't read
Con:  Cache cold for newly written data
```

---

## Step 8: Hot Key Problem

A single key receives disproportionate traffic:

```yaml
Problem: "product:featured_item" accessed 1M times/sec
         All on the same cache node (consistent hashing by key)
         → Single node overwhelmed while others are idle

Solutions:

1. Key Splitting:
   "product:featured_item:shard0"  → Node 1
   "product:featured_item:shard1"  → Node 3  ← distribute reads
   "product:featured_item:shard2"  → Node 7

   Read: pick random shard → spread load
   Write: write to all shards (or invalidate all on update)

2. Local Cache:
   Each app server maintains a small in-process cache (HashMap)
   "product:featured_item" → cached locally for 10 seconds
   → 1M reads → mostly served from local memory, no cache node traffic

3. Read Replicas per slot:
   Add extra replicas for hot shards, route reads to them
```

---

## Step 9: Cache Operations at Scale

### Pipeline and Batch Operations
```
Without pipelining: 100 GET commands → 100 round trips → 100 × 1ms = 100ms
With pipelining:    100 GET commands → 1 round trip   → 1ms

Redis PIPELINE:
  PIPELINE
    GET key1
    GET key2
    ...
    GET key100
  EXECUTE (send all at once, receive all at once)
```

### Atomic Operations
```
Increment counter atomically:
  INCR rate_limit:user:42   ← atomic, thread-safe across all clients

Check-and-set:
  WATCH key
  value = GET key
  MULTI
    SET key new_value
  EXEC   ← only executes if key hasn't changed since WATCH
           (optimistic concurrency control)
```

---

## Step 10: Adding/Removing Nodes (Online Resharding)

When adding a new cache node, some slots must be migrated:

```
Before (4 nodes, 4096 slots each):
  Node 1: slots 0-4095
  Node 2: slots 4096-8191
  Node 3: slots 8192-12287
  Node 4: slots 12288-16383

Add Node 5: each existing node donates ~1/5 of its slots
  Node 1: slots 0-3276          (moved 819 slots to Node 5)
  Node 2: slots 4096-7372       (moved 819 slots to Node 5)
  Node 3: slots 8192-11468
  Node 4: slots 12288-15564
  Node 5: slots 3277-4095, 7373-8191, 11469-12287, 15565-16383

Migration process (MIGRATE command):
  Move slot X from Node 1 to Node 5:
  1. SETSLOT X MIGRATING (mark slot as migrating in Node 1)
  2. SETSLOT X IMPORTING (mark slot as importing in Node 5)
  3. MIGRATE key by key from Node 1 to Node 5
  4. SETSLOT X NODE node5_id (atomic cutover)

During migration: requests for migrating keys work correctly
  → If key not yet migrated: served from old node
  → If key migrated: client gets redirect (ASK) to new node
```

---

## Key Takeaways

1. **Consistent hashing distributes keys across nodes** — Redis Cluster uses 16,384 hash slots
2. **LRU eviction is the most practical policy** — use `allkeys-lru` for general caches
3. **Each primary has a replica** — async replication enables fast failover at the cost of potential lost writes
4. **Hot keys require explicit handling** — key splitting or local application-level caching
5. **Pipeline batch operations to reduce round trips** — 100 commands in 1 round trip vs 100
6. **Online resharding enables zero-downtime scaling** — slots migrated one by one while serving traffic
7. **Write-through for consistency, write-behind for performance** — choose based on durability requirements

---
