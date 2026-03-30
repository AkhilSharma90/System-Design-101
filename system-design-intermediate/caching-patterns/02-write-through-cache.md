---
slug: write-through-cache
title: Write Through Cache
readTime: 15 min
orderIndex: 2
premium: false
---
Write-Through Cache: The Art of Writing Everywhere at Once
🎯 Challenge 1: The Consistency Guarantee Problem

Imagine this scenario: You're building a banking application. When a user updates their account balance, it MUST be consistent everywhere immediately. No stale data allowed - we're talking about money!

Your cache can serve reads in 2ms, but your database takes 50ms. You need both speed AND consistency.

Pause and think: How do you keep cache and database perfectly in sync without sacrificing all your speed gains?

The Answer: Write-Through Cache acts as a guaranteed synchronized writer. Every write goes through the cache to the database, ensuring cache and database are always consistent. The cache becomes the single entry point for all writes.

Key Insight: Write-Through writes to cache AND database simultaneously (or sequentially), guaranteeing consistency!

🏦 Interactive Exercise: The Dual-Ledger Bookkeeper

Scenario: You're an old-fashioned bookkeeper with two ledgers:
- Quick Reference Ledger (cache) - on your desk, fast access
- Official Master Ledger (database) - in the vault, permanent record

Someone makes a payment. What do you do?

Think about the steps:
1. Write the payment in BOTH ledgers immediately
2. Don't tell the customer "done" until BOTH are updated
3. Now both ledgers match perfectly
4. Future reads can use the quick reference (fast!)

Question: Why not just update the master ledger and update the quick reference "later"?

Write-Through Flow: The Digital Version

Cache handles all writes and ensures database consistency:

```
User updates account balance from $100 to $150:

Application: "Save new balance: $150"
             → Writes to Cache

Cache:       "Let me save this AND pass it to the database"
             1. Store $150 in cache ✓
             2. → Write to Database

Database:    "Saved $150" ✓

Cache:       ← "Database confirmed!"
             "Both locations updated!" ✓

Application: ← "Write successful!"


Next read request:

Application: → Read from Cache
Cache:       "$150" (Fast! And guaranteed fresh!)
Application: → Return $150 to user
```

Real-world parallel: Like writing in both your planner AND your phone's calendar at the same time. Sure, it takes a bit longer to write, but now you'll never have conflicting information!

Key terms decoded:
- Write-Through = Every write goes through cache to database
- Synchronous = Write completes only after BOTH cache and database are updated
- Strong Consistency = Cache and database always match

🚨 Common Misconception: "Write-Through Speeds Up Writes... Right?"

You might think: "Cache is fast, so writing to cache must be faster!"

The Reality Check:

Write-Through is actually SLOWER for writes than writing directly to the database!

```
Direct Database Write:
Application → Database: 50ms

Write-Through:
Application → Cache: Store in cache (2ms)
             → Database: Write to DB (50ms)
             ← Wait for confirmation
Total: 52ms (slower!)
```

Mental model: Write-Through is like carbon copy forms. Writing on the top sheet creates copies below, but you're still writing at the same speed - actually slightly slower because there's more layers!

Challenge question: If writes are slower, why use Write-Through at all?

The Answer: Speed isn't everything! Benefits include:
- Reads are blazing fast (served from cache)
- Cache and database always consistent
- No stale data issues
- Simplified logic (one write path)
- Fewer cache invalidation headaches

🎮 Decision Game: Cache Server Crashes!

Context: Your application uses Write-Through pattern. Suddenly, your cache server crashes completely.

What happens to new writes?
A. All writes fail (cache is required)
B. Writes go directly to database, reads slow down
C. Application automatically switches to Cache-Aside mode
D. Data becomes inconsistent

Think about it... What's the role of cache in Write-Through?

Answer: It depends on implementation, but typically A or B!

Here's the challenge with Write-Through:

```
Scenario 1: Cache as mandatory gateway
Cache is DOWN 💀

Application → Cache: Write request
Cache: 💀 (no response)
Application: Error! Write failed! ✗

Problem: Availability suffers
Benefit: Consistency maintained (no split-brain)

Scenario 2: Fallback to database
Cache is DOWN 💀

Application → Cache: Write request (fails)
Application → Database: Write directly ✓
Application: Success! But reads are slower

Problem: Reads slow down (no cache)
Benefit: Writes still work
```

Real-world parallel: If your planner is lost, you can still write in your phone's calendar directly, but you lose the benefit of the quick physical reference!

Key insight: Write-Through typically makes cache a critical component. Cache downtime affects write availability!

🚰 Problem-Solving Exercise: The Write Performance Problem

Scenario: You're building a high-traffic blogging platform. Users post comments constantly. You implement Write-Through cache, and now you notice:

```
Metrics:
- Write latency: 60ms (was 50ms without cache)
- Comment submission rate: Decreased by 15%
- Database load: Still high!
- Cache hit rate for reads: 95% (excellent!)
```

What do you think is happening?
1. Write-Through is adding overhead to every write
2. Users are waiting for both cache and database
3. No benefit for write-heavy workloads

Solution: Understand Write-Through's trade-off!

Write-Through is optimized for READ-HEAVY workloads:

```
Workload Analysis:

Read-Heavy Application (e.g., Product Catalog):
Reads:  10,000 requests/sec → Cache serves in 2ms ✓
Writes: 10 updates/sec → Takes 60ms ✗ (but rare!)
Result: Great! Most requests are fast

Write-Heavy Application (e.g., Comment System):
Reads:  100 requests/sec → Cache serves in 2ms ✓
Writes: 5,000 writes/sec → Takes 60ms each ✗
Result: Poor! Most requests are slow

Real-world Read vs Write patterns:
News Site: 99% reads, 1% writes → Perfect for Write-Through!
Chat App:  50% reads, 50% writes → Bad for Write-Through!
Analytics: 10% reads, 90% writes → Terrible for Write-Through!
```

Mental model: Write-Through is like having a translator at a meeting. If most people speak your language (reads from cache), great! But if you need the translator for every sentence (writes), the meeting becomes painfully slow.

The Write-Through sweet spot:
- Read:Write ratio of at least 10:1
- Reads must be fast (latency-sensitive)
- Writes can tolerate slightly higher latency
- Strong consistency required

Pro tip: For write-heavy workloads, consider Write-Behind (async) or just write to database directly!

🔍 Investigation: The Cache Failure Resilience

Imagine your caching layer has issues:
- Scenario A: Redis cluster is experiencing intermittent timeouts
- Scenario B: Network partition separates cache from application
- Scenario C: Cache is full and evicting entries

What happens with Write-Through in each case?

```
Scenario A: Intermittent Cache Timeouts
User: "Save my data"
App → Cache: Write attempt... timeout! ⏰
App: Retry? Fail? Wait?

Decision needed:
Option 1: Fail fast (user sees error)
Option 2: Retry writes (increases latency)
Option 3: Fallback to database only

Scenario B: Network Partition
App sees: Cache appears down
App decides: Write to database only
Result: Cache becomes stale!

Other apps: Read stale data from cache 😱
Solution: Either clear cache or wait for repair

Scenario C: Cache Full (LRU Eviction)
App → Cache: Write new data
Cache: "I'm full, evicting old entry X"
       "Storing your new data"
App: Success! ✓

Old entry X: Still in database ✓
Next read of X: Cache miss → read from DB ✓
Result: No problem! This is normal
```

Mental model: Write-Through is like a valet parking system. If the valet booth closes, you need a plan - do you park yourself, or do you come back later?

The resilience strategy:

```
Best Practice Implementation:

try {
    cache.write(key, data)
} catch (CacheException e) {
    // Cache is down! What now?

    log.error("Cache unavailable: " + e)
    metrics.increment("cache.write.failure")

    // Option 1: Fail the write
    throw new ServiceUnavailableException()

    // Option 2: Write to DB only (risk stale cache)
    database.write(key, data)
    // Schedule cache clear/refresh

    // Option 3: Queue for retry
    retryQueue.add(writeOperation)
}
```

🧩 Implementation Challenge: The Code Pattern

Scenario: You're implementing Write-Through cache. Here's the pattern:

```
function saveUser(userId, userData):
    try {
        // Step 1: Write to cache first (or simultaneously)
        cache.set(userId, userData)

        // Step 2: Write to database
        database.update("users", userId, userData)

        // Step 3: Both succeeded!
        return success

    } catch (error) {
        // Uh oh! What if cache succeeded but database failed?
        // Or vice versa?

        // Need to rollback? Need consistency?
        handleWriteFailure(error)

        return failure
    }
```

Question: What happens if the cache write succeeds but the database write fails?

The Consistency Problem:

```
Step 1: cache.set(userId, userData) ✓
Step 2: database.update() ✗ FAILS!

Result:
Cache has: New data
Database has: Old data
INCONSISTENCY! 😱

Next read:
App → Cache: Returns new data
But database doesn't have it!

If cache clears:
App → Database: Returns old data
Confusion! Data loss!
```

The Solution: Write-Through Order Matters!

```
Better Implementation:

function saveUser(userId, userData):
    try {
        // Step 1: Write to DATABASE first (source of truth)
        database.update("users", userId, userData)

        // Step 2: Then write to cache
        cache.set(userId, userData)

        return success

    } catch (error) {
        if error in database write:
            // Neither cache nor DB updated
            // Consistent! ✓
            return error

        if error in cache write:
            // DB updated ✓, cache not updated ✗
            // Not ideal, but database is correct
            // Next read will be cache miss → reads from DB ✓
            // Eventually consistent
            log.warning("Cache write failed, data in DB")
            return success (or retry cache)
    }
```

Real-world parallel: When filing your taxes, you fill out the official forms first (database), then maybe update your personal records (cache). If your personal records don't get updated, the official version is still correct!

The Golden Rules:
1. **Database is source of truth** - write there first
2. **Cache write failure is tolerable** - next read fetches from DB
3. **Database write failure is critical** - abort the operation

👋 Interactive Journey: Read-After-Write Consistency

Scenario: A user updates their email address. Immediately after, they refresh the page to see their new email. What should they see?

```
Time: 10:00:00.000
User: "Update email to new@email.com"
App: Starts write process...

Time: 10:00:00.050
Database: Email updated to new@email.com ✓

Time: 10:00:00.052
Cache: Email updated to new@email.com ✓

Time: 10:00:00.053
App → User: "Updated successfully!"

Time: 10:00:00.100
User: Refresh page (immediate read)
App → Cache: "Get user email"
Cache: "new@email.com" ✓
App → User: Shows new@email.com ✓

User: "Perfect! I see my update immediately!" 😊
```

This is Write-Through's superpower!

Compare with Cache-Aside:

```
Cache-Aside (for comparison):

Time: 10:00:00.000
User: "Update email to new@email.com"

Time: 10:00:00.050
App: Update database ✓
App: Invalidate cache ✓

Time: 10:00:00.100
User: Refresh page
App → Cache: "Get user email"
Cache: "Not found" (was invalidated)
App → Database: "Get user email"
Database: "new@email.com"
App → Cache: Store "new@email.com"
App → User: Shows new@email.com ✓

Small delay, but still works!
```

Write-Through advantage: **No cache miss penalty after writes!**

Mental model: Write-Through is like updating your resume in Google Docs. The moment you hit save, it's updated everywhere. When you reload the page, you see your changes immediately - no lag, no cache miss, no delay!

🎪 The Great Comparison: Write-Through vs Real-World Services

Let's solidify your understanding. Match Write-Through behaviors to real-world scenarios:

Write-Through Behavior → Real-World Equivalent
-------------------------
Write to both locations → ?
Synchronous writing → ?
Consistent reads → ?
Write latency penalty → ?
Single entry point → ?

Think about each one...

Answers revealed:
```
Write to both locations → 📋 Carbon copy forms, writing in two ledgers
Synchronous writing → 🔄 Waiting for both saves to complete before continuing
Consistent reads → 📊 Always seeing up-to-date information
Write latency penalty → ⏱️ Taking extra time to ensure everything is recorded
Single entry point → 🚪 All traffic goes through one door for writes
```

The big picture: Write-Through is like a meticulous librarian who updates both the computer catalog AND the physical card catalog before telling you the book is added!

💡 Final Synthesis Challenge: The Trade-off Decision

Complete this analysis:

"I should use Write-Through instead of Cache-Aside when..."

Your answer should consider:
- Consistency requirements
- Read/write patterns
- Latency tolerance
- Complexity preferences

Take a moment to formulate your complete answer...

The Complete Picture:

Use Write-Through when:

✅ Strong consistency required
   - Financial transactions
   - User profile updates
   - Inventory management

✅ Read-heavy workload (reads >> writes)
   - Most requests can be served from cache
   - Write latency penalty is rare

✅ Read-after-write consistency needed
   - Users expect to see their changes immediately
   - No stale data tolerance

✅ Simplified cache management desired
   - No need for cache invalidation logic
   - Cache always reflects database

✅ Cache as central component is acceptable
   - Can tolerate cache being critical path
   - Have good cache infrastructure

Avoid Write-Through when:

❌ Write-heavy workload
   - Writes would slow everything down
   - Cache becomes bottleneck

❌ Need highest write performance
   - Can't afford write latency increase
   - Async writes preferred

❌ Cache availability is poor
   - Frequent cache failures
   - Can't make cache critical path

❌ Eventual consistency is acceptable
   - Stale data for brief period is okay
   - Cache-Aside would be simpler

Real-world examples:

Write-Through works well for:
- User profile services (read-heavy, consistency needed)
- Configuration management (read frequently, write rarely)
- Product catalogs (many reads, few updates)
- Session management (consistency critical)

Write-Through struggles with:
- Activity logs (write-heavy, no need for cache)
- Metrics collection (tons of writes)
- Real-time analytics (write-intensive)

🎯 Quick Recap: Test Your Understanding

Without looking back, can you explain:

1. Why is it called "write-through"?
2. Does Write-Through make writes faster or slower?
3. What's the benefit of Write-Through over Cache-Aside?
4. What happens if the cache write fails?
5. What workload pattern is ideal for Write-Through?

Mental check: If you can answer these clearly, you've mastered Write-Through! If not, revisit the relevant sections above.

📊 The Write-Through Cheat Sheet

```
Characteristics:
- Pattern Type: Cache manages database writes
- Loading: Can be lazy or pre-loaded
- Read Path: Check cache → Miss? → Read DB → Update cache
- Write Path: Write to cache → Cache writes to DB → Both updated
- Consistency: Strong (cache and DB always in sync)
- Complexity: Medium (simpler than Cache-Aside for writes)

Pros:
+ Strong consistency (cache and database always match)
+ No cache invalidation needed for writes
+ Read-after-write consistency guaranteed
+ Simple write logic (one path)
+ Reads are very fast
+ No stale data issues

Cons:
- Writes are slower (must update both)
- Write latency penalty
- Cache can become single point of failure
- Poor for write-heavy workloads
- Cache must be reliable and available
- Every write incurs both cache and DB latency

Best for:
✓ User session management
✓ User profiles and preferences
✓ Configuration data
✓ Product information with occasional updates
✓ Any read-heavy workload with consistency needs
```

📈 Performance Comparison

```
Operation Performance:

Cache-Aside:
- Read (cache hit):    2ms  ✓ Fast
- Read (cache miss):   50ms ⚠️ Slow
- Write:               50ms ✓ Fast (just DB)

Write-Through:
- Read (cache hit):    2ms  ✓ Fast
- Read (cache miss):   50ms ⚠️ Slow
- Write:               52ms ⚠️ Slower (cache + DB)

Cache-Aside wins at: Writes
Write-Through wins at: Consistency, read-after-write

When cache hit rate is high (>95%):
Both patterns serve most reads in 2ms
Write-Through provides better consistency
Cache-Aside provides faster writes
```

🚀 Your Next Learning Adventure

Now that you understand Write-Through, you're ready to explore:

Immediate comparisons:
- Write-Behind (Write-Back): What if writes to database were async?
- Read-Through: What if cache handled reads automatically?
- Cache-Aside vs Write-Through: When to use which?

Dive deeper into Write-Through:
- Implementing Write-Through with Redis
- Write-Through with TTL vs. no expiration
- Handling Write-Through failures gracefully
- Write-Through in distributed systems
- Optimizing Write-Through for your access patterns

Advanced topics:
- Write-Through with write batching
- Combining Write-Through with cache warming
- Multi-tier Write-Through caches
- Write-Through cache sizing strategies
- Monitoring and metrics for Write-Through

Real-world architectures:
- AWS ElastiCache Write-Through patterns
- Redis Write-Through implementation
- Database + cache synchronization strategies
- Ensuring transactional consistency with Write-Through

Remember: Write-Through trades write performance for consistency guarantees. Choose it when consistency matters more than write speed, and when reads vastly outnumber writes! 🎯
