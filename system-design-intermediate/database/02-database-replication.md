---
slug: database-replication
title: Database Replication
readTime: 20 min
orderIndex: 3
premium: false
---

# Database Replication




Database Replication: Keeping Multiple Copies in Sync (Your Data's Backup Dancers)
🎯 Challenge 1: The Important Document Problem
Imagine this scenario: You have a critically important document - your company's only copy of customer contracts.

Single Copy (No Replication):
```
┌─────────────────────────┐
│  Main Office            │
│  Customer Contracts     │
│  (Only Copy!)           │
└─────────────────────────┘

Problems:
❌ Office burns down → All data LOST! 🔥
❌ Office closed for maintenance → Can't access data
❌ Too many people trying to read → Long wait times
❌ Office in New York → Slow access from Tokyo
❌ Single point of failure
```

With Replication (Multiple Copies):
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ New York Office │────→│ London Office   │────→│ Tokyo Office    │
│ (Primary Copy)  │     │ (Replica Copy)  │     │ (Replica Copy)  │
│ Read + Write    │     │ Read Only       │     │ Read Only       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
    Changes sync automatically across all locations!

Benefits:
✅ NY burns down → Still have London and Tokyo! (High Availability)
✅ Tokyo users read locally → Fast! (Low Latency)
✅ Multiple people read simultaneously → No waiting (Scalability)
✅ Maintenance on NY → Switch to London (Zero Downtime)
✅ Automatic backup (Disaster Recovery)
```

Pause and think: What if every change to your database was automatically copied to multiple servers around the world?

The Answer: Database Replication creates and maintains multiple copies of your data! It's like:
✅ Primary server handles writes (source of truth)
✅ Replica servers copy all changes (automatic sync)
✅ Read from any replica (distribute load)
✅ Failover to replica if primary fails (high availability)
✅ Replicas can be in different locations (disaster recovery)

Key Insight: Replication trades storage space and complexity for availability, performance, and durability!

🎬 Interactive Exercise: Replication vs Backup

You might think: "Isn't replication just fancy backup?"

The Key Differences:

Backup (Snapshot):
```
Timeline:
Monday 00:00    → Full backup created
  ↓
Tuesday 00:00   → Daily backup created
  ↓
Wednesday 00:00 → Daily backup created
Wednesday 14:30 → Database crashes! 💥

Recovery:
- Restore from Wednesday 00:00 backup
- Lost 14.5 hours of data! ❌
- Restoration takes 2 hours (downtime)
- Users can't access database during restore

Use case: Disaster recovery (long-term protection)
```

Replication (Real-time):
```
Timeline:
Primary: Write X=5
  ↓ (milliseconds)
Replica: Has X=5 ✓

Primary: Write Y=10
  ↓ (milliseconds)
Replica: Has Y=10 ✓

Primary: Crashes! 💥
  ↓
Promote Replica to Primary (30 seconds)
  ↓
System continues operating!

Recovery:
- Automatic failover to replica
- Lost 0 data! ✓ (or only last few seconds)
- Downtime: 30 seconds (brief!)
- Users barely notice

Use case: High availability (real-time protection)
```

The Comparison:
```
┌──────────────────────────────────────────────────────┐
│ Aspect        │ Backup         │ Replication        │
├───────────────┼────────────────┼────────────────────┤
│ Purpose       │ Recovery       │ Availability       │
│ Frequency     │ Hours/daily    │ Continuous (ms)    │
│ Data loss     │ Hours of data  │ Seconds (or none)  │
│ Downtime      │ Hours          │ Seconds            │
│ Access        │ Offline copy   │ Online, readable   │
│ Storage       │ Separate       │ Always synced      │
│ Use case      │ Disaster       │ High availability  │
└──────────────────────────────────────────────────────┘

Best practice: Use BOTH!
- Replication for HA (real-time)
- Backups for point-in-time recovery (protect against human error)
```

Real-world parallel: Backup is like photos in a safe (for long-term recovery). Replication is like having an understudy in a play (immediate continuity).

🏗️ Types of Replication

Type 1: Primary-Replica (Master-Slave)
```
Most common pattern:

        ┌─────────────┐
        │   Primary   │ ← All writes go here
        │  (Master)   │
        └──────┬──────┘
               │
        ┌──────┼──────┬──────┐
        ↓      ↓      ↓      ↓
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │Replica 1│ │Replica 2│ │Replica 3│
  │(Slave)  │ │(Slave)  │ │(Slave)  │
  └─────────┘ └─────────┘ └─────────┘
  Read only    Read only    Read only

Write flow:
1. Client writes to Primary
2. Primary updates its data
3. Primary sends changes to all Replicas
4. Replicas apply changes

Read flow:
- Can read from Primary or any Replica
- Usually read from Replicas (offload Primary)

Pros:
✅ Simple to understand
✅ Scales reads (add more replicas)
✅ High availability (failover to replica)
✅ Disaster recovery

Cons:
❌ Single write bottleneck (only primary accepts writes)
❌ Replication lag (replicas slightly behind)
❌ Manual failover needed (or automatic with tools)
```

Type 2: Primary-Primary (Multi-Master)
```
Both nodes accept writes:

┌─────────────┐         ┌─────────────┐
│  Primary 1  │←───────→│  Primary 2  │
│  (Master)   │         │  (Master)   │
│  Read/Write │         │  Read/Write │
└─────────────┘         └─────────────┘
       ↑                       ↑
   Writes from            Writes from
   App Server 1           App Server 2

Changes sync bidirectionally!

Write flow:
1. Client writes to Primary 1: X=5
2. Primary 1 sends change to Primary 2
3. Primary 2 applies: X=5

Simultaneously:
1. Client writes to Primary 2: Y=10
2. Primary 2 sends change to Primary 1
3. Primary 1 applies: Y=10

Pros:
✅ Scales writes (multiple writers)
✅ High availability (automatic failover)
✅ Low latency (write to nearest primary)

Cons:
❌ Conflict resolution needed!
❌ Complex setup
❌ Risk of split-brain
❌ Can lose data in conflicts
```

Type 3: Chain Replication
```
Data flows through chain:

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│Primary  │───→│Replica 1│───→│Replica 2│───→│Replica 3│
└─────────┘    └─────────┘    └─────────┘    └─────────┘
Write here                                     Read here
                                              (most up-to-date)

Write flow:
1. Client writes to Primary (head of chain)
2. Primary sends to Replica 1
3. Replica 1 sends to Replica 2
4. Replica 2 sends to Replica 3 (tail)
5. Tail acknowledges back to client

Read flow:
- Read from tail (guaranteed to have all committed data)

Pros:
✅ Strong consistency (tail has all committed data)
✅ Lower load on primary (doesn't send to all replicas)

Cons:
❌ Higher latency (goes through chain)
❌ If middle node fails, chain breaks
```

Real-world parallel:
- Primary-Replica = Teacher (primary) and students taking notes (replicas)
- Primary-Primary = Two teachers co-teaching (both can write on board)
- Chain = Assembly line (item passes through each station)

🎮 Decision Game: Replication Conflict!

Context: You have Primary-Primary replication. What happens with concurrent writes?

Scenario:
```
Time: 10:00:00
Initial state: account_balance = $1000

Primary 1 (New York):
User A withdraws $100
account_balance = $1000 - $100 = $900

Primary 2 (London):
User B withdraws $200
account_balance = $1000 - $200 = $800

Both happen simultaneously!

Primary 1 syncs to Primary 2:
Primary 2 sees: Local=$800, Incoming=$900

CONFLICT! Which is correct?
```

Options for Conflict Resolution:
1. Last Write Wins (timestamp-based)
2. First Write Wins (sequence-based)
3. Merge Values (application logic)
4. Reject Second Write (pessimistic locking)

Think about: What's safest for a bank account?

Analysis:
```
Option 1: Last Write Wins
- If NY write has later timestamp: balance = $900
- If London write has later timestamp: balance = $800
- Problem: One withdrawal is LOST! ❌
- Use case: Shopping cart (merge items), user preferences

Option 2: First Write Wins
- Whichever reached database first wins
- Other is rejected
- Problem: Rejection might be arbitrary
- Use case: Ticket booking (first come first served)

Option 3: Merge Values
- $1000 - $100 - $200 = $700 ✓ Correct!
- Requires application logic to understand operations
- Problem: Complex to implement
- Use case: Collaborative editing, CRDT

Option 4: Reject Second Write
- Use distributed locks
- Only one write at a time
- Problem: Defeats purpose of multi-primary!
- Use case: When conflicts are unacceptable

For bank account: Use Primary-Replica, not Primary-Primary!
Money is too important for automatic conflict resolution!
```

The Golden Rule:
```
Use Primary-Primary only when:
✅ Conflicts are rare
✅ Conflicts are mergeable
✅ You have conflict resolution strategy
✅ Availability > perfect consistency

Examples:
✅ CouchDB (document edits)
✅ Riak (key-value store)
✅ Multi-region writes in DynamoDB

Avoid Primary-Primary for:
❌ Financial transactions
❌ Inventory (can't sell same item twice)
❌ Anything requiring strong consistency
```

Real-world parallel: Multi-primary is like two people editing the same Google Doc. Works great until both edit the same sentence simultaneously (conflict!).

⚡ Replication Methods: How Data Gets Copied

Method 1: Statement-Based Replication
```
What gets replicated: SQL statements

Primary:
UPDATE accounts SET balance = balance - 100 WHERE id = 123;
     ↓
Replica receives:
UPDATE accounts SET balance = balance - 100 WHERE id = 123;
     ↓
Replica executes same statement

Pros:
✅ Compact (just send SQL text)
✅ Easy to understand

Cons:
❌ Non-deterministic functions break it!
   NOW() returns different time on replica
   RAND() returns different random number
   UUID() returns different UUID
❌ Triggers behave differently
❌ Auto-increment might diverge
```

Method 2: Row-Based Replication (Logical)
```
What gets replicated: Actual data changes

Primary:
UPDATE accounts SET balance = balance - 100 WHERE id = 123;
Executes: balance changes from 1000 to 900
     ↓
Replica receives:
Row 123: balance changed from 1000 to 900
     ↓
Replica applies exact change

Pros:
✅ Deterministic (exact data changes)
✅ Works with any function (NOW(), RAND(), etc.)
✅ Can replicate across different DB versions

Cons:
❌ Larger data transfer (full row changes)
❌ UPDATE affecting many rows = large replication stream
```

Method 3: Physical Replication (Binary)
```
What gets replicated: Actual disk blocks

Primary:
Writes to disk block 1234
     ↓
Replica receives:
Binary copy of disk block 1234
     ↓
Replica writes exact same bytes

Pros:
✅ Exact replica (byte-for-byte identical)
✅ Efficient (no parsing SQL)

Cons:
❌ Replicas must be same OS/hardware
❌ Can't replicate across DB versions
❌ Can't do partial replication
```

PostgreSQL Example:
```sql
-- Configure primary for replication

-- postgresql.conf
wal_level = replica
max_wal_senders = 3
wal_keep_size = '1GB'

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secret';

-- pg_hba.conf
host replication replicator 10.0.0.0/24 md5

-- On replica, create recovery.conf
primary_conninfo = 'host=primary.db port=5432 user=replicator password=secret'

-- Start replica
pg_basebackup -h primary.db -D /var/lib/postgresql/data -U replicator -P

-- Check replication status on primary
SELECT * FROM pg_stat_replication;

 application_name | state     | sync_state | replay_lag
------------------+-----------+------------+------------
 replica1         | streaming | async      | 00:00:00.5
 replica2         | streaming | sync       | 00:00:00.1
```

Real-world parallel:
- Statement-based = Telling someone "make a sandwich" (might make different sandwich)
- Row-based = Sending exact sandwich recipe with amounts (reproducible)
- Physical = Sending exact 3D-printed sandwich (identical copy)

🚨 Common Misconception: "Replicas Are Always Consistent... Right?"

You might think: "Replicas have exact same data as primary."

The Reality: There's always replication lag!

Understanding Replication Lag:
```
Time: 10:00:00.000
Primary: INSERT INTO users (id, name) VALUES (123, 'Alice');
  ↓
Time: 10:00:00.050 (50ms later)
Replica 1: Has the insert ✓

Time: 10:00:00.200 (200ms later)
Replica 2: Has the insert ✓

Time: 10:00:00.500 (500ms later)
Replica 3: Has the insert ✓

During this lag period:
- Read from Primary: Sees Alice ✓
- Read from Replica 1 (10:00:00.030): Doesn't see Alice ✗
- Read from Replica 2 (10:00:00.100): Doesn't see Alice ✗
```

Real-World Scenario:
```
User registers account:
1. App writes to Primary: INSERT user 'Alice'
2. App redirects to profile page
3. App reads from Replica: "User not found" ✗
   (Replica hasn't caught up yet!)
4. User sees error: "Registration failed"
5. User registers again (duplicate!)

This is a classic replication lag bug!
```

Solutions:

Solution 1: Read-After-Write Consistency
```python
def create_user(name):
    # Write to primary
    primary_db.execute(
        "INSERT INTO users (name) VALUES (%s)",
        (name,)
    )
    user_id = primary_db.lastrowid

    # Read from PRIMARY (not replica) for immediate reads
    user = primary_db.execute(
        "SELECT * FROM users WHERE id = %s",
        (user_id,)
    ).fetchone()

    return user

# Or: Route user's own reads to primary for N seconds
session.set('read_from_primary_until', time.time() + 5)
```

Solution 2: Monotonic Reads
```python
# Track last seen replication position
def read_user(user_id, session):
    last_seen_position = session.get('last_replication_position', 0)

    # Find replica that's caught up to what user last saw
    for replica in replicas:
        if replica.replication_position >= last_seen_position:
            user = replica.execute(
                "SELECT * FROM users WHERE id = %s",
                (user_id,)
            ).fetchone()

            # Remember this position
            session.set('last_replication_position',
                       replica.replication_position)

            return user

    # All replicas lagging? Read from primary
    return primary_db.execute(...).fetchone()
```

Solution 3: Synchronous Replication
```sql
-- PostgreSQL synchronous replication
-- Primary waits for at least 1 replica before returning

-- postgresql.conf
synchronous_commit = on
synchronous_standby_names = 'replica1,replica2'

-- Now writes wait for replica acknowledgment:
INSERT INTO users (name) VALUES ('Alice');
  ↓
Primary writes locally
  ↓
Primary sends to replicas
  ↓
Wait for replica1 or replica2 to confirm
  ↓
Return success to client

Trade-off:
✅ No replication lag for confirmed writes
❌ Slower writes (must wait for network round-trip)
❌ If all replicas down, primary blocks writes!
```

Monitoring Replication Lag:
```sql
-- PostgreSQL: Check lag on primary
SELECT
    client_addr,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    replay_lag
FROM pg_stat_replication;

-- MySQL: Check lag on replica
SHOW SLAVE STATUS\G

Seconds_Behind_Master: 2  ← Lag in seconds

-- Alert if lag > 10 seconds (indicates problem)
```

Real-world parallel: Replication lag is like news propagation. Breaking news happens (primary), but it takes time for newspapers (replicas) to print and distribute.

🔧 Failover: When Primary Dies

Automatic Failover Flow:
```
Normal operation:
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Primary │────→│Replica 1│────→│Replica 2│
└────┬────┘     └─────────┘     └─────────┘
     │
 App writes here

Primary crashes! 💥
     ↓
Monitoring detects failure (10 seconds)
     ↓
Promote Replica 1 to Primary
     ↓
┌─────────┐     ┌─────────┐
│Replica 1│────→│Replica 2│
│  (NEW   │     └─────────┘
│ Primary)│
└────┬────┘
     │
App writes here now

Total downtime: ~30 seconds
```

Failover Tools:

PostgreSQL with Patroni:
```yaml
# patroni.yml
scope: postgres-cluster
namespace: /db/
name: node1

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

  initdb:
  - encoding: UTF8
  - data-checksums

postgresql:
  use_pg_rewind: true
  parameters:
    max_connections: 100

# Patroni automatically:
# 1. Monitors primary health
# 2. Promotes replica if primary fails
# 3. Updates routing
# 4. No manual intervention needed!
```

MySQL with Orchestrator:
```sql
-- Orchestrator monitors topology
-- Automatically promotes replica on failure

-- Check cluster status
orchestrator-client -c topology -i primary.db:3306

primary.db:3306
  ├─ replica1.db:3306 (0s lag)
  ├─ replica2.db:3306 (0s lag)
  └─ replica3.db:3306 (1s lag)

-- Primary fails
-- Orchestrator promotes replica1 automatically

replica1.db:3306 (promoted to primary)
  ├─ replica2.db:3306 (now replicates from replica1)
  └─ replica3.db:3306 (now replicates from replica1)
```

Application-Level Failover:
```python
from sqlalchemy import create_engine, event
from sqlalchemy.pool import Pool

class ReplicationAwareEngine:
    def __init__(self):
        self.primary = create_engine('postgresql://primary:5432/db')
        self.replicas = [
            create_engine('postgresql://replica1:5432/db'),
            create_engine('postgresql://replica2:5432/db'),
        ]
        self.current_replica = 0

    def write(self, query, params):
        try:
            return self.primary.execute(query, params)
        except Exception as e:
            # Primary is down! Try to promote replica
            logger.error(f"Primary failed: {e}")
            self.promote_replica()
            raise

    def read(self, query, params):
        # Round-robin across replicas
        replica = self.replicas[self.current_replica]
        self.current_replica = (self.current_replica + 1) % len(self.replicas)

        try:
            return replica.execute(query, params)
        except Exception:
            # Replica failed, try next one
            return self.read(query, params)

    def promote_replica(self):
        # In real system, would:
        # 1. Stop all writes
        # 2. Choose most up-to-date replica
        # 3. Promote it to primary
        # 4. Reconfigure other replicas
        # 5. Update app config
        pass
```

Real-world parallel: Failover is like vice president becoming president. There's a clear succession plan, happens automatically, minimal disruption.

💡 Multi-Region Replication

Global Setup:
```
        US-East                 Europe              Asia-Pacific
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│ Primary (RW)      │  │ Replica (R)       │  │ Replica (R)       │
│ Users in Americas │←→│ Users in Europe   │←→│ Users in APAC     │
└───────────────────┘  └───────────────────┘  └───────────────────┘
     Low latency          Low latency           Low latency
     for US users         for EU users          for APAC users

Write flow:
1. User in Europe writes
2. Write goes to US Primary (high latency ~100ms)
3. Primary replicates to Europe and APAC

Read flow:
1. User in Europe reads
2. Read from local European replica (low latency ~10ms)

Trade-off:
✅ Fast reads everywhere
❌ Slow writes (must go to primary region)
```

Multi-Primary Multi-Region:
```
        US-East                 Europe              Asia-Pacific
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│ Primary (RW)      │  │ Primary (RW)      │  │ Primary (RW)      │
│ US users write    │←→│ EU users write    │←→│ APAC users write  │
└───────────────────┘  └───────────────────┘  └───────────────────┘

Benefits:
✅ Fast writes everywhere (write to local primary)
✅ Fast reads everywhere (read from local)

Challenges:
⚠️ Conflict resolution (concurrent writes in different regions)
⚠️ Consistency (eventual consistency only)
⚠️ Complex setup

Examples:
- CouchDB with multi-master replication
- DynamoDB global tables
- MongoDB with zone-aware writes
```

Real-world parallel: Multi-region replication is like having offices in multiple countries. Can serve customers locally (fast), but coordinating between offices takes time.

💡 Final Synthesis Challenge: The Backup Dancers

Complete this comparison:
"A database without replication is like a solo performer. A replicated database is like..."

Your answer should include:
- High availability
- Read scalability
- Disaster recovery
- Trade-offs (lag, complexity)

Take a moment to formulate your complete answer...

The Complete Picture:
A replicated database is like a performer with backup dancers:

✅ **Primary (lead dancer)**: Handles all choreography changes (writes)
✅ **Replicas (backup dancers)**: Learn and perform the choreography (reads)
✅ **If lead falls**: Backup steps up immediately (automatic failover)
✅ **Multiple performers**: Show can continue in multiple cities (scalability)
✅ **Slight delay**: Backups learn choreography with small lag (replication lag)
✅ **Global reach**: Perform simultaneously worldwide (multi-region)
✅ **Disaster recovery**: Show goes on even if one venue closes

**Replication provides:**
1. **High Availability** - System stays up when primary fails
2. **Read Scalability** - Distribute read load across replicas
3. **Disaster Recovery** - Data survives datacenter failures
4. **Low Latency** - Serve users from nearest replica

**Trade-offs:**
- Replication lag (eventual consistency)
- More complex setup and monitoring
- Storage cost (multiple copies)
- Network bandwidth for replication

**Real-world examples:**
- Netflix: Read replicas in every region for low latency
- GitHub: Replicas for high availability
- Instagram: Replicas for read scaling
- Stripe: Synchronous replication for financial data

Replication transforms fragile single-server databases into resilient, globally distributed systems!

🎯 Quick Recap: Test Your Understanding
Without looking back, can you explain:
1. What's the difference between replication and backup?
2. How does Primary-Replica replication work?
3. What is replication lag and why does it happen?
4. When should you use synchronous vs asynchronous replication?

Mental check: If you can design a replication strategy, you understand database replication!

🚀 Your Next Learning Adventure
Now that you understand replication, explore:

Advanced Replication:
- Semi-synchronous replication
- Group replication (MySQL)
- Logical replication and CDC
- Cross-datacenter replication

Replication Tools:
- Patroni (PostgreSQL HA)
- MySQL InnoDB Cluster
- MongoDB Replica Sets
- Redis Sentinel

Related Concepts:
- Database sharding
- Distributed consensus (Raft, Paxos)
- Multi-region architectures
- Read-write splitting

Real-World Case Studies:
- How GitHub handles database failover
- Netflix's multi-region database strategy
- Stripe's approach to financial data replication
- Facebook's MySQL replication at scale
