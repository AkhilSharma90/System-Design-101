# Caching Fundamentals

Phil Karlton famously said: *"There are only two hard things in Computer Science: cache invalidation and naming things."* This article covers both the power and the pitfalls of caching.

---

## What is Caching?

A cache is a fast, temporary storage layer that sits between your application and slower data sources. By storing frequently accessed data closer to where it's needed, caching dramatically reduces latency and offloads pressure from databases.

```
Without Cache:                    With Cache:

Request → App → Database          Request → App → Cache (hit!)
                ↑                                  ↑
             20ms query                         1ms lookup

If cache miss: App → Cache → Database → Cache stores result
```

**The core idea:** Compute or fetch something once, store the result, serve it many times.

---

## The Caching Hierarchy

Caching exists at every layer of your stack:

```
Fastest │  CPU L1 Cache         ~0.5 ns   (on die, kilobytes)
        │  CPU L2/L3 Cache      ~7 ns     (on die, megabytes)
        │  Application Cache    ~1 μs     (in-process, e.g. HashMap)
        │  Distributed Cache    ~1 ms     (Redis, Memcached)
        │  CDN Cache            ~10 ms    (edge servers worldwide)
        │  Database Query Cache ~100ms    (MySQL query cache)
Slowest │  Disk / Object Store  ~10ms+    (SSD, S3)
```

In system design, "caching" usually refers to **distributed caching** (Redis/Memcached) and **CDN caching**.

---

## Cache-Aside (Lazy Loading)

The most widely used pattern. The application manages the cache directly.

```
Read flow:
┌─────────┐        ┌─────────┐        ┌──────────┐
│   App   │───1.──▶│  Cache  │        │    DB    │
│         │◀──2.───│  (hit)  │        │          │
└─────────┘  data  └─────────┘        └──────────┘

On cache MISS:
┌─────────┐        ┌─────────┐        ┌──────────┐
│   App   │───1.──▶│  Cache  │        │    DB    │
│         │◀──2.───│  (miss) │        │          │
│         │───3.──────────────────────▶│          │
│         │◀──4.───────────────────────│          │
│         │───5.──▶│  store  │        │          │
└─────────┘        └─────────┘        └──────────┘
```

**Pros:** Only caches what's actually requested; cache failure doesn't break reads
**Cons:** Cache miss causes 3 trips (cache → DB → cache); initial cold start is slow
**Best for:** Read-heavy workloads with unpredictable access patterns

---

## Read-Through Cache

The cache sits in front of the database. On a miss, the cache fetches from DB automatically.

```
┌─────────┐        ┌──────────────┐        ┌──────────┐
│   App   │───────▶│    Cache     │──miss──▶│    DB    │
│         │◀───────│(auto-fetches)│◀────────│          │
└─────────┘        └──────────────┘         └──────────┘
```

**Pros:** App code is simpler — just read from cache
**Cons:** Cache library must know how to talk to your database
**Best for:** Consistent access patterns where cache-aside is too verbose

---

## Write-Through Cache

Every write goes to the cache AND the database synchronously.

```
┌─────────┐        ┌─────────┐        ┌──────────┐
│   App   │──write▶│  Cache  │──write▶│    DB    │
│         │◀─────OK─│         │◀────OK──│          │
└─────────┘        └─────────┘        └──────────┘
```

**Pros:** Cache is always consistent with DB; no stale data on reads
**Cons:** Write latency is higher (must write both); cache fills with data that may never be read
**Best for:** Systems where consistency is critical and read latency matters

---

## Write-Behind (Write-Back) Cache

Writes go to the cache first, then asynchronously flush to the database.

```
┌─────────┐        ┌─────────┐
│   App   │──write▶│  Cache  │──(async, batched)──▶ DB
│         │◀─────OK─│  (fast) │
└─────────┘        └─────────┘
             ↑
         Returns immediately!
```

**Pros:** Extremely fast writes; good for write-heavy workloads
**Cons:** Risk of data loss if cache fails before flushing; complex to implement correctly
**Best for:** High-frequency writes where brief data loss is acceptable (metrics, counters, logs)

---

## Write-Around Cache

Writes go directly to the database, bypassing the cache. Cache is only populated on reads.

```
┌─────────┐                             ┌──────────┐
│   App   │──────────write─────────────▶│    DB    │
│         │──read──▶Cache──(miss)──read▶│          │
└─────────┘                             └──────────┘
```

**Pros:** Cache doesn't fill up with data that's written once and rarely read
**Cons:** First read after write always hits the database
**Best for:** Data that's written frequently but read infrequently (log files, large media uploads)

---

## Strategy Comparison

| Strategy | Write Speed | Read Speed | Consistency | Data Loss Risk | Best For |
|----------|-------------|------------|-------------|----------------|----------|
| Cache-Aside | Normal | Fast (warm) | Good | None | General read-heavy |
| Read-Through | Normal | Fast | Good | None | Simplified read logic |
| Write-Through | Slower | Fast | Excellent | None | Consistency-critical |
| Write-Behind | Fastest | Fast | Eventual | On cache failure | Write-heavy |
| Write-Around | Normal | Slow (cold) | Excellent | None | Write-once data |

---

## Cache Eviction Policies

When the cache is full, something must be removed to make room. The eviction policy determines what gets removed.

### LRU — Least Recently Used
Evicts the item that hasn't been accessed for the longest time.

```
Access order: A → B → C → D → A → E (cache full, evict B)
Cache: [E, A, D, C]    ← B was least recently used
```

**Best for:** General-purpose caching — works well when recency predicts future access.

### LFU — Least Frequently Used
Evicts the item accessed the fewest times.

```
Access counts: A=10, B=1, C=5, D=2
Eviction: B (count=1) is removed first
```

**Best for:** Long-lived caches where popularity matters more than recency.

### FIFO — First In, First Out
Evicts the oldest item regardless of access frequency.

**Best for:** Simple queue-like use cases; rarely optimal for caching.

### TTL — Time To Live
Items expire after a set duration, regardless of access patterns.

```
cache.set("user:123", userData, ttl=3600)  # expires in 1 hour
```

**Best for:** Data with known staleness tolerance (e.g., session tokens, rate limit counters).

---

## Cache Invalidation

Cache invalidation is the hard part. Stale data in cache causes bugs that are difficult to reproduce and debug.

### TTL-Based (Expiration)
Set a time limit on cache entries. Simple but imprecise — data may be stale for the full TTL duration.

### Event-Based Invalidation
When data changes, explicitly delete or update the cache entry.

```python
def update_user(user_id, data):
    db.update(user_id, data)           # Update database
    cache.delete(f"user:{user_id}")    # Invalidate cache entry
```

### Cache-Aside Race Condition
A subtle bug when multiple processes update cache simultaneously:

```
Thread 1: Read DB (gets value=1)
Thread 2: Update DB to value=2
Thread 2: Delete cache entry
Thread 1: Write value=1 to cache  ← STALE! Thread 2's delete was overwritten
```

**Solution:** Use cache.set with a short TTL even after invalidation, or use version numbers/ETags.

---

## Distributed Caching

When a single cache server isn't enough:

```
┌──────────────────────────────────────────────────────┐
│              Distributed Cache Cluster               │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Redis 1  │  │ Redis 2  │  │ Redis 3  │           │
│  │ Keys A-F │  │ Keys G-M │  │ Keys N-Z │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                      │
│  Consistent hashing determines which node holds each │
│  key. Client routes directly to correct node.        │
└──────────────────────────────────────────────────────┘
```

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Strings, lists, sets, sorted sets, hashes, streams | Strings only |
| Persistence | Yes (RDB + AOF) | No |
| Replication | Yes (leader-follower) | No |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Multi-threading | Single-threaded (v6: I/O threads) | Multi-threaded |
| Memory efficiency | Slightly less | Slightly more |
| **Use when** | Rich data types, persistence, complex operations | Pure caching, maximum simplicity |

---

## Cache Stampede (Thundering Herd)

When a popular cache entry expires, many requests simultaneously hit the database.

```
t=0: "top_products" cache entry expires
t=1: 1,000 concurrent requests find cache miss
t=2: All 1,000 requests query the database simultaneously
t=3: Database collapses under load
```

### Solutions

**1. Mutex/Lock:** Only one request fetches from DB; others wait.
```python
if not cache.get(key):
    with lock(key):
        if not cache.get(key):   # double-check after acquiring lock
            value = db.fetch()
            cache.set(key, value)
```

**2. Probabilistic early expiration:** Before TTL expires, proactively refresh.

**3. Background refresh:** A background job refreshes the cache before expiry.

**4. Stale-while-revalidate:** Serve stale content while refreshing in the background.

---

## When NOT to Cache

Caching is not always the answer:

- **Frequently changing data** — if data changes every second, caching it adds complexity with no benefit
- **Unique per-request data** — user-specific real-time data (bank balance) should rarely be cached
- **Small datasets** — if your whole dataset fits in memory, cache everything or nothing
- **Debugging becomes hard** — caching bugs are notoriously difficult to reproduce

---

## Key Takeaways

1. **Cache-aside is the most common pattern** — flexible, works with any data store
2. **Write-through ensures consistency** — at the cost of write latency
3. **Write-behind is the fastest for writes** — but has data loss risk on failure
4. **LRU is the most practical eviction policy** — simple and generally effective
5. **Cache invalidation is the hard part** — TTL is simple but imprecise; event-based is accurate but complex
6. **Cache stampede will ruin your day** — use locks or background refresh for hot keys
7. **Redis does more than Memcached** — prefer Redis unless you need raw throughput at massive scale

---

*Previous: [Load Balancing ←](06-load-balancing.md) · Next: [Databases — SQL vs NoSQL →](08-databases-sql-vs-nosql.md)*
