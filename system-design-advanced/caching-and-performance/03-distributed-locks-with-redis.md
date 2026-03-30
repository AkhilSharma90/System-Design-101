---
slug: distributed-locks-with-redis
title: Distributed Locks With Redis
readTime: 20 min
orderIndex: 3
premium: false
---


# Distributed Locks with Redis

> Audience: advanced distributed systems engineers building real services.
>
> Goal: understand how Redis-based distributed locks actually behave under failures, what guarantees you do/don't get, and how to design safer alternatives.

---

## What you'll build in your head

By the end, you should be able to:

- Explain what a Redis lock guarantees (and what it cannot) in a distributed environment.
- Evaluate the safety of common patterns: `SET NX PX`, renewals, fencing tokens, Redlock.
- Reason about failure scenarios: client pauses, GC, partitions, clock drift, replica lag, failover, eviction.
- Choose between lock, lease, idempotency, queue, single-writer, or fencing.

[IMAGE: Flow map diagram: Basics -> Single-node Redis lock -> Failure modes -> Renewal/watchdog -> Fencing tokens -> Failover/replication -> Redlock -> Alternatives -> Decision matrix -> Synthesis challenge]

---

##  The ground rules (don’t skip)

A Redis “distributed lock” is a time-bounded lease stored in Redis. Its behavior depends on assumptions:

- Network is unreliable: messages can be delayed, duplicated, reordered, or dropped; timeouts are ambiguous.
- Processes pause: GC, CPU starvation, VM suspend, container freeze, node clock adjustments.
- Redis availability: Redis can restart, fail over, or lose volatile state.
- Replication is usually async: acknowledged writes may not survive failover unless you pay extra latency.
- Clocks are not perfect: Redis TTL uses Redis’s clock; clients use their own clocks for timeouts and (in Redlock) elapsed-time checks.

### CAP framing (what you’re really choosing)
- Under a partition, you must choose between:
  - Availability (clients proceed) vs
  - Consistency (single owner / single writer).

Redis locks are typically used to keep systems available and “mostly single-owner,” not to provide strict mutual exclusion under partitions.

[IMAGE: CAP triangle annotated: Redis locks typically bias toward A+P; strict C under partitions requires coordination systems + stronger invariants]

---

##  Challenge 1: The coffee shop espresso machine

### Scenario
You run a busy coffee shop. There’s one espresso machine. Two baristas (two app instances) might try to use it at the same time. You need a rule:

- Only one barista can use the machine at a time.
- If a barista disappears mid-shot, someone else should eventually be able to use it.
- No double-pulling shots for the same order.

In distributed systems terms:

- Espresso machine = shared resource (row, file, job, inventory bucket).
- Baristas = distributed workers.
- “Use machine” = critical section.

### Interactive question (pause and think)
If you had a shared notebook (Redis) where a barista can write “I’m using the machine,” what could go wrong?

1) The barista writes the note and then collapses.
2) The note gets erased by accident.
3) Another barista can’t see the note temporarily.
4) Two baristas both think they wrote first.

Which of these are possible in a distributed system? Pause and think.

### Progressive reveal: answer
All four can happen depending on failure mode:

- (1) process crash / container kill
- (2) Redis restart / failover / key eviction / bug / operator error
- (3) network partition / packet loss / timeouts
- (4) race conditions if the locking protocol is wrong

A distributed lock is not “mutual exclusion in the universe.” It’s a protocol that provides some mutual exclusion properties under assumptions.


Challenge question: What assumptions do you think Redis-based locks rely on? (Network? Redis availability? Clocks?)

---

##  Lock vs Lease vs Ownership

### Scenario
A restaurant gives you a pager that lets you occupy a table. But the pager expires if you don’t check in.

- A lock sounds like: “This is mine until I unlock it.”
- A lease is: “This is mine until time T, unless renewed.”

Redis locks are almost always leases.

### Interactive question (pause and think)
When you call `SET key value NX PX 30000`, what are you really getting?

A) A lock that lasts until you delete it
B) A lease that lasts 30 seconds
C) A guarantee nobody else can ever enter the critical section
D) A guarantee Redis will never forget

Pause.

### Progressive reveal: answer
Correct: B.

Redis locks are time-bounded leases. Time is part of the correctness story.

Challenge question: If time is part of correctness, what happens when a process is paused for longer than the TTL?

[IMAGE: Lease timeline: acquire -> pause -> TTL expires -> another owner acquires -> original resumes]

---

##  The simplest Redis lock: `SET NX PX`

### Scenario
You have N workers processing jobs. Each job has a unique ID. You want at most one worker to process a job.

You choose:

- Lock key: `lock:job:{id}`
- Lock value: random token unique to the lock owner
- TTL: e.g., 30 seconds

### [CODE: Python, basic Redis lock acquire/release using SET NX PX and value check]

```python
# Redis lease lock (SET NX PX) with safe release (Lua check-and-del)
# NOTE: This provides a lease, not a hard mutual exclusion guarantee.
import os
import redis

R = redis.Redis(host="redis", port=6379, decode_responses=True, socket_timeout=1)
RELEASE_LUA = """
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end
"""
_release = R.register_script(RELEASE_LUA)


def with_redis_lock(lock_key: str, ttl_ms: int, fn):
    token = os.urandom(16).hex()
    try:
        ok = R.set(lock_key, token, nx=True, px=ttl_ms)
        if not ok:
            return False  # someone else owns it
        fn()  # critical section (must be bounded / idempotent)
        return True
    except (redis.TimeoutError, redis.ConnectionError) as e:
        # Important: timeouts are ambiguous; you may or may not have acquired.
        raise RuntimeError(f"redis error while operating on lock {lock_key}: {e}")
    finally:
        # Best-effort release; TTL provides liveness.
        try:
            _release(keys=[lock_key], args=[token])
        except (redis.TimeoutError, redis.ConnectionError):
            pass

# Usage example: with_redis_lock("lock:job:42", 30_000, lambda: do_work())
```

```javascript
// Redis lease lock (SET NX PX) with safe release (Lua check-and-del)
// NOTE: This provides a lease, not a hard mutual exclusion guarantee.
import { createClient } from "redis";
import crypto from "crypto";

const client = createClient({
  url: process.env.REDIS_URL,
  socket: {
    // bounded reconnect backoff
    reconnectStrategy: (retries) => Math.min(retries * 50, 1000),
  },
});
await client.connect();

const RELEASE_LUA =
  "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";

export async function withRedisLock(lockKey, ttlMs, fn) {
  const token = crypto.randomBytes(16).toString("hex");
  try {
    const ok = await client.set(lockKey, token, { NX: true, PX: ttlMs });
    if (ok !== "OK") return false;
    await fn();
    return true;
  } catch (err) {
    // Important: timeouts are ambiguous; you may or may not have acquired.
    throw new Error(`redis error for lock ${lockKey}: ${err?.message || err}`);
  } finally {
    try {
      await client.eval(RELEASE_LUA, { keys: [lockKey], arguments: [token] });
    } catch {
      /* best-effort */
    }
  }
}

// Usage example: await withRedisLock("lock:job:42", 30000, async () => doWork());
```

### Interactive question: Why the token?
Suppose you skip the token and just `DEL lock_key` when done.

Pause and think: Under what conditions could you delete someone else’s lock?

### Progressive reveal: answer
If your lease expires (TTL) and another worker acquires the lock, and then your worker finally finishes and deletes the key, you just released the new owner’s lock.

This is the classic “unlocking someone else’s lock” bug.

Release must be ownership-aware. Use a unique token and an atomic check-and-delete.

Challenge question: Is token-checking sufficient to guarantee mutual exclusion? (Hint: consider long pauses.)

---

##  Failure scenarios you must simulate (mentally)

Distributed locks fail in the cracks between:

- Client (your process)
- Network
- Redis server(s)
- Time (TTL)
- Failover system (Sentinel/Cluster)

We’ll play through failures like a delivery service where drivers (clients) and dispatch (Redis) can lose contact.

[IMAGE: Timeline diagram: Client A acquires lock with TTL -> pauses -> TTL expires -> Client B acquires -> A resumes and writes to shared resource]

---

##  Challenge 2: The “paused barista” bug

### Scenario
Worker A acquires lock with TTL=10s. It starts updating inventory in a DB transaction.

At t=2s, A hits a GC pause or VM freeze for 15 seconds.

At t=10s, lock expires.

At t=11s, Worker B acquires lock and also updates inventory.

At t=17s, A wakes up and continues its “critical section.”

### Interactive question (pause and think)
Does token-based unlock prevent double-updates here?

- Yes, because A can’t delete B’s lock
- No, because A can still perform the critical section after losing the lease

Pause.

### Progressive reveal: answer
Correct: No.

Token-based unlock prevents incorrect unlock, but it doesn’t stop A from continuing to act after its lease expired.

A Redis lease only controls who may enter. It does not physically stop a paused process from continuing later.

Challenge question: What additional mechanism can prevent “stale owners” from committing side effects?

---

##  The missing ingredient: Fencing tokens

### Scenario (restaurant ticket numbers)
A restaurant gives each table a ticket number. The kitchen only accepts orders with a ticket number that is higher than any previously seen.

Even if an old waiter shows up late with an old ticket, the kitchen rejects it.

That’s a fencing token.

### What is a fencing token?
A monotonically increasing number associated with the lock acquisition. The protected resource enforces ordering:

- Each lock acquisition returns token t.
- Resource accepts operations only if t is greater than last accepted token.

In practice:

- Use Redis `INCR` on a counter to generate token.
- Store last token in the DB row, storage metadata, or service state.

[IMAGE: Fencing diagram: lock acquisition returns token t; downstream resource stores last_t; rejects stale t]

### [CODE: Lua, acquire lock + fencing token]

```lua
-- KEYS[1] = lock key
-- KEYS[2] = token counter key
-- ARGV[1] = lock owner random value
-- ARGV[2] = ttl ms
-- returns: {acquired(0/1), fencing_token}

if redis.call('set', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
  local t = redis.call('incr', KEYS[2])
  return {1, t}
else
  return {0, 0}
end
```

```python
# Acquire-with-fencing-token via Lua (atomic SET NX PX + INCR)
import os
import redis

R = redis.Redis(host="redis", port=6379, decode_responses=True, socket_timeout=1)
ACQUIRE_FENCE_LUA = """
if redis.call('set', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
  local t = redis.call('incr', KEYS[2])
  return {1, t}
else
  return {0, 0}
end
"""
_acquire = R.register_script(ACQUIRE_FENCE_LUA)


def acquire_with_fence(lock_key: str, fence_key: str, ttl_ms: int = 30_000):
    token = os.urandom(16).hex()
    try:
        acquired, fence = _acquire(keys=[lock_key, fence_key], args=[token, str(ttl_ms)])
        return (token, int(fence)) if int(acquired) == 1 else (None, 0)
    except (redis.TimeoutError, redis.ConnectionError) as e:
        raise RuntimeError(f"redis error acquiring fence lock {lock_key}: {e}")

# Usage example: token,fence = acquire_with_fence("lock:acct:7","fence:acct:7")
```

```javascript
// Acquire-with-fencing-token via Lua (atomic SET NX PX + INCR)
import { createClient } from "redis";
import crypto from "crypto";

const r = createClient({ url: process.env.REDIS_URL });
await r.connect();

const ACQUIRE_FENCE_LUA = `
if redis.call('set', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
  local t = redis.call('incr', KEYS[2])
  return {1, t}
else
  return {0, 0}
end`;

export async function acquireWithFence(lockKey, fenceKey, ttlMs = 30000) {
  const token = crypto.randomBytes(16).toString("hex");
  try {
    const [acquired, fence] = await r.eval(ACQUIRE_FENCE_LUA, {
      keys: [lockKey, fenceKey],
      arguments: [token, String(ttlMs)],
    });
    return acquired === 1 ? { token, fence: Number(fence) } : null;
  } catch (err) {
    throw new Error(`redis error acquiring fence lock ${lockKey}: ${err?.message || err}`);
  }
}

// Usage example: const h = await acquireWithFence("lock:acct:7","fence:acct:7");
```

### Interactive question (pause and think)
If you have fencing tokens, do you still need TTL?

- A) No, fencing tokens replace TTL
- B) Yes, TTL is still needed to recover from crashes

Pause.

### Progressive reveal: answer
Correct: B.

TTL is for liveness (eventual progress). Fencing tokens are for safety (reject stale owners).

TTL gives liveness; fencing gives safety against paused or partitioned clients.

Challenge question: Where should fencing be enforced in Redis, in your app, or in the downstream system?

---

##  Enforcing fencing: “the kitchen must check the ticket”

### Scenario
You protect a shared resource: e.g., updating a row in Postgres.

You add a column:

- `fence_token BIGINT`

Update uses conditional write:

- Only succeed if incoming token > stored token.

### [CODE: SQL, enforcing fencing token on update]

```sql
-- Suppose resource is "inventory_item" with columns:
-- id, quantity, fence_token

-- Worker with token :t wants to decrement quantity by 1
UPDATE inventory_item
SET quantity = quantity - 1,
    fence_token = :t
WHERE id = :id
  AND fence_token < :t;

-- If rowcount == 0, your token is stale; abort.
```

### Production note: make the invariant explicit
- Initialize `fence_token` to 0.
- Consider `CHECK (fence_token >= 0)`.
- If multiple operations exist, ensure every write path enforces the fence.

###
“If I use Redis locks, I don’t need conditional updates.”

Reality: Redis locks control entry best-effort. Your database is where correctness often must be enforced.

Put the strongest correctness check closest to the state you’re protecting.

Challenge question: What if the protected resource is not a DB row but an external API call? How could fencing work there?

---

##  Decision game: Which statement is true?

Pick the true statement(s). Pause before reading the answer.

1) If a client holds a Redis lock with TTL, mutual exclusion is guaranteed as long as Redis is up.
2) Token-based unlock prevents deleting another client’s lock.
3) Without fencing, a paused client can cause split-brain behavior even if unlock is correct.
4) Redlock guarantees safety even under network partitions.

Pause and think: which are true?

### Progressive reveal: answer
- (1) False: pauses/partitions can violate mutual exclusion at the resource.
- (2) True: if implemented atomically.
- (3) True: stale owners can act.
- (4) False as a blanket claim: Redlock is a probabilistic mitigation under assumptions; it does not remove the need for fencing for correctness-critical side effects.

Most Redis lock correctness debates are really about what your system assumes about time, partitions, and failover.

Challenge question: What is your service’s tolerance for duplicate critical section execution? (Never? Rare? Acceptable with idempotency?)

---

##  Renewals (“watchdog”) and why they don’t solve everything

### Scenario (parking meter)
You park with a 30-second meter (TTL). You can keep feeding it coins (renewal) so you don’t get towed.

A common pattern:

- Acquire lock with TTL=30s
- Background thread renews every 10s if still owner

### [CODE: Python, renewal watchdog sketch]

> Correction: the original heading said “Java”; this is Python.

```python
# Renewal watchdog (thread) that signals work to stop on lease loss
import os
import threading
import redis

R = redis.Redis(host="redis", port=6379, decode_responses=True, socket_timeout=1)
RENEW_LUA = """
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('pexpire', KEYS[1], ARGV[2])
else
  return 0
end
"""
RELEASE_LUA = """
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end
"""
_renew = R.register_script(RENEW_LUA)
_release = R.register_script(RELEASE_LUA)


def run_with_watchdog(lock_key: str, ttl_ms: int, work_fn):
    token = os.urandom(16).hex()

    # Acquire
    ok = R.set(lock_key, token, nx=True, px=ttl_ms)
    if not ok:
        return False

    stop = threading.Event()
    lost = threading.Event()

    def watchdog():
        # renew every ~ttl/3
        interval_s = max(0.001, ttl_ms / 3000.0)
        while not stop.wait(interval_s):
            try:
                if int(_renew(keys=[lock_key], args=[token, str(ttl_ms)])) != 1:
                    lost.set()
                    stop.set()
            except (redis.TimeoutError, redis.ConnectionError):
                # safest default: assume lease may be lost
                lost.set()
                stop.set()

    t = threading.Thread(target=watchdog, name="redis-lock-watchdog", daemon=True)
    t.start()

    try:
        work_fn(lost)  # work_fn should periodically check lost.is_set()
        return not lost.is_set()
    finally:
        stop.set()
        t.join(timeout=1)
        try:
            _release(keys=[lock_key], args=[token])
        except (redis.TimeoutError, redis.ConnectionError):
            pass

# Usage: run_with_watchdog("lock:job:42", 30000, lambda lost: do_work(lost))
```

```javascript
// Renewal watchdog (timer) that aborts work on lease loss
import { createClient } from "redis";
import crypto from "crypto";

const r = createClient({ url: process.env.REDIS_URL });
await r.connect();

const RENEW_LUA =
  "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('pexpire',KEYS[1],ARGV[2]) else return 0 end";
const RELEASE_LUA =
  "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";

export async function runWithWatchdog(lockKey, ttlMs, workFn) {
  const token = crypto.randomBytes(16).toString("hex");
  const ok = await r.set(lockKey, token, { NX: true, PX: ttlMs });
  if (ok !== "OK") return false;

  let lost = false;
  const interval = Math.max(1, Math.floor(ttlMs / 3));

  const timer = setInterval(async () => {
    try {
      const renewed = await r.eval(RENEW_LUA, {
        keys: [lockKey],
        arguments: [token, String(ttlMs)],
      });
      if (renewed !== 1) lost = true;
    } catch {
      lost = true;
    }
  }, interval);

  try {
    await workFn(() => lost);
    return !lost;
  } finally {
    clearInterval(timer);
    try {
      await r.eval(RELEASE_LUA, { keys: [lockKey], arguments: [token] });
    } catch {
      /* best-effort */
    }
  }
}

// Usage: await runWithWatchdog("lock:job:42", 30000, async (isLost) => { while(!isLost()) await step(); });
```

### Interactive question (pause and think)
Does renewal prevent the paused barista bug?

- A) Yes, renewal keeps the lock alive
- B) No, because if the process is paused, renewal thread is paused too

Pause.

### Progressive reveal: answer
Correct: B.

If the JVM pauses, the watchdog pauses. If the host is frozen, everything pauses. The lease can still expire.

Renewal reduces accidental expiry during normal operation, but cannot eliminate expiry under stop-the-world pauses or partitions.

### Production insight: what renewal is good for
- Reduces expiry during normal latency spikes.
- Lets you keep TTL small (faster crash recovery) while supporting longer work.

### Production insight: what renewal cannot guarantee
- Safety under long pauses.
- Safety under partitions.
- Safety if Redis loses the key.

Challenge question: If you rely on renewal, what is the worst-case pause you must tolerate? How will you detect you lost the lease?

---

##  Redis failover: when the ground moves under your lock

### Scenario (manager swap)
A manager (Redis primary) keeps the reservation book. A backup manager (replica) is copying notes.

If the manager faints, the backup takes over.

But what if the backup didn’t see the latest reservation note yet?

### The core issue
With Redis replication:

- Replication is typically asynchronous.
- A lock write may be acknowledged by the primary but not yet replicated.
- Failover can promote a replica missing that write.

Result: a lock can disappear, allowing another client to acquire it.

[IMAGE: Failover timeline: Client A SET NX PX on primary, ACK -> replication lag -> primary dies -> replica promoted without key -> Client B acquires lock]

### Interactive question (pause and think)
If you use a single Redis primary with replicas and Sentinel, is a lock safe across failover?

- A) Yes, because Redis is consistent
- B) Not necessarily, because acknowledged writes may be lost on failover

Pause.

### Progressive reveal: answer
Correct: B.

With async replication, failover can violate the assumption “if I got OK, the lock exists.”

### Mitigations (and their costs)
1) `WAIT numreplicas timeout` after acquiring
   - Improves durability of the lock write by waiting for replication acknowledgements.
   - Costs: extra latency; still not a perfect guarantee under partitions/timeouts.

2) `min-replicas-to-write` / `min-replicas-max-lag`
   - Prevents writes when replication health is poor.
   - Costs: reduces availability during replica issues (CAP trade-off).

3) Dedicated Redis for locks with persistence
   - AOF (`appendonly yes`) can help survive restarts.
   - Costs: write amplification; still doesn’t solve failover anomalies by itself.

4) Prefer fencing + downstream CAS
   - Even if the lock disappears, stale owners can’t commit.

Challenge question: Which knob would you turn first in production: `WAIT` or fencing? Why?

---

##  Redis Cluster and sharding: one lock, one shard

### Scenario (multiple reservation books)
Now the coffee shop has multiple reservation books (shards). Your lock key maps to one book.

That’s fine for a single lock key. But multi-key atomicity becomes hard.

### Key points
- Redis Cluster hashes keys to slots.
- `SET NX PX` is per-key, so it works on a single slot.
- Multi-resource locking (lock A and lock B) can deadlock or require ordering.

[IMAGE: Cluster slots diagram: lock keys map to slots; multi-key ops require same hash tag or separate coordination]

### Production guidance for multi-resource locking
- Prefer avoiding multi-lock designs.
- If you must:
  - Use a global ordering of lock keys.
  - Use try-lock + bounded retries + jitter.
  - Consider a single-writer architecture (consistent hashing) instead.


Redis makes single-key atomic operations easy; multi-key coordination is where complexity explodes.


Challenge question: If you need to lock multiple resources, what strategy would you use? (Ordering? Try-lock + backoff? Transactional system instead?)

---

##  Challenge 3: Protecting a critical section vs protecting correctness

### Scenario
You want to ensure “only one worker sends a billing charge for invoice 123.”

You add a Redis lock around the charge call.

### Interactive question (pause and think)
Is a lock the right tool?

Possible alternatives:

- Idempotency key with payment provider
- DB unique constraint: “invoice charged once”
- Outbox pattern + single consumer
- Redis lock

Which would you prefer for billing? Pause.

### Progressive reveal: answer
For billing, idempotency + durable state constraint is usually stronger than a volatile lock.

Redis locks can help reduce concurrency, but correctness should not depend solely on them.


Locks are often a performance/concurrency optimization, not the ultimate correctness mechanism.


Challenge question: Name one domain where a Redis lock is a reasonable primary mechanism (hint: ephemeral leader election for cache warming).

---

##  Common misconceptions (and the uncomfortable truth)

### Misconception 1: “If I got the lock, I’m safe.”
Reality: you’re safe only if:

- your lease doesn’t expire
- Redis doesn’t lose the key
- you can detect lease loss
- your critical section is bounded
- downstream state rejects stale operations (fencing)

### Misconception 2: “TTL makes it safe.”
Reality: TTL makes it eventually available. It doesn’t prevent stale owners from acting.

### Misconception 3: “Redlock solves everything.”
Reality: Redlock is a specific algorithm with assumptions. It may reduce some failover risks but introduces others (latency, partial quorums, clock assumptions). Many production teams prefer single Redis + fencing or a system designed for coordination (ZooKeeper/etcd/Consul).

### Misconception 4: “Redis is fast so locks are cheap.”
Reality: contention + retries can create thundering herds and amplify latency. Locks can become a hidden global bottleneck.

The hardest part is not `SET NX PX`; it’s designing the system behavior when the lock lies.

Challenge question: What is your system’s plan when two owners act concurrently? Can you tolerate it, detect it, or roll it back?

---

##  Redlock: what it tries to fix, and what it assumes

### Scenario (five independent coffee shops)
Instead of one reservation book, you keep five independent books in five separate shops (independent Redis masters). To reserve the espresso machine, you must get a reservation in a majority of shops quickly.

That’s the intuition behind Redlock.

### The algorithm (high-level)
- You have N independent Redis masters (commonly 5).
- To acquire:
  1. Record start time.
  2. Try `SET lock value NX PX ttl` on each master.
  3. If you acquired on at least quorum (e.g., 3/5) and elapsed time < ttl, you consider lock acquired.
  4. Otherwise, release any partial locks and retry after random backoff.
- To release: delete the key (with token check) on all masters.

[IMAGE: Redlock diagram: client attempts N masters; quorum success; partial release on failure]

### Interactive question (pause and think)
What failure is Redlock trying to address?

A) Client pauses
B) Single Redis failover losing acknowledged writes
C) Network partitions between clients and Redis
D) All of the above

Pause.

### Progressive reveal: answer
Mostly B (and some aspects of C). It does not fundamentally solve A without fencing.

### The controversial part (be precise)
Redlock relies on assumptions like:

- Independent failures among Redis masters
- Bounded clock drift and bounded message delays relative to TTL
- Clients behaving correctly (releasing partial locks)

Critiques (not exhaustive):

- In asynchronous networks with partitions, time-based leases cannot provide strong safety without additional constraints.
- If a client is paused, it may believe it holds a lock that has expired; Redlock doesn’t stop it from acting.
- Operational complexity: N masters, cross-AZ latency, partial failure handling.

Redlock can reduce the probability of certain failover anomalies but does not eliminate the need for fencing if side effects must be correct.

Challenge question: Under what conditions would you accept Redlock? (Think: low-stakes cache rebuild vs financial transfer.)

---

##  Matching exercise: choose the right tool

Match the problem to the preferred coordination primitive.

### Problems
1) Ensure only one service instance runs a periodic cache refresh job.
2) Ensure an invoice is charged at most once.
3) Ensure only one worker processes a Kafka partition.
4) Ensure a shared file in object storage is updated without lost updates.
5) Ensure only one deployment controller acts as leader.

### Tools
A) Redis lock (lease)
B) DB uniqueness constraint / transactional write
C) Kafka consumer group partition assignment
D) Fencing token + conditional write
E) etcd/ZooKeeper lease + watch

### Interactive question (pause and think)
Write your mapping.

### Progressive reveal: answer
A reasonable mapping:

1 -> A (or E) depending on criticality
2 -> B (plus idempotency)
3 -> C
4 -> D
5 -> E

Use locks for coordination, but use transactional systems for state correctness.

Challenge question: Which of these choices changes if you are in a multi-region active-active architecture?

---

##  Designing a Redis lock you can live with

### Scenario
You still want Redis locks because:

- You need low-latency coordination.
- You can tolerate rare duplicates.
- You’ll add downstream checks.

Let’s design a “production-grade enough” lock.

### Design checklist
1) Acquire atomically: `SET key token NX PX ttl`.
2) Release safely: Lua check-and-del.
3) Renew carefully: Lua check-and-pexpire.
4) Detect lease loss: if renewal fails, stop work.
5) Bound critical section time: keep work < TTL when possible.
6) Use fencing for side effects: monotonic tokens + conditional writes.
7) Backoff + jitter on contention.
8) Metrics: lock acquisition latency, contention rate, renewal failures.

[IMAGE: Checklist diagram: Acquire -> Maintain -> Prove ownership -> Side effects with fencing -> Release]

### [CODE: Lua, renew-if-owner]

```lua
-- KEYS[1] = lock key
-- ARGV[1] = token
-- ARGV[2] = ttl ms

if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('pexpire', KEYS[1], ARGV[2])
else
  return 0
end
```

```python
# Renew-if-owner (atomic) and surfacing lease-loss to caller
import redis

R = redis.Redis(host="redis", port=6379, decode_responses=True, socket_timeout=1)
RENEW_LUA = """
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('pexpire', KEYS[1], ARGV[2])
else
  return 0
end
"""
_renew = R.register_script(RENEW_LUA)


def renew_or_raise(lock_key: str, token: str, ttl_ms: int) -> None:
    try:
        ok = int(_renew(keys=[lock_key], args=[token, str(ttl_ms)]))
    except (redis.TimeoutError, redis.ConnectionError) as e:
        # Conservative choice: treat as lease loss unless you can prove otherwise.
        raise RuntimeError(f"redis error renewing {lock_key}: {e}")
    if ok != 1:
        raise PermissionError(f"lost lease for {lock_key}; stop side effects")

# Usage: renew_or_raise("lock:job:42", token, 30000)
```

```javascript
// Renew-if-owner (atomic) and surfacing lease-loss to caller
import { createClient } from "redis";

const r = createClient({ url: process.env.REDIS_URL });
await r.connect();

const RENEW_LUA =
  "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('pexpire',KEYS[1],ARGV[2]) else return 0 end";

export async function renewOrThrow(lockKey, token, ttlMs) {
  try {
    const ok = await r.eval(RENEW_LUA, {
      keys: [lockKey],
      arguments: [token, String(ttlMs)],
    });
    if (ok !== 1) throw new Error("lost lease");
  } catch (err) {
    const msg = err?.message || String(err);
    throw new Error(`cannot renew ${lockKey}: ${msg}`);
  }
}

// Usage: await renewOrThrow("lock:job:42", token, 30000)
```

The lock is a living contract: acquire, maintain, and prove ownership continuously.

### Production decision: transient renewal timeout
If renewal fails due to a transient Redis timeout, do you stop immediately or retry?

- Safer for correctness: stop immediately (assume you may have lost the lease).
- Safer for availability: retry briefly (but you must prevent side effects until ownership is re-proven).

A common compromise:
- On renewal error: pause side effects, attempt a bounded number of re-prove steps (`GET == token`), then either continue or abort.

---

##  Edge cases that bite (with progressive reveal)

### Edge case 1: The “timeout but succeeded” ambiguity

#### Scenario
Client sends `SET NX PX`. Network times out. Client doesn’t know if Redis applied it.

[IMAGE: Ambiguous timeout diagram: request sent -> timeout -> Redis may have applied -> client uncertain]

#### Interactive question (pause and think)
What’s the safe behavior?

A) Assume lock not acquired; retry immediately
B) Assume lock acquired; enter critical section
C) Query Redis to see whether your token is present

Pause.

#### Progressive reveal: answer
Best: C, but even that can be tricky if you generated a token and the request might not have reached Redis.

Common approach:

- Use a unique token per attempt.
- After timeout, do a `GET lockKey` and compare to your token.
- If equals, you acquired.
- Otherwise, treat as not acquired and retry with backoff.

Production note:
- Beware client libraries that automatically retry writes: you can accidentally acquire twice with different tokens.

Distributed systems often force you to handle “did it happen?” uncertainty.

Challenge question: How does this ambiguity change if you use connection pooling and retries at the driver layer?

---

### Edge case 2: Eviction and memory pressure

#### Scenario
Redis configured with eviction policy (e.g., `allkeys-lru`). Under memory pressure, Redis evicts keys, including locks.

[IMAGE: Eviction diagram: memory pressure -> lock key evicted -> new owner acquires]

#### Interactive question (pause and think)
If a lock key is evicted, what happens?

- A) Nothing, lock is special
- B) Lock disappears; another client can acquire

Pause.

#### Progressive reveal: answer
Correct: B.

Mitigations:

- Don’t run locks on an evicting cache instance.
- Use a dedicated Redis with `maxmemory-policy noeviction`.
- Monitor memory and eviction counters.

A lock stored in a cache that can forget is not a lock; it’s a suggestion.

Challenge question: If you must share Redis with cache workloads, what guardrails can you add?

---

### Edge case 3: Clock drift and TTL interpretation

Redis TTL is measured by Redis’s clock, but clients reason about time too (e.g., Redlock elapsed time).

[IMAGE: Clock drift diagram: client clock vs Redis clock; drift affects elapsed-time checks]

#### Interactive question (pause and think)
Where can clock drift hurt you?

- A) In Redlock’s “elapsed time < ttl” check
- B) In client-side timeouts and renew scheduling
- C) Both

Pause.

#### Progressive reveal: answer
Correct: C.

Time-based coordination assumes bounded drift and bounded pauses; your environment may not provide that.

Challenge question: What’s your maximum observed GC pause? How does it compare to your TTL?

---

## [SEARCH] Observability: know when your lock is lying

### What to measure
- Acquire attempts, success rate, contention
- Time to acquire (p50/p95/p99)
- Renewal success/failure counts
- Time spent holding lock
- “Lost lease” events (renewal failed or ownership not provable)
- Downstream fencing rejections (stale token)

[IMAGE: Dashboard mock: lock contention, renewal failures, fencing rejections correlated with GC pauses]

If you don’t measure lease loss, you will assume it never happens until it happens in the worst possible moment.

Challenge question: Which metric would you alert on first: renewal failures or fencing rejections? Why?

---

##  Real-world usage patterns (and safer variants)

### Pattern A: Leader election for non-critical tasks
- Example: one instance runs cache warmup.
- Redis lock is often acceptable.
- If two leaders run briefly, impact is low.

### Pattern B: Rate-limiting a global action
- Example: “send at most one email per user per minute.”
- Better: store last-send timestamp in DB or Redis with atomic check.
- Lock can help but atomic state update is stronger.

### Pattern C: Scheduling / cron in a cluster
- Use lock to ensure one scheduler instance runs a job.
- Make job idempotent.
- Track job runs in DB.

### Pattern D: Protecting a critical write
- Prefer DB transactions, uniqueness constraints, compare-and-set.
- If you use Redis lock, add fencing.

Redis locks are best when duplicates are tolerable and work is idempotent.

Challenge question: Name one operation in your system that is not idempotent. How would you protect it without relying on a lock?

---

##  Quiz: “Lock, lease, or queue?”

### Scenario
You have 100 workers processing image thumbnails. Only one should process a given image ID.

Options:

A) Redis lock per image ID
B) Put image IDs into a queue with exactly-once processing
C) Use a DB row with status transitions (NEW->PROCESSING->DONE) and conditional updates

### Interactive question (pause and think)
Which do you choose and why?

### Progressive reveal: answer
Often C is the most robust: state machine in durable storage with conditional transitions. A lock (A) may be used as optimization, but DB transition is the correctness anchor. Queue (B) can work, but “exactly once” is hard; you still typically need idempotency/state.

Durable state transitions beat ephemeral coordination for correctness.

Challenge question: If you choose C, do you still need Redis at all? What does Redis add?

---

##  Comparison table: Redis lock vs etcd/ZooKeeper vs DB CAS

| Property | Redis `SET NX PX` | Redis + fencing | etcd/ZooKeeper lease | DB conditional update (CAS) |
|---|---|---|---|---|
| Latency | Very low | Low-medium | Medium | Medium |
| Operational complexity | Low | Medium | Medium-high | Medium |
| Handles client pauses safely | No | Yes (if enforced downstream) | Better for coordination; still need idempotency/fencing for external side effects | Yes (for DB state) |
| Survives Redis failover anomalies | Risky | Risky (but safety via fencing) | Designed for coordination | N/A |
| Good for leader election | Often | Often | Yes | Sometimes |
| Good for protecting DB writes | Weak | Stronger | Stronger | Strong |
| Requires TTL/time assumptions | Yes | Yes | Yes (leases) | No (unless you add timeouts) |

The “best” lock depends on what you’re protecting and where correctness must be enforced.

Challenge question: Where does your system sit on the spectrum: coordination convenience vs correctness-critical?

---

##  Putting it together: a safer Redis locking recipe

### Scenario
You must prevent concurrent execution of “rebalance account X” across workers. Rebalance writes to DB.

### Recipe
1) Acquire lease in Redis with token.
2) Get fencing token (monotonic).
3) Perform DB updates with conditional fencing.
4) Renew lease periodically; if renewal fails, stop and roll back if possible.
5) Release lease with token check.

### [CODE: Python, end-to-end sketch with fencing]

> Correction: the original heading said “Go”; this is Python.

```python
# End-to-end: acquire lease+fence, run work, enforce fencing via callback
import os
import redis

R = redis.Redis(host="redis", port=6379, decode_responses=True, socket_timeout=1)
ACQUIRE_FENCE_LUA = """
if redis.call('set', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
  local t = redis.call('incr', KEYS[2])
  return {1, t}
else
  return {0, 0}
end
"""
RELEASE_LUA = """
if redis.call('get',KEYS[1])==ARGV[1] then
  return redis.call('del',KEYS[1])
else
  return 0
end
"""
_acquire = R.register_script(ACQUIRE_FENCE_LUA)
_release = R.register_script(RELEASE_LUA)


def run_fenced(lock_key: str, fence_key: str, ttl_ms: int, apply_side_effect):
    token = os.urandom(16).hex()
    try:
        acquired, fence = _acquire(keys=[lock_key, fence_key], args=[token, str(ttl_ms)])
        if int(acquired) != 1:
            return False

        # Side effects must enforce fencing (e.g., DB CAS update)
        apply_side_effect(int(fence))
        return True
    except (redis.TimeoutError, redis.ConnectionError) as e:
        raise RuntimeError(f"redis error in fenced flow: {e}")
    finally:
        try:
            _release(keys=[lock_key], args=[token])
        except (redis.TimeoutError, redis.ConnectionError):
            pass

# Usage: run_fenced("lock:acct:7","fence:acct:7",30000, lambda f: db_update_with_fence(f))
```

```javascript
// End-to-end: acquire lease+fence, run work, enforce fencing via callback
import { createClient } from "redis";
import crypto from "crypto";

const r = createClient({ url: process.env.REDIS_URL });
await r.connect();

const ACQUIRE_FENCE_LUA = `
if redis.call('set', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) then
  local t = redis.call('incr', KEYS[2])
  return {1, t}
else
  return {0, 0}
end`;
const RELEASE_LUA =
  "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";

export async function runFenced(lockKey, fenceKey, ttlMs, applySideEffect) {
  const token = crypto.randomBytes(16).toString("hex");
  try {
    const [acquired, fence] = await r.eval(ACQUIRE_FENCE_LUA, {
      keys: [lockKey, fenceKey],
      arguments: [token, String(ttlMs)],
    });
    if (acquired !== 1) return false;

    // Side effects must enforce fencing (e.g., DB CAS update)
    await applySideEffect(Number(fence));
    return true;
  } catch (err) {
    throw new Error(`fenced flow failed: ${err?.message || err}`);
  } finally {
    try {
      await r.eval(RELEASE_LUA, { keys: [lockKey], arguments: [token] });
    } catch {
      /* best-effort */
    }
  }
}

// Usage: await runFenced("lock:acct:7","fence:acct:7",30000,(f)=>dbUpdateWithFence(f));
```

Redis coordinates who should try; the DB decides who is allowed to win.

### Trade-off: retrying DB failures while holding the lock
If the DB update fails due to transient error, do you keep the lock and retry, or release and let others try?

- Keep lock and retry:
  - Pros: avoids stampede; preserves locality.
  - Cons: longer lock hold time; higher chance of TTL expiry; more renewal pressure.

- Release and let others try:
  - Pros: reduces lock hold time; can improve throughput under partial failures.
  - Cons: can amplify contention; requires idempotency/fencing to avoid duplicates.

A common production pattern:
- Retry briefly while holding lock (bounded), then release and rely on fencing + durable state.

---

##  When not to use Redis locks

### Avoid Redis locks when:
- Side effects are irreversible and non-idempotent (money movement, external emails without idempotency).
- You cannot add fencing/conditional writes.
- Your environment has long unpredictable pauses (heavy GC, noisy neighbors) relative to TTL.
- Redis is configured as an evicting cache.
- You need strong correctness across partitions.

### Prefer alternatives:
- DB transactions + unique constraints
- etcd/ZooKeeper for coordination
- Single-writer per shard (consistent hashing)
- Queues with idempotent consumers


Don’t use a coordination primitive as a substitute for a correctness primitive.


Challenge question: What would it take to make your operation idempotent so you don’t need a lock?

---

##  Final synthesis challenge: The delivery dispatch incident

### Scenario
You run a delivery platform. Each order must be assigned to exactly one driver.

Current design:

- Multiple “assignment workers” race.
- Each worker tries to lock `lock:order:{id}` in Redis with TTL=20s.
- The worker that gets the lock calls `AssignDriver(orderId, driverId)` in a downstream service.

Incident:

- During a Redis failover, some lock keys are lost.
- Two workers assign two different drivers to the same order.
- Both drivers show up.

[IMAGE: Incident timeline: failover loses lock -> two workers proceed -> conflicting assignments]

### Your task
Design a safer system. You may still use Redis, but you must ensure correctness.

#### Step 1 - Pause and think
What is the correctness invariant?

- A) At most one worker holds the lock
- B) At most one driver assignment is accepted for an order

Pause.

#### Progressive reveal: answer
Correct: B.

Locks are a means, not the invariant.

#### Step 2 - Choose your architecture (decision game)
Which approach would you pick?

1) Keep Redis lock, increase TTL to 5 minutes.
2) Keep Redis lock, add renewal watchdog.
3) Add fencing token and enforce it in the downstream assignment service or DB.
4) Remove lock; use DB row state machine with `WHERE status='NEW'` conditional update.
5) Use Kafka partitioning by orderId so only one consumer processes an order.

Pause and choose.

#### Progressive reveal: one strong answer
A robust design is usually (3) + (4):

- DB enforces a single accepted assignment with a conditional update/unique constraint.
- Redis lock may still be used to reduce contention.
- If you keep Redis, add fencing so stale workers cannot override.

Kafka partitioning (5) can be part of it, but you still need idempotency/state for retries.

### Deliverable: write your own post-incident fix
Fill in the blanks:

- Correctness anchor: __________________________
- Redis role (if any): _________________________
- How stale workers are rejected: ______________
- How retries are handled: _____________________
- What you will monitor: _______________________

Distributed locks are coordination hints. Correctness lives in durable state + monotonic ordering (fencing) + idempotency.

---

##  Closing: a practical rule of thumb

If you remember only one thing:

> If executing the critical section twice is catastrophic, don’t rely on a Redis lock alone. Use fencing/conditional writes or a system built for coordination.

---
