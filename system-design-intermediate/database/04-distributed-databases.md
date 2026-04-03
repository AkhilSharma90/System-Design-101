---
slug: distributed-databases
title: Distributed Databases
readTime: 20 min
orderIndex: 4
premium: false
---

# Distributed Databases

> Databases that spread data across multiple nodes and regions — combining the scalability of NoSQL with the transactional guarantees of SQL.





Distributed Databases: When One Server Isn't Enough (Google's Spanner Revolution)
🎯 Challenge 1: The Planetary Library Problem
Imagine this scenario: You're building a library system that needs to serve the entire planet - billions of users, petabytes of data, 24/7 availability.

Traditional Single Database (Centralized):
```yaml
┌─────────────────────────────────────┐
│    Library in New York              │
│    - All books stored here          │
│    - Single building                │
│    - One massive catalog            │
└─────────────────────────────────────┘
         ↑          ↑          ↑
    User in NY  User in     User in
    (fast!)     London      Tokyo
                (slow!)     (very slow!)

Problems:
❌ Users in Asia wait 200ms+ (latency)
❌ Single point of failure (building burns = all data lost)
❌ Can't handle billions of users (capacity limit)
❌ Expensive to scale vertically (bigger building)
❌ Backup takes hours (too much data)
```

Distributed Database (Decentralized):
```yaml
New York          London           Tokyo          São Paulo
┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
│ Library   │←──→│ Library   │←──→│ Library   │←──→│ Library   │
│ Shard 1   │    │ Shard 2   │    │ Shard 3   │    │ Shard 4   │
│ Books A-F │    │ Books G-M │    │ Books N-S │    │ Books T-Z │
└───────────┘    └───────────┘    └───────────┘    └───────────┘
      ↑                ↑                ↑                ↑
   Users in        Users in         Users in         Users in
   Americas        Europe           Asia             S. America
   (5ms!)          (5ms!)           (5ms!)           (5ms!)

Coordination:
- Each library has part of the collection (sharding)
- Each library has copies of some books (replication)
- Libraries coordinate to maintain consistency
- Router directs you to right library

Benefits:
✅ Low latency worldwide (serve from nearest location)
✅ Massive scalability (billions of users, petabytes of data)
✅ High availability (one library down, others continue)
✅ No single point of failure (distributed across globe)
✅ Easier to scale horizontally (add more libraries)
```

Pause and think: What if your database could be spread across multiple servers, multiple datacenters, even multiple continents, working as one unified system?

The Answer: Distributed databases split data and processing across multiple nodes while appearing as a single system! It's like:
✅ Data partitioned across multiple servers (horizontal scaling)
✅ Each partition replicated for availability (fault tolerance)
✅ Nodes coordinate using consensus algorithms (consistency)
✅ Queries route to appropriate nodes automatically (transparency)
✅ Scales to planetary size (Google, Amazon, Facebook)

Key Insight: Distributed databases trade simplicity for massive scale and global availability!

🎬 Interactive Exercise: Single vs Distributed Database

Single Database (Monolithic):
```yaml
┌─────────────────────────────────────┐
│     PostgreSQL on Big Server        │
│     - 1TB RAM                       │
│     - 64 CPU cores                  │
│     - 10TB SSD                      │
│     - Handles 10,000 queries/sec    │
│     - Cost: $50,000/month           │
└─────────────────────────────────────┘

Scaling:
- Year 1: Works great! ✓
- Year 2: Need more RAM → Upgrade to 2TB ($70k/month)
- Year 3: Need more CPU → Upgrade to 128 cores ($100k/month)
- Year 4: Hit hardware limits! ❌ Can't scale further

Characteristics:
✅ Simple (one server, ACID transactions)
✅ Consistent (no distributed coordination)
❌ Limited scalability (hardware ceiling)
❌ Single point of failure
❌ Expensive at scale
```

Distributed Database (Horizontal):
```yaml
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Node 1  │ │  Node 2  │ │  Node 3  │ │  Node 4  │ │  Node 5  │
│ 32GB RAM │ │ 32GB RAM │ │ 32GB RAM │ │ 32GB RAM │ │ 32GB RAM │
│ 8 cores  │ │ 8 cores  │ │ 8 cores  │ │ 8 cores  │ │ 8 cores  │
│ 1TB SSD  │ │ 1TB SSD  │ │ 1TB SSD  │ │ 1TB SSD  │ │ 1TB SSD  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
Each handles 2,000 queries/sec = 10,000 total
Cost: $5,000/month each = $25,000/month total (cheaper!)

Scaling:
- Year 1: 5 nodes ✓
- Year 2: Add 5 more nodes → 10 nodes total
- Year 3: Add 10 more nodes → 20 nodes total
- Year 4: Add 20 more nodes → 40 nodes total

Unlimited growth! Just add more commodity servers!

Characteristics:
✅ Unlimited scalability (add more nodes)
✅ High availability (nodes fail independently)
✅ Cost effective (commodity hardware)
❌ Complex (distributed coordination)
❌ Eventual consistency (or coordination overhead)
```

The Trade-off:
```
┌────────────────────────────────────────────────────────┐
│ Aspect        │ Single DB      │ Distributed DB       │
├───────────────┼────────────────┼──────────────────────┤
│ Scalability   │ Vertical (↑)   │ Horizontal (→)       │
│ Cost at scale │ Very expensive │ Cost effective       │
│ Consistency   │ Strong (ACID)  │ Eventual/Tunable     │
│ Complexity    │ Simple         │ Complex              │
│ Latency       │ Low (local)    │ Variable (network)   │
│ Availability  │ Single point   │ No single point      │
│ Max size      │ ~10TB          │ Unlimited (petabytes)│
│ Transactions  │ Easy           │ Hard (distributed)   │
└────────────────────────────────────────────────────────┘
```

Real-world parallel: Single database is like a skyscraper (limited height, expensive). Distributed database is like a city (unlimited growth, add more buildings).

🏗️ Types of Distributed Databases

Type 1: Distributed SQL (NewSQL)
```yaml
Examples: Google Spanner, CockroachDB, YugabyteDB

Architecture:
┌────────────────────────────────────────────────────────┐
│              SQL Query Interface                       │
│         (Looks like PostgreSQL/MySQL)                  │
└────────────┬──────────────┬──────────────┬────────────┘
             │              │              │
        ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
        │ Region 1│    │ Region 2│    │ Region 3│
        │ US-East │    │ EU-West │    │ Asia    │
        │ 3 nodes │    │ 3 nodes │    │ 3 nodes │
        └─────────┘    └─────────┘    └─────────┘

Features:
✅ ACID transactions (even across nodes!)
✅ SQL interface (familiar)
✅ Horizontal scalability
✅ Strong consistency
✅ Multi-region deployment

Use case:
- Global applications needing ACID
- Migration from traditional RDBMS
- Financial systems at scale

Trade-off:
⚠️ Slower than single-node (coordination overhead)
⚠️ More expensive than NoSQL
```

Type 2: Eventually Consistent NoSQL
```yaml
Examples: Cassandra, DynamoDB, Riak

Architecture:
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Node 1  │ │  Node 2  │ │  Node 3  │ │  Node 4  │
│  (Equal) │ │  (Equal) │ │  (Equal) │ │  (Equal) │
│   A-F    │ │   G-M    │ │   N-S    │ │   T-Z    │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
      ↕            ↕            ↕            ↕
All nodes equal (no master!)
Data replicated across nodes
Write to any node

Features:
✅ Extreme scalability (linear)
✅ High availability (no single point)
✅ Low latency (write anywhere)
✅ Tunable consistency
✅ Multi-datacenter

Use case:
- Massive scale (billions of records)
- High write throughput
- Eventual consistency acceptable

Trade-off:
❌ No ACID transactions
❌ Eventual consistency
❌ No JOINs
```

Type 3: Sharded Traditional DB
```yaml
Example: Vitess (MySQL), Citus (PostgreSQL)

Architecture:
         ┌────────────────┐
         │  Query Router  │
         │    (Vitess)    │
         └────────┬───────┘
                  │
    ┌─────────────┼─────────────┬─────────────┐
    ↓             ↓             ↓             ↓
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│MySQL    │  │MySQL    │  │MySQL    │  │MySQL    │
│Shard 1  │  │Shard 2  │  │Shard 3  │  │Shard 4  │
│Users    │  │Users    │  │Users    │  │Users    │
│1-25M    │  │25M-50M  │  │50M-75M  │  │75M-100M │
└─────────┘  └─────────┘  └─────────┘  └─────────┘

Features:
✅ Familiar MySQL/PostgreSQL
✅ Scales horizontally
✅ Can keep existing app code
✅ ACID within shard

Use case:
- Scaling existing MySQL/PostgreSQL
- Need SQL and horizontal scale

Trade-off:
❌ Cross-shard queries slow
❌ Cross-shard transactions hard
❌ Resharding complexity
```

Type 4: Distributed Document Stores
```yaml
Example: MongoDB (sharded cluster)

Architecture:
┌────────────────────────────────────┐
│      Mongos (Query Router)         │
└────────┬──────────┬──────────┬─────┘
         │          │          │
    ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
    │Shard 1 │ │Shard 2 │ │Shard 3 │
    │Replica │ │Replica │ │Replica │
    │  Set   │ │  Set   │ │  Set   │
    └────────┘ └────────┘ └────────┘

Features:
✅ Flexible schema (documents)
✅ Horizontal scaling (sharding)
✅ High availability (replica sets)
✅ Rich query language

Use case:
- Document-oriented data
- Flexible schema needs
- Rapid development

Trade-off:
❌ Eventual consistency (default)
❌ Limited transactions (improving)
```

Real-world parallel:
- Distributed SQL = National chain with strict policies (consistent everywhere)
- NoSQL = Franchises with local autonomy (eventual sync)
- Sharded traditional = Multiple branches of same bank
- Document stores = Flexible filing system across locations

🎮 Decision Game: Which Distributed Database?

Context: You're choosing a database for different use cases.

Scenarios:
A. Global e-commerce platform (need ACID for orders)
B. Social media feed (billions of posts, eventual consistency OK)
C. Real-time analytics (massive data ingestion)
D. Financial trading system (strong consistency critical)
E. IoT sensor data (millions of devices)
F. Multi-tenant SaaS (need isolation)
G. Content management system (flexible schema)
H. Gaming leaderboard (extremely high writes)

Options:
1. Distributed SQL (CockroachDB/Spanner)
2. Eventually Consistent NoSQL (Cassandra/DynamoDB)
3. Sharded PostgreSQL (Citus)
4. MongoDB Sharded Cluster

Answers:
```
A. Global e-commerce → Distributed SQL (1)
   Reason: Need ACID for orders, global scale

B. Social media feed → NoSQL (2)
   Reason: Massive scale, eventual consistency fine

C. Real-time analytics → NoSQL (2)
   Reason: High write throughput needed

D. Financial trading → Distributed SQL (1)
   Reason: Strong consistency critical

E. IoT sensor data → NoSQL (2)
   Reason: Billions of writes, time-series

F. Multi-tenant SaaS → Sharded PostgreSQL (3)
   Reason: Need SQL, isolation per tenant

G. Content management → MongoDB (4)
   Reason: Flexible schema, rich queries

H. Gaming leaderboard → NoSQL (2)
   Reason: Extreme write throughput
```

🚨 Common Misconception: "Distributed = Eventually Consistent... Right?"

You might think: "All distributed databases sacrifice consistency."

The Reality: Modern distributed databases offer strong consistency!

Understanding Consistency Models:

Eventual Consistency:
```
Time: 10:00:00
Node 1 (US): Write X=5
Node 2 (EU): Still has X=0

Time: 10:00:01 (1 second later)
Node 1 (US): X=5
Node 2 (EU): X=5 (synced!)

During that 1 second:
- Read from Node 1: X=5 ✓
- Read from Node 2: X=0 ✗ (stale!)

Eventually consistent = Will be consistent eventually
```

Strong Consistency (Linearizability):
```
Time: 10:00:00
Node 1 (US): Write X=5
  ↓
Consensus protocol (Raft/Paxos)
  ↓
Wait for majority of nodes to confirm
  ↓
Node 1: X=5 ✓
Node 2: X=5 ✓
Node 3: X=5 ✓
  ↓
Return success to client

Now reading from ANY node: X=5 ✓

Strongly consistent = Always see latest value
```

Google Spanner Example:
```sql
-- Spanner provides ACID across continents!

-- Create table (automatically distributed)
CREATE TABLE users (
  user_id INT64 NOT NULL,
  email STRING(256),
  balance NUMERIC
) PRIMARY KEY (user_id);

-- ACID transaction across multiple rows/nodes
BEGIN TRANSACTION;
  UPDATE users SET balance = balance - 100 WHERE user_id = 123;
  UPDATE users SET balance = balance + 100 WHERE user_id = 456;
COMMIT;

-- Even if user 123 and 456 are on different continents!
-- Spanner guarantees ACID using:
-- 1. Two-phase commit
-- 2. TrueTime (atomic clocks for ordering)
-- 3. Paxos for consensus

Result: Distributed AND strongly consistent! ✓
```

How It's Possible:
```
Spanner's TrueTime:
- Atomic clocks in every datacenter
- GPS-synchronized time
- Knows uncertainty bound (±7ms)
- Uses time to order transactions globally

Example:
Transaction 1: timestamp = 10:00:00.003 ± 0.007
Transaction 2: timestamp = 10:00:00.020 ± 0.007

Spanner knows T2 happened after T1 (no overlap)
Can safely order them!

Trade-off:
✅ Strong consistency globally
❌ Higher latency (coordination)
❌ More expensive (atomic clocks, coordination)
```

Real-world parallel: Strong consistency is like a global conference call (everyone hears same thing, but takes time to coordinate). Eventual consistency is like email (everyone gets message, but at different times).

⚡ Distributed Consensus: How Nodes Agree

The Challenge:
```
3 nodes need to agree on a value:

Node 1 proposes: X=5
Node 2 proposes: X=10
Node 3 proposes: X=15

How do they agree on ONE value?
What if Node 2 is down?
What if network is slow?
```

Raft Consensus Algorithm:
```
Step 1: Leader Election
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Node 1  │    │ Node 2  │    │ Node 3  │
│Candidate│    │Candidate│    │Candidate│
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     ├─ Vote for me! ──────────────→
     │              │              │
     ←─── Vote for Node 1 ─────────┤
     │              │              │
     └──────── Becomes Leader! ────┘

Step 2: Log Replication
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Leader  │    │Follower │    │Follower │
│ Node 1  │    │ Node 2  │    │ Node 3  │
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     ├─ Replicate X=5 ─────────────→
     │              │              │
     ←─── ACK ──────┤              │
     │              ←───── ACK ────┤
     │              │              │
     └── Commit (majority confirmed!)

Step 3: Client Gets Response
Client: "SET X=5"
  ↓
Leader: "Replicated to majority, committed!" ✓
  ↓
Client: "Success!"

Key: Need majority (quorum) to commit
3 nodes: Need 2 to agree
5 nodes: Need 3 to agree
```

Handling Failures:
```
Scenario: Leader fails during replication

Before:
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Leader  │    │Follower │    │Follower │
│ Node 1  │    │ Node 2  │    │ Node 3  │
│ Log: X=5│    │ Log: X=5│    │ Log: X=5│
└────┬────┘    └─────────┘    └─────────┘
     │
   💥 Crashes!

After:
┌─────────┐    ┌─────────┐
│ Leader  │    │Follower │
│ Node 2  │←───│ Node 3  │
│ Log: X=5│    │ Log: X=5│
└─────────┘    └─────────┘

New election!
Node 2 becomes leader (has all committed data)
System continues! ✓

No data lost! (X=5 was committed to majority)
```

CockroachDB Example:
```sql
-- CockroachDB uses Raft for consensus

-- Create table (automatically replicated with Raft)
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL
);

-- Insert (replicated via Raft)
INSERT INTO accounts VALUES (1, 1000);

Behind the scenes:
1. Leader receives INSERT
2. Proposes to followers via Raft
3. Waits for majority (quorum)
4. Commits when majority confirms
5. Returns success to client

-- Even if one node fails:
-- System continues (has quorum)
-- Data is safe (majority has it)

-- Check replication status
SHOW RANGES FROM TABLE accounts;

 range_id | replicas | lease_holder
----------+----------+--------------
    42    | {1,2,3}  |      1
```

Real-world parallel: Raft is like a committee vote. Majority must agree before decision is official. If some members absent, majority of present members still sufficient.

🔧 Distributed Transactions: The Hard Problem

The Two-Phase Commit (2PC):
```
Scenario: Transfer money across 2 databases

Phase 1: Prepare
Coordinator: "Can you commit this transaction?"
   ↓
Database 1 (US): "Yes, I can" ✓
Database 2 (EU): "Yes, I can" ✓
   ↓
All voted YES? Proceed to phase 2

Phase 2: Commit
Coordinator: "Everyone commit!"
   ↓
Database 1: Commits ✓
Database 2: Commits ✓
   ↓
Coordinator: "Transaction complete!" ✓

If anyone votes NO:
Coordinator: "Everyone rollback!"
All databases rollback
Transaction aborted
```

The Problem with 2PC:
```
Scenario: Coordinator crashes after "prepare" but before "commit"

Phase 1: Prepare
Coordinator: "Can you commit?"
Database 1: "Yes" (waiting...)
Database 2: "Yes" (waiting...)
   ↓
Coordinator crashes! 💥
   ↓
Databases stuck waiting!
Can't commit (no confirmation)
Can't rollback (might have been commit)

BLOCKED! This is why 2PC is problematic in practice.
```

Modern Solutions:

Saga Pattern:
```
Instead of locking, use compensating transactions:

Step 1: Debit Account A (succeeds)
Step 2: Credit Account B (fails!)
Step 3: Compensate: Credit Account A back

Series of local transactions
Each can be reversed
No distributed locking!

Use case: Microservices, long-running transactions
```

Spanner's Solution:
```
Uses TrueTime + Paxos:

1. Assign globally unique timestamp
2. Use Paxos for consensus (better than 2PC)
3. Wait out uncertainty (ensure timestamp is globally unique)
4. Commit

Result: True distributed ACID transactions! ✓
Cost: Higher latency (coordination + wait)
```

Real-world parallel: 2PC is like getting signatures from multiple people. If courier (coordinator) lost in transit, everyone waits. Saga is like a reversible process - can undo if something fails.

💡 Sharding in Distributed Databases

Automatic Sharding (CockroachDB):
```
-- Create table
CREATE TABLE users (
  id UUID PRIMARY KEY,
  name STRING,
  email STRING
);

-- CockroachDB automatically:
-- 1. Splits data into ranges (64MB each)
-- 2. Distributes ranges across nodes
-- 3. Replicates each range (3 copies default)
-- 4. Rebalances automatically

-- Check how data is distributed
SHOW RANGES FROM TABLE users;

range_id | start_key | end_key | replicas | lease_holder
---------+-----------+---------+----------+-------------
   100   | /1        | /1000   | {1,2,3}  |      1
   101   | /1000     | /2000   | {2,3,4}  |      2
   102   | /2000     | /3000   | {3,4,5}  |      3

-- Application doesn't need to know about sharding!
-- Just query normally:
SELECT * FROM users WHERE id = '...';
-- CockroachDB routes to correct range automatically
```

Manual Sharding (MongoDB):
```javascript
// Enable sharding on database
sh.enableSharding("myapp")

// Shard collection by user_id (hash-based)
sh.shardCollection("myapp.users", { user_id: "hashed" })

// MongoDB automatically:
// - Distributes data across shards
// - Routes queries to correct shard
// - Balances data across shards

// Check shard distribution
db.users.getShardDistribution()

// Output:
// Shard shard1: 2500000 docs, 50% of data
// Shard shard2: 2500000 docs, 50% of data
```

Real-world parallel: Automatic sharding is like a valet parking service (handles distribution automatically). Manual sharding is like parking lot sections (you decide where to park).

🌐 Multi-Region Deployment Patterns

Pattern 1: Primary in One Region (Read Replicas Everywhere)
```
       US-East (Primary)           Europe (Replica)         Asia (Replica)
┌──────────────────────┐      ┌──────────────────┐    ┌──────────────────┐
│ Read/Write           │─────→│ Read Only        │───→│ Read Only        │
│ 3 nodes (Raft)       │      │ 3 nodes          │    │ 3 nodes          │
└──────────────────────┘      └──────────────────┘    └──────────────────┘

Characteristics:
✅ Strong consistency (single primary)
✅ Fast reads globally (local replicas)
❌ Slow writes globally (must go to primary)

Use case: Read-heavy workload, writes can tolerate latency
```

Pattern 2: Regional Primaries (Multi-Region Primary-Primary)
```
       US-East                    Europe                    Asia
┌──────────────────┐      ┌──────────────────┐    ┌──────────────────┐
│ Primary          │←────→│ Primary          │←──→│ Primary          │
│ Data: US users   │      │ Data: EU users   │    │ Data: APAC users │
└──────────────────┘      └──────────────────┘    └──────────────────┘

Characteristics:
✅ Fast writes locally (write to local primary)
✅ Fast reads locally (read from local)
⚠️ Cross-region transactions slow
⚠️ Eventual consistency across regions

Use case: Geo-partitioned data (US users in US, EU users in EU)
```

Pattern 3: Global Consensus (Spanner-style)
```
       US-East                    Europe                    Asia
┌──────────────────┐      ┌──────────────────┐    ┌──────────────────┐
│ 3 nodes          │←────→│ 3 nodes          │←──→│ 3 nodes          │
│ Participate in   │      │ Participate in   │    │ Participate in   │
│ global consensus │      │ global consensus │    │ global consensus │
└──────────────────┘      └──────────────────┘    └──────────────────┘

All regions participate in Paxos/Raft
Any region can initiate transactions
Strong consistency globally

Characteristics:
✅ Strong consistency everywhere
✅ ACID transactions globally
❌ Higher latency (global consensus)

Use case: Financial systems, need global ACID
```

Real-world parallel:
- Pattern 1 = Headquarters with branch offices
- Pattern 2 = Regional headquarters (autonomous)
- Pattern 3 = Federation (all regions vote on decisions)

💡 Final Synthesis Challenge: The Global Corporation

Complete this comparison:
"A single database is like a company in one building. A distributed database is like..."

Your answer should include:
- Data distribution
- Consistency guarantees
- Scalability approach
- Global availability

Take a moment to formulate your complete answer...

The Complete Picture:
A distributed database is like a multinational corporation with offices worldwide:

✅ **Headquarters + Branches** (Primary-Replica): Central office makes decisions, branches execute locally
✅ **Regional Autonomy** (Multi-Primary): Each region operates independently, syncs periodically
✅ **Federation** (Consensus): All offices vote on major decisions (Raft/Paxos)
✅ **Departments** (Sharding): Each office handles specific customer segments
✅ **Redundancy** (Replication): Multiple offices have same information
✅ **Global Scale**: Can serve billions worldwide from nearest location
✅ **Coordination**: Offices communicate to maintain consistency

**Benefits:**
1. **Unlimited Scale** - Add more nodes = more capacity (horizontal)
2. **Global Availability** - Serve users from nearest datacenter (low latency)
3. **Fault Tolerance** - One datacenter fails, others continue
4. **Geo-Distribution** - Data in multiple locations (compliance, disaster recovery)

**Trade-offs:**
- Complexity (distributed coordination)
- Consistency challenges (or latency cost)
- More expensive operations (network calls)
- Harder to reason about (distributed state)

**Real-world examples:**
- **Spanner** (Google): Global ACID transactions for AdWords
- **DynamoDB** (Amazon): Massive scale for e-commerce
- **Cassandra** (Netflix): Billions of viewing events
- **CockroachDB**: Cloud-native distributed SQL
- **MongoDB Atlas**: Global document database

**When to use:**
- Data > 1TB and growing fast
- Need global low latency
- Single server can't handle load
- High availability critical
- Can accept complexity trade-off

**When NOT to use:**
- Data < 100GB
- Single region sufficient
- Team lacks distributed systems expertise
- Simple application

Distributed databases transform single-server limits into planetary-scale systems!

🎯 Quick Recap: Test Your Understanding
Without looking back, can you explain:
1. How do distributed databases differ from sharded traditional databases?
2. What is Raft consensus and why is it needed?
3. How does Google Spanner achieve strong consistency globally?
4. When should you choose distributed SQL vs NoSQL?

Mental check: If you can design a distributed database architecture, you understand distributed databases!

🚀 Your Next Learning Adventure
Now that you understand distributed databases, explore:

Advanced Topics:
- Vector clocks and conflict resolution
- CRDTs (Conflict-Free Replicated Data Types)
- Distributed tracing and observability
- Jepsen testing (distributed systems testing)

Distributed Databases:
- CockroachDB deep dive
- YugabyteDB architecture
- TiDB (distributed MySQL)
- FoundationDB (key-value store)

Consensus Algorithms:
- Raft visualization
- Paxos algorithm
- Byzantine fault tolerance
- Blockchain consensus

Real-World Case Studies:
- How Uber built Schemaless
- Discord's database scaling
- Dropbox's migration to distributed storage
- How Stripe handles distributed transactions


---

## Key Takeaways

1. **Distributed databases spread data across multiple nodes and regions** — enabling horizontal scalability beyond a single machine
2. **CAP theorem constrains distributed database design** — you must choose between consistency and availability during network partitions
3. **Consensus protocols (Raft, Paxos) keep distributed nodes in agreement** — at the cost of latency for cross-node coordination
4. **CockroachDB, Spanner, and YugabyteDB offer distributed SQL** — combining the scalability of NoSQL with SQL's transactional guarantees
