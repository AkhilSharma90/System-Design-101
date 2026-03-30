# Databases — SQL vs NoSQL

The database choice is one of the most consequential architectural decisions you'll make. It's also one of the most misunderstood. This article cuts through the hype and gives you a framework for choosing the right database for every use case.

---

## Relational Databases (SQL)

Relational databases store data in tables with rows and columns. Data is structured, relationships are explicit, and a powerful query language (SQL) lets you express complex queries.

```
┌──────────────────────────────────────────────────────┐
│                    users table                       │
├────┬──────────────┬───────────┬──────────────────────┤
│ id │ name         │ email     │ created_at           │
├────┼──────────────┼───────────┼──────────────────────┤
│ 1  │ Alice        │ a@ex.com  │ 2024-01-15           │
│ 2  │ Bob          │ b@ex.com  │ 2024-01-16           │
└────┴──────────────┴───────────┴──────────────────────┘
                        │ (1 user has many orders)
                        ▼
┌──────────────────────────────────────────────────────┐
│                    orders table                      │
├────┬─────────┬────────────┬──────────────────────────┤
│ id │ user_id │ amount     │ status                   │
├────┼─────────┼────────────┼──────────────────────────┤
│ 1  │ 1       │ $50.00     │ shipped                  │
│ 2  │ 1       │ $120.00    │ delivered                │
│ 3  │ 2       │ $30.00     │ pending                  │
└────┴─────────┴────────────┴──────────────────────────┘
```

### ACID Properties

SQL databases guarantee ACID transactions:

| Property | Meaning | Example |
|----------|---------|---------|
| **A**tomicity | All operations succeed, or none do | Transfer $100: debit + credit both happen or neither does |
| **C**onsistency | DB always moves from one valid state to another | Foreign keys are never violated |
| **I**solation | Concurrent transactions don't interfere | Two users booking the last seat can't both succeed |
| **D**urability | Committed data survives crashes | After "commit" returns, data is on disk |

### When to Use SQL

- Complex relationships between entities (joins)
- ACID transactions are required (payments, bookings, inventory)
- Data structure is well-defined and unlikely to change frequently
- Need ad-hoc querying and reporting
- Team is experienced with relational modeling

**Examples:** PostgreSQL, MySQL, SQLite, Oracle, SQL Server

---

## NoSQL Databases

NoSQL ("Not only SQL") databases trade some relational features for flexibility, scale, or performance. There are four main types:

### 1. Key-Value Stores

The simplest model: a hash map. Every value is accessed by a unique key.

```
┌────────────────┬───────────────────────────────────┐
│      Key       │              Value                │
├────────────────┼───────────────────────────────────┤
│ session:abc123 │ {"user_id": 42, "expires": ...}   │
│ user:42:prefs  │ {"theme": "dark", "lang": "en"}   │
│ rate:ip:1.2.3.4│ 47 (request count)                │
└────────────────┴───────────────────────────────────┘
```

**Examples:** Redis, DynamoDB, Memcached
**Best for:** Sessions, caches, real-time leaderboards, rate limiting

### 2. Document Stores

Stores semi-structured documents (usually JSON/BSON). Flexible schema — documents in the same collection can have different fields.

```json
// users collection
{
  "_id": "user_42",
  "name": "Alice",
  "email": "alice@example.com",
  "addresses": [
    {"street": "123 Main St", "city": "NYC", "primary": true},
    {"street": "456 Oak Ave", "city": "LA"}
  ],
  "preferences": {
    "theme": "dark",
    "notifications": ["email", "push"]
  }
}
```

**Examples:** MongoDB, CouchDB, Firestore
**Best for:** Content management, catalogs, user profiles, event logs

### 3. Wide-Column Stores (Column Families)

Data is organized by columns rather than rows. Optimized for writing and reading massive amounts of time-series or sparse data.

```
Row Key         │ Column Family: profile      │ Column Family: events
────────────────┼─────────────────────────────┼─────────────────────────
user#alice      │ name: Alice | age: 30        │ 2024-01: login | purchase
user#bob        │ name: Bob                   │ 2024-01: login
user#charlie    │ name: Charlie | role: admin  │ (no events)
```

**Examples:** Apache Cassandra, Google Bigtable, HBase, ScyllaDB
**Best for:** IoT time-series, click-stream analytics, message storage (WhatsApp uses Cassandra)

### 4. Graph Databases

Stores nodes (entities) and edges (relationships). Optimized for traversing complex relationships.

```
(Alice)──[FRIENDS_WITH]──(Bob)
   │                       │
[LIKES]               [WORKS_AT]
   │                       │
(Post #42)           (Algoroq Inc.)
                           │
                      [HAS_EMPLOYEE]
                           │
                        (Charlie)
```

**Examples:** Neo4j, Amazon Neptune, ArangoDB
**Best for:** Social networks, recommendation engines, fraud detection, knowledge graphs

---

## ACID vs BASE

| Property | ACID (SQL) | BASE (NoSQL) |
|----------|-----------|-------------|
| Consistency | Strong (always) | **BA**sically Available |
| Availability | Sacrificed during partition | **S**oft state |
| Eventual consistency | No | **E**ventual consistency |
| Performance at scale | Harder | Easier |
| Use case | Transactions | High availability, large scale |

---

## Indexing Basics

Indexes dramatically speed up reads at the cost of storage and write overhead.

### Without Index
```
SELECT * FROM users WHERE email = 'alice@example.com';

→ Full table scan: check every row (O(n))
→ 1M rows = 1M comparisons
```

### With Index (B-Tree)
```
B-Tree index on email:
         [m@...]
        /       \
   [alice@]    [zoe@]
   /    \       /   \
[a@]  [e@]  [x@]  [z@]

→ Binary search: O(log n)
→ 1M rows = ~20 comparisons
```

### Index Types

| Type | Best For | Example |
|------|----------|---------|
| B-Tree | Range queries, sorting, equality | `WHERE age > 30` |
| Hash | Exact match only | `WHERE user_id = 42` |
| Full-text | Text search, tokenized queries | `WHERE body LIKE '%system design%'` |
| Composite | Multi-column queries | `WHERE city = 'NYC' AND age > 25` |
| Partial | Subset of rows | `WHERE status = 'active'` (index only active users) |

### Index Trade-offs
- **Reads:** Dramatically faster
- **Writes:** Slower (index must be updated on every INSERT/UPDATE/DELETE)
- **Storage:** Additional disk space
- **Rule of thumb:** Index columns used in WHERE, ORDER BY, JOIN conditions. Don't index every column.

---

## Normalization vs Denormalization

**Normalization:** Organize data to reduce redundancy. Each piece of data lives in one place.
- Pro: No update anomalies, storage efficient
- Con: Requires JOINs, which get expensive at scale

**Denormalization:** Duplicate data for faster reads.
- Pro: No JOINs needed, faster reads
- Con: Data can get out of sync, uses more storage

```
Normalized (good for writes):        Denormalized (good for reads):

users:   id | name                   orders: id | amount | user_name
orders:  id | user_id | amount                    ↑
                 ↑ JOIN needed             embedded, no join needed
         slow at scale                     fast reads, update anomalies
```

---

## Choosing SQL vs NoSQL: Decision Framework

```
Does your data have complex relationships requiring JOINs?
               │
          YES ▼                  NO ▼
      ┌───────────┐       Do you need massive write scale
      │   Use SQL │       or flexible schema?
      └───────────┘              │
                          YES ▼  │   NO ▼
                       What's    │  Use SQL or
                       the data  │  Key-Value
                       pattern?  │
                          │      │
          ┌───────────────┼──────┘
          │               │
     Simple K-V    Document-like    Time-series    Relationships
    (sessions)    (user profiles)   (IoT, logs)   (social graph)
          │               │              │              │
          ▼               ▼              ▼              ▼
      Redis/DynamoDB   MongoDB       Cassandra       Neo4j
```

---

## Real-World Choices

| Company | Use Case | Database | Why |
|---------|----------|----------|-----|
| Instagram | User photos/posts metadata | PostgreSQL | Relational data, ACID |
| Uber | Location data | Cassandra | High write throughput, time-series |
| LinkedIn | Social graph | Graph DB + MySQL | Connection traversal |
| Netflix | User viewing history | Cassandra | Scale, availability |
| Airbnb | Listings, bookings | MySQL + DynamoDB | ACID for bookings, scale for search |
| GitHub | Code, PRs, issues | MySQL | Relational, ACID |
| Twitter | Tweets, timelines | MySQL + Redis + Manhattan | Tweets in SQL, timelines in Redis |

---

## Key Takeaways

1. **SQL is not "old" — it's often the right choice** — ACID transactions, powerful querying, mature tooling
2. **NoSQL trades consistency for scale or flexibility** — know what you're trading
3. **Key-value: use for caches and sessions**; Document: use for flexible schemas; Wide-column: time-series and high writes; Graph: relationship traversal
4. **Index the columns you query** — without indexes, full table scans kill performance
5. **Normalize for writes; denormalize for reads** — you often need both at different layers
6. **Start with SQL** — you can always migrate to NoSQL when you hit specific limits
7. **The data model is your most important early decision** — it's hard to change later

---

*Previous: [Caching Fundamentals ←](07-caching-fundamentals.md) · Next: [Networking Essentials →](09-networking-essentials.md)*
