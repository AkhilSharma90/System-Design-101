---
slug: master-slave-archiecture
title: Master Slave Archiecture
readTime: 20 min
orderIndex: 2
premium: false
---

# Master-Slave Architecture






Master-Slave Architecture: The Classic Database Pattern (Now Called Primary-Replica)
🎯 Challenge 1: The Single Chef Kitchen Problem
Imagine this scenario: You run a restaurant with one chef who does everything.

Single Chef (No Delegation):
```yaml
┌──────────────────────────────────────┐
│          Single Chef                 │
│                                      │
│  - Takes orders from customers       │
│  - Prepares all dishes              │
│  - Answers recipe questions         │
│  - Trains new staff                 │
│  - Everything!                      │
└──────────────────────────────────────┘

Problems:
❌ Chef overwhelmed (handles everything)
❌ Long wait times (bottleneck)
❌ Chef sick = Restaurant closed! (single point of failure)
❌ Can't scale (only one chef can fit in kitchen)
❌ Customers far away wait longer (latency)
```

Head Chef + Sous Chefs (Master-Slave Pattern):
```yaml
          ┌─────────────────────┐
          │    Head Chef        │ ← Creates new recipes (WRITES)
          │    (Master/Primary) │    Updates menu
          └──────────┬──────────┘    Trains others
                     │
         ┌───────────┼───────────┬──────────┐
         ↓           ↓           ↓          ↓
    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Sous    │ │ Sous    │ │ Sous    │ │ Sous    │
    │ Chef 1  │ │ Chef 2  │ │ Chef 3  │ │ Chef 4  │
    │ (Slave) │ │ (Slave) │ │ (Slave) │ │ (Slave) │
    └─────────┘ └─────────┘ └─────────┘ └─────────┘
    Answers      Answers      Answers      Answers
    customer     customer     customer     customer
    questions    questions    questions    questions
    (READS)      (READS)      (READS)      (READS)

Flow:
1. Head Chef creates new recipe → Teaches all Sous Chefs
2. Customers ask about recipes → Any Sous Chef can answer
3. Menu change needed → Only Head Chef can do it
4. Head Chef busy/sick → Promote a Sous Chef to Head Chef

Benefits:
✅ Head Chef focuses on important work (writes)
✅ Customers get fast answers (reads from nearest Sous Chef)
✅ Many Sous Chefs = handle many customers (scalability)
✅ Head Chef leaves → Sous Chef promoted (high availability)
✅ Sous Chefs in different locations (global distribution)
```

Pause and think: What if your database had one server for writes and many servers for reads?

The Answer: Master-Slave architecture (now called Primary-Replica) separates write and read responsibilities! It's like:
✅ Master/Primary = Single source of truth (handles all writes)
✅ Slaves/Replicas = Copies that follow master (handle reads)
✅ Data flows one direction (master → slaves)
✅ Automatic promotion on failure (slave becomes master)
✅ Read scalability (add more slaves = more read capacity)

**Terminology Note:** The industry is transitioning from "Master-Slave" to "Primary-Replica" for inclusivity. They mean the same thing. Modern documentation uses Primary-Replica.

Key Insight: This pattern trades write scalability for read scalability - you can't scale writes, but you can scale reads infinitely!

🎬 Interactive Exercise: Write vs Read Workloads

Understanding the 90/10 Rule:
```sql
Most applications are read-heavy:

Typical Web Application:
┌────────────────────────────────┐
│ 90% Reads (SELECT)             │ ← Can go to replicas!
│ - View user profile            │
│ - Browse products              │
│ - Check order status           │
│ - View comments                │
│ - Load dashboard               │
├────────────────────────────────┤
│ 10% Writes (INSERT/UPDATE)     │ ← Must go to primary!
│ - Create account               │
│ - Update profile               │
│ - Place order                  │
│ - Post comment                 │
└────────────────────────────────┘

With Primary-Replica:
- 10% traffic → Primary (manageable!)
- 90% traffic → Distributed across replicas (scales!)
```

Single Database (No Replication):
```
All Traffic → Single Database

With 1000 requests/second:
├─ 900 reads/sec
└─ 100 writes/sec

Single database handles:
1000 total requests/sec ⚠️ (struggling)

Scaling option: Bigger server 💰 (expensive!)
```

Primary-Replica (With Replication):
```
        ┌──────────────┐
        │   Primary    │ ← 100 writes/sec (10%)
        └──────┬───────┘
               │
    ┌──────────┼──────────┬──────────┐
    ↓          ↓          ↓          ↓
┌────────┐┌────────┐┌────────┐┌────────┐
│Replica││Replica││Replica││Replica│
│   1   ││   2   ││   3   ││   4   │
└────────┘└────────┘└────────┘└────────┘
  225        225        225        225
reads/sec reads/sec reads/sec reads/sec

Primary handles: 100 writes/sec (easy! ✓)
Each replica handles: 225 reads/sec (easy! ✓)
Total capacity: 1000 requests/sec (distributed!)

Need more capacity? Add more replicas!
```

The Math:
```
Without Replication:
1 server handling 1000 req/sec = stressed
Solution: Upgrade to massive server = expensive

With Replication (1 primary + 4 replicas):
Primary: 100 req/sec (10% of total)
Each replica: 225 req/sec (22.5% of reads)
5 commodity servers = cheaper + more scalable

To double capacity:
- Add 4 more replicas (8 total)
- Each now handles ~112 reads/sec
- Linear scaling! ✓
```

Real-world parallel: Primary-Replica is like a company with one CEO (makes decisions) and many managers (answer questions). CEO isn't overwhelmed because managers handle most inquiries.

🏗️ How Primary-Replica Works (The Details)

The Write Path:
```sql
Step 1: Client sends write
Client: UPDATE users SET email='new@email.com' WHERE id=123;
   ↓
Step 2: Primary receives and executes
Primary: Executes UPDATE on local database
Primary: Data changed: user 123 email updated ✓
   ↓
Step 3: Primary logs change
Primary: Writes to binlog/WAL (write-ahead log)
Entry: "user 123 email changed to new@email.com"
   ↓
Step 4: Primary sends to replicas
Primary → Replica 1: "Apply this change"
Primary → Replica 2: "Apply this change"
Primary → Replica 3: "Apply this change"
   ↓
Step 5: Replicas apply change
Replica 1: Executes UPDATE ✓
Replica 2: Executes UPDATE ✓
Replica 3: Executes UPDATE ✓
   ↓
Step 6: (Optional) Replicas acknowledge
Replica 1 → Primary: "Done!"
Replica 2 → Primary: "Done!"
Replica 3 → Primary: "Done!"
   ↓
Step 7: Primary returns to client
Primary → Client: "UPDATE successful!" ✓

Total time: ~50-200ms (depending on synchronous vs async)
```

The Read Path:
```sql
Option A: Read from Primary
Client: SELECT * FROM users WHERE id=123;
   ↓
Primary: Returns data ✓
   ↓
Client: Gets response (10ms)

Pros: Always latest data
Cons: Primary handles read load too

Option B: Read from Replica (Common)
Client: SELECT * FROM users WHERE id=123;
   ↓
Replica 2: Returns data ✓
   ↓
Client: Gets response (10ms)

Pros: Offloads primary, scalable
Cons: Might be slightly stale (replication lag)
```

Replication Modes:

Asynchronous (Default):
```yaml
Client → Primary: UPDATE user 123
           ↓
Primary: Execute UPDATE
           ↓
Primary → Client: "Success!" ✓ (returns immediately)
           ↓
           (Async) Send to replicas in background
           ↓
Replicas: Apply change (few milliseconds later)

Pros:
✅ Fast writes (don't wait for replicas)
✅ Primary not blocked if replica is down

Cons:
❌ If primary crashes before replication, some writes lost!
❌ Replicas lag behind (eventual consistency)
```

Synchronous:
```yaml
Client → Primary: UPDATE user 123
           ↓
Primary: Execute UPDATE
           ↓
Primary → Replicas: Send change
           ↓
Wait for replicas to confirm...
           ↓
Replica 1: "Applied!" ✓
Replica 2: "Applied!" ✓
           ↓
Primary → Client: "Success!" ✓

Pros:
✅ No data loss (replicas guaranteed to have data)
✅ Read from replica = guaranteed fresh

Cons:
❌ Slower writes (wait for network round-trip)
❌ If all replicas down, primary blocks writes!
```

Semi-synchronous (Best of Both):
```yaml
Primary waits for at least 1 replica (not all):

Client → Primary: UPDATE user 123
           ↓
Primary: Execute UPDATE
           ↓
Primary → All Replicas: Send change
           ↓
Replica 1: "Applied!" ✓ (fastest replica)
           ↓
Primary → Client: "Success!" ✓ (doesn't wait for others)
           ↓
Replica 2: "Applied!" ✓ (slower, in background)
Replica 3: "Applied!" ✓

Pros:
✅ Fast enough (wait for 1 replica, not all)
✅ Data safe (at least 1 replica confirmed)
✅ If fastest replica down, waits for another

Balanced approach! ⚖️
```

Real-world parallel:
- Async = Mail drop box (drop and go, delivered later)
- Synchronous = Certified mail (wait for signature)
- Semi-sync = Quick signature from receptionist (fast but confirmed)

🎮 Decision Game: Which Replication Mode?

Context: You're configuring replication. Which mode should you use?

Scenarios:
A. Social media likes counter
B. Bank account balance
C. Blog post content
D. Shopping cart contents
E. User session data
F. Financial transaction log
G. Product catalog
H. Real-time analytics dashboard

Options:
1. Asynchronous (fast, might lose recent writes)
2. Semi-synchronous (balanced)
3. Synchronous (slow, no data loss)

Think about: Can you afford to lose data? Need fast writes?

Answers:
```
A. Social likes → Asynchronous (1)
   Reason: Losing a few likes acceptable, need fast

B. Bank balance → Synchronous (3)
   Reason: Cannot lose money data!

C. Blog posts → Semi-synchronous (2)
   Reason: Important but not critical, balanced

D. Shopping cart → Asynchronous (1)
   Reason: Temporary data, can reconstruct

E. User session → Asynchronous (1)
   Reason: OK if user re-logs in

F. Financial log → Synchronous (3)
   Reason: Audit trail, cannot lose

G. Product catalog → Semi-synchronous (2)
   Reason: Important but doesn't change often

H. Analytics → Asynchronous (1)
   Reason: Approximate data OK, need speed
```

Configuration Examples:
```sql
-- MySQL: Semi-synchronous replication
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000; -- 1 second

-- PostgreSQL: Synchronous replication
-- postgresql.conf
synchronous_commit = remote_apply
synchronous_standby_names = 'replica1,replica2'

-- MongoDB: Write concern
db.users.update(
  { _id: 123 },
  { $set: { email: 'new@example.com' } },
  { writeConcern: { w: "majority", j: true } }
)
```

🚨 Common Misconception: "Replicas Are Identical to Primary... Right?"

You might think: "Replica = exact copy at all times."

The Reality: Replicas are eventually consistent!

Replication Lag Scenario:
```
Time: 10:00:00.000
Primary: user.balance = $1000

Time: 10:00:00.010
Primary: UPDATE users SET balance = 900 WHERE id=123;
Primary: user.balance = $900 ✓

Time: 10:00:00.015 (5ms later)
Replica 1: Still sees user.balance = $1000 ✗ (lagging!)

Time: 10:00:00.050 (40ms later)
Replica 1: user.balance = $900 ✓ (caught up!)

During this 40ms window:
- Read from Primary: $900 ✓
- Read from Replica 1: $1000 ✗ (stale!)
```

Real-World Bug:
```python
# User withdraws money
def withdraw(user_id, amount):
    # Write to primary
    primary_db.execute(
        "UPDATE accounts SET balance = balance - %s WHERE id = %s",
        (amount, user_id)
    )

    # Redirect to account page
    return redirect('/account')

def account_page(user_id):
    # Reads from replica (for performance)
    account = replica_db.execute(
        "SELECT balance FROM accounts WHERE id = %s",
        (user_id,)
    ).fetchone()

    return f"Your balance: ${account.balance}"

# Bug!
# 1. User withdraws $100
# 2. Primary updated: balance = $900
# 3. Page redirects to /account
# 4. Reads from replica (not yet updated): balance = $1000
# 5. User sees old balance! "Withdrawal didn't work!"
```

Solutions:

Solution 1: Read Your Own Writes
```python
def withdraw(user_id, amount):
    primary_db.execute(...)

    # Store that this user should read from primary temporarily
    cache.set(f"read_from_primary:{user_id}", True, timeout=5)

    return redirect('/account')

def account_page(user_id):
    # Check if user needs primary reads
    if cache.get(f"read_from_primary:{user_id}"):
        account = primary_db.execute(...).fetchone()
    else:
        account = replica_db.execute(...).fetchone()

    return f"Your balance: ${account.balance}"
```

Solution 2: Sticky Sessions to Primary
```python
# Route user to primary for N seconds after write
session['route_to_primary_until'] = time.time() + 5

def get_db(session):
    if time.time() < session.get('route_to_primary_until', 0):
        return primary_db
    return replica_db
```

Solution 3: Check Replication Position
```python
def withdraw(user_id, amount):
    primary_db.execute(...)

    # Get current replication position
    position = primary_db.execute("SELECT pg_current_wal_lsn()").fetchone()
    session['min_replication_position'] = position

def account_page(user_id):
    min_pos = session.get('min_replication_position')

    # Find replica that's caught up
    for replica in replicas:
        replica_pos = replica.execute("SELECT pg_last_wal_replay_lsn()").fetchone()
        if replica_pos >= min_pos:
            return replica.execute(...).fetchone()

    # All replicas lagging, use primary
    return primary_db.execute(...).fetchone()
```

Real-world parallel: Replication lag is like news propagation. Breaking news (primary) takes time to reach newspapers (replicas).

⚡ Failover: Promoting a Replica to Primary

Automatic Failover Flow:
```
Normal State:
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Primary │────→│Replica 1│────→│Replica 2│
│ Server A│     │ Server B│     │ Server C│
└────┬────┘     └─────────┘     └─────────┘
     │
  All writes

Primary Crashes! 💥
     ↓
Step 1: Detection (10 seconds)
Health checks fail
Monitoring: "Primary is down!"
     ↓
Step 2: Choose new primary
Algorithm: Pick most up-to-date replica
Winner: Replica 1 (has most recent data)
     ↓
Step 3: Promote Replica 1
Replica 1: Becomes new primary
Replica 1: Stops following old primary
Replica 1: Starts accepting writes
     ↓
Step 4: Reconfigure other replicas
Replica 2: Now follows Replica 1 (new primary)
     ↓
Step 5: Update connection strings
DNS/Load Balancer: Points to new primary
Apps reconnect to new primary
     ↓
New State:
┌─────────┐     ┌─────────┐
│ PRIMARY │────→│Replica 2│
│(was Rep1)│     │ Server C│
│ Server B│     └─────────┘
└────┬────┘
     │
  All writes

Total downtime: 30-60 seconds
```

When Old Primary Returns:
```
Old Primary (Server A) comes back online:
     ↓
Option 1: Becomes replica
- Server A: Rewinds to divergence point
- Server A: Follows new primary (Server B)
- Now have 2 replicas again ✓

Option 2: Manual recovery
- DBA inspects for split-brain
- Manually reconfigures
- Safer but requires human intervention
```

Split-Brain Prevention:
```
Dangerous scenario:
Both old and new primary accept writes!

┌─────────┐                    ┌─────────┐
│ Old     │                    │ New     │
│ Primary │ Network partition! │ Primary │
│         │←──────✗──────────→│         │
│ Server A│                    │ Server B│
└────┬────┘                    └────┬────┘
     │                              │
 App 1 writes here            App 2 writes here

Data diverges! This is SPLIT-BRAIN! ✗

Prevention:
- Use quorum (need majority of nodes agreement)
- Fencing (shut down old primary forcefully)
- Distributed consensus (Raft, Paxos)
```

Tools for Automatic Failover:

MySQL with MHA (Master High Availability):
```bash
# MHA monitors primary health
# Automatically promotes replica on failure

# Configuration
cat /etc/mha/app1.cnf
[server default]
master_ip_failover_script=/usr/local/bin/master_ip_failover

[server1]
hostname=primary.db
candidate_master=1

[server2]
hostname=replica1.db
candidate_master=1

[server3]
hostname=replica2.db
no_master=1

# Run MHA manager
nohup masterha_manager --conf=/etc/mha/app1.cnf &

# When primary fails:
# 1. MHA detects failure
# 2. Promotes best replica
# 3. Reconfigures other replicas
# 4. Executes IP failover script
# 5. Updates DNS/VIP
```

PostgreSQL with Patroni:
```yaml
# patroni.yml
scope: postgres-ha
name: node1

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    maximum_lag_on_failover: 1048576

# Patroni uses etcd/Consul/Zookeeper for coordination
# Automatic leader election
# No manual intervention needed!

# Check cluster status
patronictl -c patroni.yml list

+ Cluster: postgres-ha (123456) -------+----+-----------+
| Member | Host      | Role    | State  | Lag in MB |
+--------+-----------+---------+--------+-----------+
| node1  | 10.0.0.1  | Leader  | running|         0 |
| node2  | 10.0.0.2  | Replica | running|         0 |
| node3  | 10.0.0.3  | Replica | running|         0 |
+--------+-----------+---------+--------+-----------+
```

Real-world parallel: Automatic failover is like a monarchy with clear succession rules. When king dies, crown prince automatically becomes king.

💡 Read-Write Splitting in Application

Application Pattern:
```python
class DatabaseRouter:
    def __init__(self):
        self.primary = connect('postgresql://primary:5432/db')
        self.replicas = [
            connect('postgresql://replica1:5432/db'),
            connect('postgresql://replica2:5432/db'),
            connect('postgresql://replica3:5432/db'),
        ]
        self.replica_index = 0

    def execute_write(self, query, params):
        """All writes go to primary"""
        return self.primary.execute(query, params)

    def execute_read(self, query, params):
        """Reads go to replicas (round-robin)"""
        replica = self.replicas[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replicas)
        return replica.execute(query, params)

# Usage
db = DatabaseRouter()

# Write operations
db.execute_write(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ("Alice", "alice@example.com")
)

# Read operations
users = db.execute_read(
    "SELECT * FROM users WHERE active = %s",
    (True,)
)
```

Django Database Router:
```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'primary.db',
    },
    'replica1': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica1.db',
    },
    'replica2': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica2.db',
    },
}

# database_router.py
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        """Reads go to replicas"""
        return random.choice(['replica1', 'replica2'])

    def db_for_write(self, model, **hints):
        """Writes go to primary"""
        return 'default'

# settings.py
DATABASE_ROUTERS = ['myapp.database_router.PrimaryReplicaRouter']

# Now all .save() goes to primary
# All .get() and .filter() go to replicas!
user = User.objects.get(id=123)  # Reads from replica
user.email = 'new@example.com'
user.save()  # Writes to primary
```

Real-world parallel: Read-write splitting is like having separate checkout (write) and browsing (read) in a store. Browsing happens anywhere, checkout at specific counter.

💡 Final Synthesis Challenge: The Organization Structure

Complete this comparison:
"A database without primary-replica is like a company with no delegation. A primary-replica database is like..."

Your answer should include:
- Write vs read separation
- Scalability benefits
- High availability
- Trade-offs

Take a moment to formulate your complete answer...

The Complete Picture:
A primary-replica database is like a well-organized company with clear delegation:

✅ **Primary (CEO)**: Makes all decisions (writes), source of truth
✅ **Replicas (Managers)**: Answer questions (reads), spread across locations
✅ **Delegation**: CEO not overwhelmed (writes are minority of workload)
✅ **Scalability**: Add more managers (replicas) = handle more inquiries (reads)
✅ **Succession**: CEO leaves, VP promoted (automatic failover)
✅ **Global presence**: Managers in each region (low latency reads worldwide)
✅ **Eventually consistent**: New policy (write) takes time to reach all managers

**Benefits:**
1. **Read Scalability** - Add replicas = handle more read traffic
2. **High Availability** - Primary fails, promote replica
3. **Disaster Recovery** - Data exists on multiple servers
4. **Geographic Distribution** - Serve users from nearest replica
5. **Offload Primary** - Primary focuses on writes

**Trade-offs:**
- Replication lag (eventual consistency)
- Writes don't scale (still bottleneck on primary)
- More complex application code (read-write splitting)
- Split-brain risk (need proper failover)

**Real-world examples:**
- Reddit: Read replicas for comment browsing
- Stack Overflow: Replicas in multiple data centers
- WordPress.com: Read replicas for blog views
- Netflix: Replicas for catalog browsing

Primary-replica transforms single-server databases into scalable, highly available systems!

🎯 Quick Recap: Test Your Understanding
Without looking back, can you explain:
1. Why is it called primary-replica instead of master-slave now?
2. How does primary-replica handle read-heavy workloads?
3. What is replication lag and when does it matter?
4. How does automatic failover work?

Mental check: If you can design a primary-replica system, you understand the pattern!

🚀 Your Next Learning Adventure
Now that you understand primary-replica, explore:

Advanced Topics:
- Multi-primary replication
- Cascading replication
- Delayed replicas (for recovery from mistakes)
- Logical replication

Replication Tools:
- ProxySQL (MySQL read-write splitting)
- PgBouncer (PostgreSQL connection pooling)
- HAProxy (database load balancing)
- Orchestrator (MySQL topology management)

Related Concepts:
- Database sharding
- Distributed databases
- CQRS (Command Query Responsibility Segregation)
- Event sourcing

Real-World Case Studies:
- How Wikipedia handles millions of readers
- GitHub's database architecture
- Instagram's replica lag handling
- Twitter's database evolution


---

## Key Takeaways

1. **Master-slave architecture routes all writes to one master and reads to replicas** — simple and effective for read-heavy workloads
2. **The master is a single point of failure** — automated failover with leader election is essential for production systems
3. **Replica lag means reads may return stale data** — acceptable for most use cases but not for operations like checking account balance after a transfer
4. **Semi-synchronous replication balances consistency and performance** — at least one replica confirms before the master commits
