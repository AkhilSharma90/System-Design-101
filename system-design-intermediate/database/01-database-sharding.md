---
slug: database-sharding
title: Database Sharding
readTime: 20 min
orderIndex: 1
premium: false
---


Database Sharding: Breaking Up Your Data for Massive Scale (Instagram's Growth Secret)
🎯 Challenge 1: The Phone Book Problem
Imagine this scenario: You're managing a phone book for an entire country - 300 million people.

Single Book Approach (Traditional Database):
```
┌─────────────────────────────────────┐
│   THE MASSIVE PHONE BOOK            │
│   300 Million Entries               │
│                                     │
│   Aaron ... 300MB                   │
│   Bob ...                           │
│   Carol ...                         │
│   ...                               │
│   Xavier ...                        │
│   Yvonne ...                        │
│   Zachary ... 300MB                 │
└─────────────────────────────────────┘

Problems:
❌ Book is HUGE (can't lift it!)
❌ Takes forever to find anything (scan millions of pages)
❌ Only one person can use it at a time (lock contention)
❌ Can't make it bigger (physical limit)
❌ Copying takes hours (backup nightmare)
```

Sharded Approach (Distributed Database):
```
Split alphabetically into 26 books:

Book A-B        Book C-D        Book E-F    ...    Book Y-Z
┌──────────┐    ┌──────────┐    ┌──────────┐      ┌──────────┐
│ Aaron    │    │ Carol    │    │ Emma     │      │ Yvonne   │
│ Bob      │    │ David    │    │ Frank    │      │ Zachary  │
│ 23MB     │    │ 23MB     │    │ 23MB     │      │ 23MB     │
└──────────┘    └──────────┘    └──────────┘      └──────────┘
Server 1        Server 2        Server 3    ...    Server 26

Benefits:
✅ Each book is manageable size
✅ Parallel lookups (26 people can search at once!)
✅ Easy to scale (add more servers)
✅ Fast queries (only search relevant book)
✅ Quick backups (each book backs up independently)
```

Pause and think: What if you could split your massive database across multiple servers, with each server handling only a portion of the data?

The Answer: Database Sharding splits your data horizontally across multiple databases! It's like:
✅ Each shard = independent database with subset of data
✅ Distribute load across multiple servers (no single bottleneck)
✅ Scale linearly (add shards = add capacity)
✅ Queries hit only relevant shards (faster lookups)
✅ Each shard can be optimized independently

Key Insight: Sharding trades simplicity for scalability - you get massive throughput but with added complexity!

🎬 Interactive Exercise: Vertical vs Horizontal Scaling

Before Sharding - Vertical Scaling (Bigger Server):
```
Year 1: Single Server
┌─────────────────────┐
│  Database           │
│  10M users          │
│  4 CPU, 16GB RAM    │
│  1TB SSD            │
└─────────────────────┘
Load: 1,000 queries/sec ✓

Year 2: Database growing...
┌─────────────────────┐
│  Database           │
│  50M users          │
│  8 CPU, 64GB RAM    │  ← Upgraded!
│  4TB SSD            │
└─────────────────────┘
Load: 5,000 queries/sec ✓

Year 3: Database still growing...
┌─────────────────────┐
│  Database           │
│  200M users         │
│  32 CPU, 256GB RAM  │  ← Upgraded again!
│  16TB SSD           │
└─────────────────────┘
Load: 20,000 queries/sec ⚠️ (struggling)
Cost: $50,000/month 💰

Year 4: Database too big!
❌ Can't upgrade further (hardware limits)
❌ Too expensive
❌ Single point of failure
❌ Can't handle load
```

With Sharding - Horizontal Scaling (More Servers):
```
Year 3: Split into 4 shards
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Shard 1      │  │ Shard 2      │  │ Shard 3      │  │ Shard 4      │
│ Users 1-50M  │  │ Users 51-100M│  │ Users 101-150M│ │ Users 151-200M│
│ 8CPU, 32GB   │  │ 8CPU, 32GB   │  │ 8CPU, 32GB   │  │ 8CPU, 32GB   │
│ 4TB SSD      │  │ 4TB SSD      │  │ 4TB SSD      │  │ 4TB SSD      │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
Load: 5,000 qps   Load: 5,000 qps   Load: 5,000 qps   Load: 5,000 qps
Total: 20,000 qps ✓
Cost: $10,000/month each = $40,000/month (cheaper!)

Year 5: Add more shards!
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Shard 1 │  │Shard 2 │  │Shard 3 │  │Shard 4 │  │Shard 5 │  │Shard 6 │
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘

Linear scaling! Just add more shards!
```

The Trade-off:
```
Vertical Scaling:
✅ Simple (just one database)
✅ ACID transactions work normally
✅ No cross-server queries
❌ Hardware limits (can't scale forever)
❌ Expensive (big servers cost more per resource)
❌ Single point of failure

Horizontal Scaling (Sharding):
✅ Unlimited scaling (add more shards)
✅ Cheaper (commodity servers)
✅ High availability (one shard fails, others work)
❌ Complex (application must know about shards)
❌ Difficult cross-shard queries
❌ Distributed transactions are hard
```

Real-world parallel: Vertical scaling is like building a taller building (expensive, has limits). Horizontal scaling is like building more buildings (cheaper, unlimited).

🏗️ Sharding Strategies: How to Split Your Data

Strategy 1: Range-Based Sharding
```
Split by value ranges:

Users table:
user_id: 1-1M      → Shard 1
user_id: 1M-2M     → Shard 2
user_id: 2M-3M     → Shard 3
user_id: 3M-4M     → Shard 4

┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Shard 1     │  │ Shard 2     │  │ Shard 3     │  │ Shard 4     │
│ Users 1-1M  │  │ Users 1M-2M │  │ Users 2M-3M │  │ Users 3M-4M │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘

Query: "Get user 1,500,000"
Router: 1.5M falls in range 1M-2M → Shard 2

Pros:
✅ Simple to understand
✅ Range queries are efficient (users 1M-1.1M all on Shard 2)
✅ Easy to add new shard (just extend range)

Cons:
❌ Uneven distribution (new users go to latest shard = hot spot!)
❌ Older shards might be idle (cold data)
❌ Hard to rebalance
```

Strategy 2: Hash-Based Sharding
```
Use hash function to determine shard:

shard = hash(user_id) % number_of_shards

Example with 4 shards:
user_id: 123   → hash(123) = 456789 → 456789 % 4 = 1 → Shard 1
user_id: 456   → hash(456) = 234567 → 234567 % 4 = 3 → Shard 3
user_id: 789   → hash(789) = 891234 → 891234 % 4 = 2 → Shard 2
user_id: 1011  → hash(1011) = 567890 → 567890 % 4 = 2 → Shard 2

┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Shard 0     │  │ Shard 1     │  │ Shard 2     │  │ Shard 3     │
│ 25% of users│  │ 25% of users│  │ 25% of users│  │ 25% of users│
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘

Pros:
✅ Even distribution (hash spreads data uniformly)
✅ No hot spots (new and old users distributed evenly)
✅ Simple logic

Cons:
❌ Range queries don't work (users 1-1000 scattered across all shards)
❌ Adding/removing shards requires rehashing ALL data!
❌ Can't easily change number of shards
```

Strategy 3: Consistent Hashing
```
Hash both shards and keys onto a circle:

         0°
    ┌────────┐
    │        │
    │  Hash  │
270°│ Circle │ 90°
    │        │
    └────────┘
        180°

Shard positions (hash of shard name):
Shard 1: 45°
Shard 2: 120°
Shard 3: 200°
Shard 4: 310°

User positions (hash of user_id):
User 123: 30°  → Goes to next shard clockwise = Shard 1 (45°)
User 456: 150° → Goes to Shard 3 (200°)
User 789: 250° → Goes to Shard 4 (310°)
User 999: 340° → Goes to Shard 1 (45°, wraps around)

Adding Shard 5 at 170°:
Only users between 120° and 170° move from Shard 3 to Shard 5!
(~12.5% of data moves instead of 100%!)

Pros:
✅ Even distribution
✅ Adding/removing shards only affects neighbors
✅ Minimal data movement

Cons:
❌ More complex to implement
❌ Need virtual nodes for balance
```

Strategy 4: Directory-Based Sharding
```
Maintain a lookup table:

┌──────────────────────────────┐
│  Shard Directory             │
├──────────────┬───────────────┤
│ User ID      │ Shard         │
├──────────────┼───────────────┤
│ 1-500,000    │ Shard 1       │
│ 500,001-1M   │ Shard 2       │
│ 1M-2M        │ Shard 3       │
│ 2M-3M        │ Shard 4       │
└──────────────┴───────────────┘

Query: "Get user 750,000"
1. Look up directory: 750,000 → Shard 2
2. Query Shard 2

Pros:
✅ Flexible (can assign any range to any shard)
✅ Easy to rebalance (just update directory)
✅ Can handle uneven data

Cons:
❌ Directory is single point of failure
❌ Extra lookup on every query (latency)
❌ Directory can become bottleneck
```

Code Example (Hash-based Sharding):
```python
import hashlib

class ShardRouter:
    def __init__(self, num_shards):
        self.num_shards = num_shards
        self.shards = [
            f"postgresql://shard{i}.db.example.com:5432/mydb"
            for i in range(num_shards)
        ]

    def get_shard(self, user_id):
        # Hash user_id and determine shard
        hash_value = int(hashlib.md5(str(user_id).encode()).hexdigest(), 16)
        shard_index = hash_value % self.num_shards
        return self.shards[shard_index]

    def get_user(self, user_id):
        shard_url = self.get_shard(user_id)
        conn = connect_to_database(shard_url)

        result = conn.execute(
            "SELECT * FROM users WHERE user_id = %s",
            (user_id,)
        )
        return result

# Usage
router = ShardRouter(num_shards=4)

# These users go to different shards
user1 = router.get_user(12345)   # → Shard 2
user2 = router.get_user(67890)   # → Shard 0
user3 = router.get_user(11111)   # → Shard 3
```

Real-world parallel:
- Range-based = Library sections (A-F, G-M, N-Z)
- Hash-based = Random assignment (ensures even distribution)
- Consistent hashing = Circle of responsibility (easy to add/remove)
- Directory = Phone book with map (flexible but need to check map)

🎮 Decision Game: Choose Your Sharding Key

Context: You're sharding different types of data. What should be the shard key?

Scenarios:
A. Social media posts table (need to show user's posts)
B. E-commerce orders table (need order history per user)
C. Log events table (time-series data)
D. Product catalog (need to search by category)
E. Messages table (conversation threads)
F. Analytics events (billions of events)

Options:
1. user_id (group by user)
2. created_at (group by time)
3. category (group by type)
4. conversation_id (group by thread)
5. event_id (random distribution)

Think about: How is data accessed most often?

Answers:
```
A. Social media posts → user_id (1)
   Reason: "Show posts by user" is primary query
   Result: All user's posts on same shard (fast!)

B. E-commerce orders → user_id (1)
   Reason: "Show my orders" is common
   Result: User's order history on one shard

C. Log events → created_at (2)
   Reason: Queries are time-based ("logs from last hour")
   Result: Recent logs on same shard

D. Product catalog → category (3)
   Reason: Browse by category is primary
   Result: All electronics on one shard, clothing on another

E. Messages → conversation_id (4)
   Reason: Need all messages in thread together
   Result: Entire conversation on one shard

F. Analytics events → event_id (5)
   Reason: No specific access pattern, need distribution
   Result: Events spread evenly
```

The Golden Rules for Shard Key Selection:
```
Good shard key:
✅ High cardinality (many unique values)
✅ Even distribution (no hot spots)
✅ Matches query patterns (data accessed together is stored together)
✅ Immutable (doesn't change over time)

Bad shard key:
❌ Low cardinality (country: only ~200 values)
❌ Monotonically increasing (user_id: new users all go to same shard)
❌ Doesn't match queries (need to hit all shards for common queries)
❌ Changes frequently (requires data movement)
```

🚨 Common Misconception: "Sharding Solves All Scaling Problems... Right?"

You might think: "Just shard my database and infinite scale!"

The Reality: Sharding introduces significant complexity!

Problems Sharding Creates:

Problem 1: Cross-Shard Queries
```sql
-- Simple query on single database:
SELECT u.name, COUNT(p.id) as post_count
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.name;

-- With sharding:
Shard 1: Has users 1-1M and their posts
Shard 2: Has users 1M-2M and their posts
Shard 3: Has users 2M-3M and their posts

Query must:
1. Hit ALL shards (scatter)
2. Get partial results from each
3. Aggregate results in application (gather)
4. Much slower! ⚠️

Application must do the JOIN and GROUP BY!
```

Problem 2: Distributed Transactions
```python
# Transfer money between users on different shards

def transfer_money(from_user_id, to_user_id, amount):
    from_shard = get_shard(from_user_id)  # Shard 1
    to_shard = get_shard(to_user_id)      # Shard 3

    # Need 2-phase commit across shards!

    # Phase 1: Prepare
    from_shard.begin_transaction()
    from_shard.execute(
        "UPDATE accounts SET balance = balance - %s WHERE user_id = %s",
        (amount, from_user_id)
    )
    from_shard.prepare()  # Vote to commit

    to_shard.begin_transaction()
    to_shard.execute(
        "UPDATE accounts SET balance = balance + %s WHERE user_id = %s",
        (amount, to_user_id)
    )
    to_shard.prepare()  # Vote to commit

    # Phase 2: Commit
    if both_voted_yes:
        from_shard.commit()
        to_shard.commit()
    else:
        from_shard.rollback()
        to_shard.rollback()

    # Complex! Error-prone! Slow!
```

Problem 3: Auto-increment IDs Don't Work
```
Shard 1: user_id = 1, 2, 3, 4...
Shard 2: user_id = 1, 2, 3, 4... ✗ Collision!

Solutions:
1. UUID (globally unique, but large and not sequential)
2. Snowflake ID (Twitter's solution: timestamp + shard_id + sequence)
3. Database sequence with gaps (Shard 1: 1,4,7,10... Shard 2: 2,5,8,11...)

Example Snowflake ID:
┌────────────────────────┬──────────┬──────────────┐
│   Timestamp (41 bits)  │Shard (10)│Sequence (12) │
└────────────────────────┴──────────┴──────────────┘
= 64-bit unique ID
```

Problem 4: Schema Changes
```
-- Without sharding:
ALTER TABLE users ADD COLUMN email_verified BOOLEAN;
(One command, done!)

-- With sharding (4 shards):
ALTER TABLE users ADD COLUMN email_verified BOOLEAN; -- Shard 1
ALTER TABLE users ADD COLUMN email_verified BOOLEAN; -- Shard 2
ALTER TABLE users ADD COLUMN email_verified BOOLEAN; -- Shard 3
ALTER TABLE users ADD COLUMN email_verified BOOLEAN; -- Shard 4

Problems:
- What if one fails?
- Need to coordinate across shards
- Downtime multiplied by number of shards
- Can't do atomic schema changes
```

Problem 5: Resharding (Changing Number of Shards)
```
Initial: 4 shards
user_id 123 → hash(123) % 4 = 3 → Shard 3

Add 5th shard (now 5 shards total):
user_id 123 → hash(123) % 5 = 2 → Shard 2 ✗

User 123's data is on Shard 3 but now maps to Shard 2!
Need to move ~80% of all data!

This is called resharding and it's EXPENSIVE!
```

Real-world parallel: Sharding is like opening multiple branch offices:
- More capacity (good!)
- But coordination meetings are harder (cross-shard queries)
- Transactions across branches are complex (distributed transactions)
- Reorganizing offices is disruptive (resharding)

⚡ Sharding in Practice: Real-World Architecture

Instagram's Sharding Strategy:
```
Users table sharded by user_id:

┌────────────────────────────────────────────────────┐
│              Routing Layer                         │
│   (knows which shard has which user_id range)      │
└────────┬─────────┬─────────┬─────────┬────────────┘
         │         │         │         │
    ┌────▼───┐┌───▼────┐┌───▼────┐┌───▼────┐
    │Shard 1 ││Shard 2 ││Shard 3 ││Shard 4 │
    │Users   ││Users   ││Users   ││Users   │
    │1-250M  ││250M-   ││500M-   ││750M-1B │
    │        ││500M    ││750M    ││        │
    └────────┘└────────┘└────────┘└────────┘

Posts table sharded by user_id (same key!):
    ┌────────┐┌────────┐┌────────┐┌────────┐
    │Shard 1 ││Shard 2 ││Shard 3 ││Shard 4 │
    │Posts   ││Posts   ││Posts   ││Posts   │
    │by users││by users││by users││by users│
    │1-250M  ││250M-   ││500M-   ││750M-1B │
    └────────┘└────────┘└────────┘└────────┘

Key insight: User and their posts on SAME shard!
This allows fast joins within a shard!

Query: "Get user 150M and their posts"
Router: → Shard 1 (150M is in 1-250M range)
Shard 1:
  SELECT * FROM users WHERE id = 150000000;
  SELECT * FROM posts WHERE user_id = 150000000;
Fast! Both tables on same shard!
```

Application Code with Sharding:
```python
from sqlalchemy import create_engine

class ShardedDatabase:
    def __init__(self):
        self.shards = {
            0: create_engine('postgresql://shard0.db:5432/mydb'),
            1: create_engine('postgresql://shard1.db:5432/mydb'),
            2: create_engine('postgresql://shard2.db:5432/mydb'),
            3: create_engine('postgresql://shard3.db:5432/mydb'),
        }

    def get_shard_for_user(self, user_id):
        # Range-based sharding
        if user_id < 250_000_000:
            return self.shards[0]
        elif user_id < 500_000_000:
            return self.shards[1]
        elif user_id < 750_000_000:
            return self.shards[2]
        else:
            return self.shards[3]

    def get_user_with_posts(self, user_id):
        shard = self.get_shard_for_user(user_id)

        # Query within single shard (fast!)
        user = shard.execute(
            "SELECT * FROM users WHERE id = %s",
            (user_id,)
        ).fetchone()

        posts = shard.execute(
            "SELECT * FROM posts WHERE user_id = %s ORDER BY created_at DESC LIMIT 20",
            (user_id,)
        ).fetchall()

        return {'user': user, 'posts': posts}

    def get_users_by_country(self, country):
        # Scatter-gather query (hits all shards)
        results = []

        for shard in self.shards.values():
            shard_results = shard.execute(
                "SELECT * FROM users WHERE country = %s",
                (country,)
            ).fetchall()
            results.extend(shard_results)

        return results

# Usage
db = ShardedDatabase()

# Fast query (single shard)
user_data = db.get_user_with_posts(150_000_000)

# Slow query (all shards)
us_users = db.get_users_by_country('US')
```

Vitess (MySQL Sharding Framework):
```yaml
# VSchema - defines how data is sharded
{
  "sharded": true,
  "vindexes": {
    "hash": {
      "type": "hash"
    }
  },
  "tables": {
    "users": {
      "column_vindexes": [
        {
          "column": "user_id",
          "name": "hash"
        }
      ]
    },
    "posts": {
      "column_vindexes": [
        {
          "column": "user_id",
          "name": "hash"
        }
      ]
    }
  }
}

# Vitess automatically routes queries to correct shard!
# Application doesn't need to know about sharding
```

Real-world parallel: Sharding framework is like a postal system. You write address (user_id), postal system (Vitess) figures out which post office (shard) to route to.

🔧 Best Practices for Sharding

1. Denormalize to Avoid Cross-Shard Queries:
```sql
-- Bad: Normalized (requires cross-shard JOIN)
-- Users table (sharded by user_id)
-- Posts table (sharded by post_id) ✗

-- Good: Denormalized (same shard key)
-- Users table (sharded by user_id)
-- Posts table (sharded by user_id) ✓
--   - Includes author_name (denormalized!)
--   - Includes author_avatar (denormalized!)

-- Now can get user + posts without cross-shard query!
```

2. Use Consistent Hashing or Add Shards Carefully:
```python
# Bad: Simple modulo (adding shard requires moving ALL data)
shard = hash(user_id) % num_shards

# Better: Use consistent hashing
# Or: Plan for resharding from the start

# Example: Pre-allocate logical shards
# 1024 logical shards mapped to 4 physical shards
logical_shard = hash(user_id) % 1024
physical_shard = logical_shard % 4

# Later, can split physical shard without changing logical mapping
```

3. Monitor Shard Balance:
```python
def check_shard_balance():
    for shard_id, shard in shards.items():
        row_count = shard.execute("SELECT COUNT(*) FROM users").fetchone()[0]
        disk_usage = shard.execute("SELECT pg_database_size('mydb')").fetchone()[0]

        print(f"Shard {shard_id}:")
        print(f"  Rows: {row_count:,}")
        print(f"  Size: {disk_usage / 1024**3:.2f} GB")

# Alert if one shard has >30% more data than average (hot spot!)
```

4. Have a Resharding Plan:
```
Option 1: Shard Splitting (double shards)
4 shards → 8 shards
Each shard splits in half

Option 2: Consistent Hashing
Add new shard, only ~12.5% of data moves

Option 3: Logical Shards
Start with 1024 logical shards on 4 physical shards
Later redistribute logical shards to more physical shards

All require:
- Maintenance window OR
- Dual-write strategy (write to old and new location)
```

💡 Final Synthesis Challenge: The Library System

Complete this comparison:
"A single library building is simple but limited. A sharded database is like..."

Your answer should include:
- Data distribution strategy
- Query routing
- Trade-offs and complexity
- When to shard vs when not to

Take a moment to formulate your complete answer...

The Complete Picture:
A sharded database is like a library system with multiple branches across the city:

✅ **Distribution**: Books split across branches (by author, genre, or location)
✅ **Capacity**: Each branch manageable size, unlimited growth (add more branches)
✅ **Parallel access**: Multiple people can search simultaneously (different branches)
✅ **Local optimization**: Each branch optimized for its collection
✅ **Routing**: Need directory to know which branch has which books
✅ **Complex queries**: "Find all science books" requires checking all branches (slow)
✅ **Coordination**: Moving books between branches is expensive (resharding)
✅ **Trade-off**: More capacity but more complexity

**When to shard:**
- Single database can't handle load (>10K qps)
- Data too large for one server (>1TB)
- Clear sharding key (user_id, tenant_id)
- Mostly single-shard queries

**When NOT to shard:**
- Can still scale vertically (< 1TB, < 10K qps)
- Lots of cross-entity queries (JOINs everywhere)
- No clear sharding key
- Team too small (sharding requires expertise)

**Real-world examples:**
- Instagram: Sharded by user_id (billions of users)
- Slack: Sharded by workspace_id (millions of workspaces)
- GitHub: Sharded by repository_id
- Twitter: Custom sharding (Snowflake IDs)

Sharding transforms single-server limits into distributed scalability - but only when the complexity is worth it!

🎯 Quick Recap: Test Your Understanding
Without looking back, can you explain:
1. What's the difference between vertical and horizontal scaling?
2. How does hash-based sharding differ from range-based?
3. Why are cross-shard queries problematic?
4. When should you shard your database?

Mental check: If you can design a sharding strategy, you understand database sharding!

🚀 Your Next Learning Adventure
Now that you understand sharding, explore:

Advanced Sharding:
- Vitess (MySQL sharding)
- Citus (PostgreSQL sharding)
- MongoDB sharding architecture
- YugabyteDB distributed SQL

Sharding Challenges:
- Resharding strategies
- Cross-shard transactions
- Global secondary indexes
- Consistent hashing deep-dive

Related Concepts:
- Database replication
- Partitioning vs sharding
- Multi-tenancy strategies
- NewSQL databases

Real-World Case Studies:
- How Discord scaled to billions of messages
- Uber's sharding evolution
- Pinterest's sharding journey
- Shopify's multi-tenant sharding
