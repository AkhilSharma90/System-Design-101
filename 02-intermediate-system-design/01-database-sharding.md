# Database Sharding

At some point, every successful product hits a wall: a single database server can no longer handle the load. Sharding is the technique that breaks this ceiling — splitting your data across multiple database servers to scale writes, storage, and read throughput beyond what any single machine can provide.

---

## The Problem: Single Database Limits

A single PostgreSQL or MySQL instance has hard limits:

```
┌─────────────────────────────────────────────────────────┐
│                 Single DB Bottlenecks                   │
│                                                         │
│  Storage:   Typically capped at ~50–100 TB per server   │
│  Writes:    ~10,000–50,000 TPS max (hardware bound)     │
│  CPU:       Query complexity multiplies with data size  │
│  RAM:       Working set must fit for good performance   │
│                                                         │
│  Read replicas solve read scale — but NOT write scale.  │
│  Sharding solves write scale.                           │
└─────────────────────────────────────────────────────────┘
```

---

## Vertical vs Horizontal Partitioning

### Vertical Partitioning (Different Tables, Different Servers)

Split your database by feature domain.

```
Before:                        After:
┌──────────────────────┐       ┌──────────┐  ┌──────────┐  ┌──────────┐
│   Single Database    │       │  User DB │  │ Order DB │  │Product DB│
│  - users             │ ──▶   │ - users  │  │ - orders │  │-products │
│  - orders            │       │ - sessions│  │ - items  │  │-inventory│
│  - products          │       └──────────┘  └──────────┘  └──────────┘
│  - sessions          │
└──────────────────────┘       Each DB on its own server!
```

**Pros:** Simple, maps naturally to microservices
**Cons:** Cross-table JOINs become expensive network calls; doesn't help if one table is huge

### Horizontal Partitioning (Sharding)

Split a single large table across multiple databases by rows.

```
Before:                    After (3 shards by user_id):
users table (500M rows)

┌────────────────────┐    ┌────────────┐ ┌────────────┐ ┌────────────┐
│ user_id │  name    │    │  Shard 0   │ │  Shard 1   │ │  Shard 2   │
│ 1       │  Alice   │    │ user_id%3=0│ │ user_id%3=1│ │ user_id%3=2│
│ 2       │  Bob     │    │            │ │            │ │            │
│ 3       │  Charlie │    │ 3, 6, 9... │ │ 1, 4, 7... │ │ 2, 5, 8... │
│ ...     │  ...     │    └────────────┘ └────────────┘ └────────────┘
│ 500M    │  ...     │
└────────────────────┘    Each shard ≈ 167M rows
```

---

## Sharding Strategies

### Range-Based Sharding

Distribute rows based on a range of values in the shard key.

```
user_id 1–10M      → Shard A
user_id 10M–20M    → Shard B
user_id 20M–30M    → Shard C
```

**Pros:** Range queries are efficient (all data for a date range is on one shard)
**Cons:** Uneven distribution — early users may be on one shard that gets all the historical queries; hotspots possible

### Hash-Based Sharding

Apply a hash function to the shard key, then modulo by number of shards.

```
shard_id = hash(user_id) % num_shards

hash(1) % 3 = 0  → Shard A
hash(2) % 3 = 1  → Shard B
hash(3) % 3 = 2  → Shard C
```

**Pros:** Uniform distribution, no hotspots
**Cons:** Range queries span all shards; resharding requires moving ~all data (consistent hashing solves this — next article)

### Directory-Based Sharding

A lookup table maps each key to its shard.

```
┌──────────────────────┐      ┌────────────────────────────┐
│   Shard Directory    │      │        Shards               │
│                      │      │                             │
│  user 1–1000  → S1   │ ────▶│  Shard 1: user 1–1000      │
│  user 1001–X  → S2   │ ────▶│  Shard 2: user 1001+       │
│  VIP users    → S3   │ ────▶│  Shard 3: VIP accounts     │
└──────────────────────┘      └────────────────────────────┘
```

**Pros:** Maximum flexibility — shard logic can be anything
**Cons:** The directory itself becomes a bottleneck and single point of failure

### Geographic Sharding

Route data based on user geography for latency and compliance.

```
EU users   → EU Shard (Frankfurt)
US users   → US Shard (Virginia)
APAC users → APAC Shard (Singapore)
```

**Pros:** Low latency for users, GDPR compliance (EU data stays in EU)
**Cons:** Uneven user distribution by region; cross-region queries are expensive

---

## Choosing a Shard Key

The shard key decision is the most important sharding decision. A bad shard key creates hotspots and cross-shard queries.

### What Makes a Good Shard Key

| Property | Why It Matters |
|----------|---------------|
| High cardinality | Many distinct values → even distribution |
| Low correlation with time | Prevents new-data hotspot on most-recent shard |
| Matches query patterns | Most queries should target one shard |
| Immutable | Changing a key requires moving the data |

### Examples

| Scenario | Good Shard Key | Bad Shard Key |
|----------|---------------|---------------|
| Multi-tenant SaaS | `tenant_id` | `created_at` (time-based hotspot) |
| Social media posts | `user_id` | `post_id` (celebrity hotspot) |
| E-commerce orders | `user_id` or `order_id` hash | `status` (few values, huge hotspot) |
| Messaging | `conversation_id` | `sender_id` (celebrity problem) |

---

## The Cross-Shard Query Problem

Sharding breaks JOINs. When data lives on different shards, queries that need to combine it get complex.

```
Query: "Find all orders with their user names"

Without sharding:
  SELECT o.*, u.name FROM orders o JOIN users u ON o.user_id = u.id
  → Single query, fast

With sharding (users on Shard A, orders on Shard B):
  → Query Shard A for users: [user_1, user_2, ...]
  → Query Shard B for orders: [order_1, order_2, ...]
  → Merge in application code
  → Slow, complex, and doesn't scale with data size
```

**Solutions:**
- Shard related data together (keep users and their orders on the same shard using `user_id` as the shard key for both)
- Denormalize: embed user data in orders at write time
- Use a separate analytics database (replicate all data to a warehouse for complex queries)

---

## Resharding

When a shard grows too large or traffic patterns change, you need to reshard. This is painful with modulo hashing.

```
Before (3 shards, hash % 3):        After (4 shards, hash % 4):
  Key "alice" → shard 2               Key "alice" → shard 1  ← MOVED
  Key "bob"   → shard 0               Key "bob"   → shard 2  ← MOVED
  Key "carol" → shard 1               Key "carol" → shard 0  ← MOVED
                                       ~75% of all keys move!
```

**Consistent hashing** (next article) solves this — only ~1/N keys move when adding a shard.

---

## Hotspot Problem

A "hot" shard receives disproportionate traffic. Common causes:

- **Celebrity accounts:** A famous user's data causes one shard to handle 1000x traffic
- **Time-based hotspot:** All new writes go to the "today" shard
- **Popular items:** A viral product concentrates all orders to one shard

### Mitigation Strategies

```
Approach 1 — Add qualifier to shard key:
  Instead of: shard_key = user_id
  Use:        shard_key = user_id + random_suffix(1-10)
  → Spreads one user's data across 10 shards

Approach 2 — Dedicated shard for hot keys:
  celebrity_user_id → "VIP shard" (oversized, many replicas)
  normal users     → standard shards

Approach 3 — Cache the hot data:
  Popular items → Redis cache in front of the shard
```

---

## Real-World Sharding Examples

### Instagram (PostgreSQL Sharding)
Instagram sharded PostgreSQL across thousands of servers. Their shard key strategy embedded the shard ID in every object's 64-bit ID:

```
64-bit ID structure:
┌─────────────────┬─────────────┬──────────────────┐
│  41 bits         │  13 bits    │  10 bits          │
│  timestamp       │  shard_id   │  sequence number  │
└─────────────────┴─────────────┴──────────────────┘

Given any object ID, you can instantly compute which shard it lives on.
```

### Discord (Cassandra)
Discord uses Cassandra (which is natively sharded using consistent hashing). Their `messages` table is partitioned by `channel_id` — all messages in a channel are on the same shard, making reads fast.

```
Partition key: (channel_id, bucket)  ← bucket = month, limits partition size
Clustering key: message_id DESC      ← newest messages first
```

---

## Alternatives to Sharding

Before sharding, consider these alternatives — each is simpler:

| Alternative | When to Use |
|------------|-------------|
| Add read replicas | Read-heavy workload; writes aren't the bottleneck |
| Vertical scaling | Upgrade the database server — often buys years |
| Better indexing | Slow queries, not volume, are the problem |
| Caching (Redis) | Reads are the bottleneck; data fits in memory |
| Denormalization | Complex JOINs are slow; reduce them with duplication |
| NewSQL (CockroachDB, Spanner) | Need SQL + horizontal scale; handles sharding internally |

---

## Key Takeaways

1. **Sharding splits data horizontally across multiple databases** — it scales writes beyond one machine
2. **Hash-based sharding distributes evenly** — but resharding is painful without consistent hashing
3. **Range-based sharding enables efficient range queries** — but watch for hotspots
4. **Choose your shard key carefully** — it determines your query patterns and scaling ceiling
5. **Cross-shard JOINs are expensive** — shard related data together to avoid them
6. **The hotspot problem is real** — add randomness to shard keys for celebrity/viral data
7. **Exhaust simpler options first** — sharding adds significant operational complexity

---

*Next: [Consistent Hashing →](02-consistent-hashing.md)*
