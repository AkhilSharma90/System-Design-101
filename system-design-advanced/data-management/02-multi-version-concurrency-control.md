---
slug: multi-version-concurrency-control
title: Multi Version Concurrency Control
readTime: 20 min
orderIndex: 2
premium: false
---





# Multi-Version Concurrency Control (MVCC) an Interactive Deep Dive

> Audience: engineers who already know basic transactions, isolation levels, and replication, and want to reason rigorously about MVCC in distributed environments.

---

##  The cafe with one espresso machine and 200 customers

You run a busy coffee shop. There’s **one espresso machine** (the shared resource), but customers want two kinds of experiences:

- **Readers**: people who just want to *look at the menu* and decide (read-only).
- **Writers**: people placing orders that *change the queue* (writes).

If you force everyone into a single line with a single “lock” on the machine, the shop becomes a bottleneck.

**Your mission:** allow customers to read the menu and see the order queue without blocking the barista, while still ensuring orders are correct.

###
If you were designing the shop rules, which would you prefer?

A) Readers block writers so readers always see the “latest” queue.

B) Writers block readers so orders can be placed quickly.

C) Readers see a *consistent snapshot* of the queue as of some moment, and writers keep working.

Pick one. Don’t overthink.

---

###
Most distributed databases aiming for high concurrency pick **C**.

That’s the central promise of **MVCC**:

> **Readers don’t block writers; writers don’t block readers** (in the common case), because reads use *versions*.

Instead of “the row,” the database stores **multiple versions** of a row, each valid for some time interval.

**Clarification (important):** MVCC reduces *read/write* blocking, but it does **not** eliminate:
- writer/writer coordination (locks, intents, or validation)
- latches on in-memory structures (B-tree page latches, memtable locks)
- schema-change locks
- “maintenance” interference (vacuum/compaction, index backfills)

---

###
**MVCC = time travel for data.**

- Reads select the version valid at the reader’s snapshot time.
- Writes create a new version, typically without overwriting the old one immediately.
- Garbage collection eventually removes versions no longer needed.

---

##  A timeline per row

Imagine each row as a small timeline:

- Version V1 valid from time 10 to time 20
- Version V2 valid from time 20 to time 35
- Version V3 valid from time 35 to infinity

A transaction reading at time 27 picks V2; reading at time 12 picks V1.

[IMAGE: A horizontal timeline for one row with colored segments for versions, labeled with begin_ts and end_ts; overlay multiple reader snapshot times as vertical lines selecting different versions.]

###  Which statement is true?

1) MVCC guarantees serializable isolation by default.
2) MVCC guarantees readers never block writers.
3) MVCC stores multiple versions and uses a snapshot to decide which is visible.
4) MVCC eliminates the need for conflict detection.

Pause and pick.

---

###
**3 is always true.**

- (1) depends on the system and isolation level (Snapshot Isolation, Serializable, etc.).
- (2) is “mostly true,” but there are caveats: schema changes, vacuum/compaction, hot-spot updates, and some locking paths.
- (4) is false: MVCC shifts the problem; you still need write-write conflict detection and sometimes read-write validation.

###
MVCC is a **mechanism**. Isolation level is a **policy** implemented on top of it.

---

##  Build MVCC from scratch (conceptually)

You are designing a key-value store with transactions. Each key can be updated. You want:

- Consistent reads (no half-applied writes)
- High concurrency
- Reasonable performance under replication

###
What is the minimum metadata you need per version to answer:

- “Is this version visible to transaction T?”

---

###  Typical metadata
Common MVCC systems store something like:

- **begin timestamp** (often a *commit timestamp*) — when version becomes visible
- **end timestamp** (or deletion marker) — when version stops being visible
- **creator transaction id** (sometimes) — if version is uncommitted

In practice, implementations vary:

- PostgreSQL uses `xmin`/`xmax` transaction IDs plus commit status lookup.
- Many distributed systems use **commit timestamps** from a timestamp oracle or hybrid logical clock.
- Some systems store **write intents** (uncommitted versions) separately.

[IMAGE: A table showing versions with columns: key, value, begin_ts, end_ts, created_by_txn, committed?; plus arrows showing visibility checks.]

###
Visibility = **snapshot time** + **version validity interval** + **commit status**.

---

##  “MVCC means no locks”

MVCC reduces *read locks*, but many systems still use locks for:

- Write-write conflicts (two writers updating same row)
- Predicate locking / range locking (to avoid phantoms under serializable)
- Index structure protection (latches)
- Commit coordination (2PC/consensus)

###
If MVCC doesn’t lock reads, how do we prevent two writers from both committing updates to the same row?

---

###
You need **conflict detection**:

- **Pessimistic**: lock the row/key at write time.
- **Optimistic**: allow both to proceed but abort one at commit if it conflicts.

Many distributed MVCC databases do a hybrid:

- Lock or “intent” at write time to avoid wasted work.
- Validate at commit to ensure snapshot rules.

###
MVCC removes *read locks*, not the need for coordination.

---

##  The restaurant: receipts vs rewriting the menu

Non-MVCC (in-place updates) is like rewriting the menu board every time a dish changes. Customers reading the menu might catch it mid-erasure.

MVCC is like:

- Every time the menu changes, you **print a new menu sheet**.
- Customers who sat down earlier keep reading their sheet (snapshot).
- New customers get the latest sheet.

The kitchen doesn’t pause to let everyone re-read the menu.

---

##  MVCC visibility rules (core logic)

Let’s formalize the “menu sheet” rule.

A transaction `T` has a **snapshot** `S`.

A version `V` is visible to `T` if:

- `V.begin <= S` (it was committed before snapshot)
- and `V.end > S` (it wasn’t superseded/deleted before snapshot)
- and `V` is committed (or created by `T` itself)

**Clarification:** some systems use `begin_ts` as *commit_ts*; others store `created_ts` and a separate commit record. The visibility predicate is conceptually the same: “committed before snapshot and not overwritten before snapshot.”

###
Match each term to its role:

| Term | Role |
|------|------|
| Snapshot timestamp | A) When a version stops being visible |
| Begin/commit timestamp | B) The time used to decide what the transaction can see |
| End timestamp | C) When a version starts being visible |

Pause and match.

---

###
- Snapshot timestamp -> **B**
- Begin/commit timestamp -> **C**
- End timestamp -> **A**

---

##  Snapshot Isolation vs Serializable using MVCC

MVCC is often paired with **Snapshot Isolation (SI)**:

- Each transaction reads from a snapshot.
- Writes create new versions.
- At commit, system checks for write-write conflicts.

SI prevents many anomalies but not all.

###  Which anomaly can happen under SI?

1) Dirty read
2) Non-repeatable read
3) Write skew
4) Lost update (without extra checks)

Pause and pick all that apply.

---

###
Under Snapshot Isolation:

- Dirty reads: **No** (reads see committed snapshot)
- Non-repeatable reads: **No** (same snapshot throughout txn)
- Write skew: **Yes** (classic SI anomaly)
- Lost update: **Usually prevented** *if* the system checks write-write conflicts on the same row/key.

**Production nuance:** “lost update” can still happen at the *application level* when:
- you read row A, then write row B based on A (no direct write/write conflict)
- you do blind writes that overwrite without checking a predicate (depends on API)
- you rely on secondary indexes that are not transactionally consistent

###
“MVCC automatically gives serializable.”

Reality: SI is not serializable. Serializable MVCC typically requires additional machinery:

- Predicate/range locks
- SSI (Serializable Snapshot Isolation) with dependency tracking
- Commit-time validation with read sets

###
MVCC + SI = great performance and strong guarantees, but not full serializability.

---

##  Write skew: the story you should be able to tell at a whiteboard

Two doctors are on call. Rule: **at least one doctor must be on call**.

- Txn A reads: Dr. Alice=on, Dr. Bob=on
- Txn B reads: Dr. Alice=on, Dr. Bob=on

Then:

- Txn A sets Alice=off
- Txn B sets Bob=off

Both commit. End state: nobody on call.

Each transaction updated a different row, so no write-write conflict is detected. SI allows it.

[IMAGE: A dependency diagram showing Txn A and Txn B reading both rows from same snapshot, then writing disjoint rows, leading to invalid final state.]

###
How would you prevent this?

Pause and think.

---

###
- Use **serializable** isolation (SSI/predicate locking).
- Model invariant with a single row (forcing same-row conflict) — sometimes practical, sometimes not.
- Use explicit locks / `SELECT ... FOR UPDATE` on both rows.
- Use constraints enforced at commit with validation.

###
Write skew is an **invariant violation** caused by **concurrent reads + disjoint writes**.

---

##  MVCC on one node vs distributed

On a single machine, MVCC mostly answers:

- How do we assign timestamps?
- How do we store versions efficiently?
- How do we garbage collect?

In distributed systems, add:

- How do we get a **global order** (or enough ordering) for timestamps?
- How do we read a **consistent snapshot** across shards/replicas?
- How do we handle **failures** mid-commit?
- How do we keep **replicas** consistent with versions?

###
What’s the hardest part of distributed MVCC?

A) Storing multiple versions
B) Choosing an index structure
C) Coordinating timestamps and commit across nodes

---

###
**C** dominates complexity.

Storing versions is engineering. Distributed correctness is coordination.

###
Distributed MVCC is not “MVCC + networking.” It’s “MVCC + distributed commit + distributed time.”

---

##  Distributed MVCC design space

Common architectural choices:

1) **Central timestamp oracle** (some systems use a centralized allocator)
2) **Per-node clocks + Hybrid Logical Clocks (HLC)**
3) **Logical timestamps from consensus** (commit index / log position)

And for commits:

- **2PC** over shards
- **Consensus per transaction** (less common)
- **Deterministic ordering** (not MVCC-centric, but interacts)

###  Timestamp strategies

| Strategy | Pros | Cons | Typical fit |
|---|---|---|---|
| Central timestamp oracle | Simple global ordering; easy snapshot reads | Availability/latency bottleneck; needs HA + fencing | Percolator-style systems; many MVCC KV stores |
| HLC (physical+logical) | No single bottleneck; preserves causality-ish ordering | Requires clock monotonicity assumptions; still needs “safe time” frontiers | Geo-distributed systems |
| Consensus-derived order | Strong order tied to replication | Commit latency includes consensus; cross-shard still hard | Log-centric designs |

**Network/time assumptions (state them explicitly):**
- With HLC you assume bounded clock drift rate and monotonic clocks per node; you still cannot assume synchronized wall time.
- With TrueTime-like approaches you assume a bounded uncertainty interval and pay commit-wait.

---

##  The “snapshot across shards” problem

You have a database sharded by key:

- Shard 1 stores keys A–M
- Shard 2 stores keys N–Z

A transaction reads `K` from shard 1 and `T` from shard 2.

###
How do you guarantee the transaction sees a **single consistent snapshot** across both shards?

A) Each shard picks its own snapshot timestamp.
B) The coordinator chooses one snapshot timestamp `S`, and every shard reads at `S`.
C) Read from leaders only; followers are unsafe.

---

###
**B** is the core pattern: one snapshot timestamp `S` used everywhere.

But implementing B requires:

- A way to pick `S` such that shards can serve reads at `S`.
- Ensuring replicas have applied commits up to `S` (or can serve historical versions).

Key concept:

> **Read timestamp must be <= each shard’s “safe time”** (the time up to which the shard is sure it has all commits).

Different systems name this differently:

- safe timestamp
- resolved timestamp
- read frontier
- applied timestamp (careful: “applied” alone is not enough if there can be gaps)

[IMAGE: Two shards with their own safe_time lines; coordinator picks S that is <= min(safe_time_1, safe_time_2).]

###
A distributed snapshot is only as fresh as the **slowest shard’s safe time**.

---

## [COMMON MISCONCEPTION] “Just pick now() as snapshot time”

If nodes have skewed clocks, “now” is not globally meaningful.

Even with perfectly synchronized clocks, “now” does not ensure all shards have applied all commits up to “now.”

###
Think of each shard like a kitchen station. Even if the order was placed at 12:01, the dessert station might not have received the ticket yet. Serving a “snapshot at 12:01” requires every station to have processed tickets up to 12:01.

###
Snapshot time is not “wall time.” It’s “time for which the system has a completeness guarantee.”

---

##  MVCC + replication: leader/follower and read-your-writes

### Scenario
You write `x=5` to the leader. Immediately after, you read from a follower.

### [PAUSE AND THINK]
Should you see `x=5`?

- If you want **read-your-writes**, yes.
- If follower replication is async, maybe not.

In MVCC terms: your write created a new version with commit timestamp `c`. A follower can only serve reads at snapshot `S` if it has applied all commits up to `S`.

So to read your write:

- read from leader, or
- read from a follower whose applied/safe timestamp >= `c`, or
- use session guarantees with “minimum read timestamp” pinned to your last write.

[IMAGE: Leader with commit ts c; follower lagging with applied ts < c; read at S=c fails or returns old version.]

###
MVCC makes staleness explicit: a replica’s **applied frontier** limits which snapshots it can serve.

**Production insight:** if you offer follower reads, expose a client-visible token (e.g., `min_read_ts`) so services can enforce session consistency without always hitting leaders.

---

##  Distributed transaction commit with MVCC

A write transaction touches 3 shards. Each shard will store new versions.

Question: when do those versions become visible?

###
A) As soon as each shard writes its local version.
B) Only after all shards agree the transaction commits, using a commit protocol.
C) Immediately, but readers ignore them until a background process validates.

---

###
**B** is the usual correctness requirement.

Most systems implement:

- **Prepare**: write intents / provisional versions
- **Commit**: mark intents committed with a commit timestamp
- **Abort**: remove/ignore intents

This is typically **2PC** (two-phase commit) with a coordinator.

###
Coordinator crashes after some shards commit and others haven’t.

Without recovery, you risk:

- in-doubt transactions
- stuck intents blocking others
- inconsistent visibility

###
MVCC needs a **durable commit decision**. In distributed systems, that’s where 2PC/consensus enters.

**Distributed systems rigor (CAP framing):**
- Under a partition, a system that provides *atomic multi-shard transactions* must choose between:
  - **Consistency** (block/abort writes when quorum/coordination unavailable)
  - **Availability** (accept non-atomic outcomes or weaker semantics)
- Most “strong” MVCC systems choose **CP** for transactional writes across shards.

---

##  How “write intents” work (and why they exist)

A common MVCC trick: uncommitted writes are stored as **intents** (a provisional version).

- Readers at snapshot `S` ignore intents from other transactions.
- Writers that conflict can detect the intent and either wait, push, or abort.

This allows the system to:

- avoid exposing uncommitted data
- detect write-write conflicts early
- support recovery (intents can be resolved later)

[IMAGE: A key with committed versions and an uncommitted intent at the head; arrows showing reader skipping intent and writer encountering it.]

###  Intent handling
Which statement is true?

1) Intents are visible to all readers.
2) Intents can be treated as locks plus data.
3) Intents eliminate the need for commit protocol.

---

### [ANSWER]
**2** is true.

Intents are often “lock+value”: they both reserve the key and store the tentative new version.

**Clarification:** some engines store intents in a separate lock table (data in-place or in a provisional MVCC record). The semantics are the same.

---

##  Garbage collection: the delivery service’s storage room

Every new version is like keeping a copy of a delivery receipt.

- Great for auditing and for customers who started reading earlier.
- But the storage room fills up.

###
When can we safely delete old versions?

###
What condition must be true before deleting a version with end timestamp `E`?

---

###
A version can be collected when **no active transaction** can still read it.

In snapshot terms:

- Let `min_active_snapshot` be the minimum snapshot timestamp among all currently running transactions.
- Any version with `end_ts < min_active_snapshot` is not visible to any active transaction.

In distributed systems, computing `min_active_snapshot` is tricky:

- Transactions span nodes.
- Clients may disconnect.
- Coordinators may fail.

Systems use:

- leases
- heartbeats
- transaction record timeouts
- “closed timestamps” / “resolved timestamps” propagated via replication

[IMAGE: Multiple transactions with snapshot times; a GC line at min_active_snapshot; versions to the left are safe to delete.]

###
MVCC performance is often limited by **GC and long-running transactions**.

**Production insight:** enforce max transaction duration (or at least max *read snapshot age*) for OLTP paths; route long analytics to replicas / separate systems.

---

## MISCONCEPTION “MVCC makes long reads cheap”

Long read-only transactions can be expensive because they:

- keep `min_active_snapshot` low
- prevent GC from reclaiming old versions
- increase storage and compaction work
- can slow down reads and writes (more versions to scan; larger LSM levels)

MVCC trades lock contention for **version retention pressure**.

---

## MVCC and distributed indexes

Indexes must also be versioned or at least consistent with MVCC visibility.

Two broad strategies:

1) **Versioned index entries**
   - Each index entry points to a row version with timestamps.
   - Reads apply visibility.

2) **Single index entry + version chain in heap**
   - Index points to latest version; traverse chain to find visible version.

Distributed twist:

- Secondary indexes may be stored on different nodes than base data.
- Maintaining them atomically requires distributed transactions.

###
If secondary index updates are async, what anomaly might you see?

Pause and think.

---

###
You can see:

- missing rows (index not updated yet)
- phantom-like behavior
- inconsistent reads between index and base table

Systems either:

- update index and base in the same transaction (costly), or
- accept eventual consistency for indexes (documented semantics), or
- use changefeeds / backfill and tolerate anomalies.

###
MVCC correctness is easiest when **all read paths consult the same versioned truth**.

---

##  Read paths — point reads vs range scans

Point reads are easy: find the newest version <= snapshot.

Range scans are harder:

- Need to avoid phantoms under serializable.
- Need to be efficient when many versions exist.

###
Why are range scans especially tricky in distributed MVCC?

A) They require scanning more keys.
B) They interact with concurrent inserts/deletes (phantoms).
C) They require global locks.

---

###
**B** is the correctness issue; **A** is the performance issue.

Serializable isolation needs to ensure that if you scan “all orders with status=pending,” concurrent inserts don’t violate your assumptions.

Approaches:

- Predicate locking (range locks)
- SSI with read-write dependency tracking
- Deterministic execution / ordered commits

[IMAGE: A key range [k1,k9] scanned at snapshot S; concurrent insert k5 at S+1; show phantom issue.]

###
Phantoms are about **sets**, not individual rows.

---

##  MVCC under failures: what breaks first?

Distributed MVCC must handle:

- node crashes
- network partitions
- coordinator failures
- replica lag
- clock skew (if timestamps depend on time)

###  Coordinator crash during 2PC

Progressive reveal question:

If the coordinator dies after some shards have prepared intents, what should happen?

Pause and think.

---

###
Prepared intents are “in doubt.” Systems resolve them using:

- a **transaction record** stored durably (often replicated)
- coordinator recovery (new coordinator reads txn record)
- timeouts and transaction push/abort protocols

Without a durable transaction record, you can get stuck.

###
To make MVCC robust, the commit decision must be **recoverable** and **discoverable** by any participant.

**Production insight:** treat “intent resolution” as a first-class background subsystem with SLOs (age, backlog). If it falls behind, reads and GC degrade.

---

###  Network partition + timestamp oracle

If you rely on a centralized timestamp oracle and it becomes unreachable from some nodes:

- reads may continue at older snapshots (if safe)
- writes may block (can’t get commit timestamps)
- or the system might fail over to another oracle (needs fencing)

####  Fencing
Which statement is true?

1) If two timestamp oracles run concurrently, it’s fine as long as they’re fast.
2) Timestamp allocation must ensure monotonicity and uniqueness; failover needs fencing to prevent split-brain.
3) Timestamp oracle only affects performance, not correctness.

---

###
**2** is true.

###
Time assignment is part of correctness, not an optimization.

---

###  Replica applies commits out of order

In MVCC, versions have commit timestamps. If a replica applies commits out of timestamp order, can it still serve snapshot reads?

####
Is “apply order” important?

---

###
It depends on the system’s invariants:

- If replication log defines a total order, replicas apply in log order.
- If commits can arrive out of order, the replica must track a **resolved timestamp**: the highest timestamp such that all commits <= it are known/applied.

Only snapshots <= resolved_ts are safe.

###
Serving a snapshot requires a **completeness frontier**, not just “latest timestamp seen.”

---

##  MVCC is not free

###  MVCC vs locking (high level)

| Dimension | MVCC | Two-phase locking (2PL) |
|---|---|---|
| Reader/writer concurrency | High | Lower (read locks block writes or vice versa) |
| Write amplification | Higher (new versions) | Lower (in-place updates) |
| Storage | Higher; needs GC | Lower |
| Long transactions | Painful (version retention) | Blocks others (locks) |
| Distributed snapshot reads | Natural but needs safe time | Harder; often still needs coordination |
| Serializable isolation | Requires extra machinery | More direct (but can deadlock) |

###
If you’re building a write-heavy system with few reads, is MVCC always the best choice?

Pause.

---

###
Not necessarily. MVCC shines when:

- you have many reads
- you want low read latency
- you can tolerate GC/compaction costs

Write-heavy workloads may suffer due to:

- version churn
- compaction overhead
- hotspot contention (intents/locks)

###
MVCC optimizes for **read concurrency**, not pure write throughput.

---

##  Usage patterns

MVCC appears in many places, but with different trade-offs:

- PostgreSQL: MVCC with transaction IDs; vacuum; SSI optional.
- MySQL/InnoDB: MVCC with undo logs; consistent reads; gap locks for some isolation.
- Spanner: MVCC with TrueTime; external consistency; multi-version storage.
- CockroachDB: MVCC with HLC timestamps; intents; closed timestamps.
- TiDB/TiKV: MVCC over RocksDB; Percolator-like transactions; timestamp oracle.

###
Which systems use MVCC but still require locks?

Answer: most of them.

###
MVCC is ubiquitous because it composes well with replication and snapshots.

---

##  Implementing MVCC visibility (toy model)

**Corrections vs many toy examples:**
- Visibility must consider *commit status* and *own writes*.
- Version selection should not rely solely on `begin_ts` ordering if there can be multiple versions with same begin_ts (rare but possible with coarse timestamps); in real systems you’d include a tiebreaker (txn id / sequence).

```python
# Implementing MVCC visibility + version selection (point reads)
from dataclasses import dataclass
from typing import List, Optional

@dataclass(frozen=True)
class Version:
    value: str
    begin_ts: int
    end_ts: int  # exclusive; use a large number for "infinity"
    created_by: Optional[str] = None
    committed: bool = True

def mvcc_point_read(chain: List[Version], snapshot_ts: int, txn_id: str) -> Optional[str]:
    """Return the visible value at snapshot_ts for txn_id.

    Assumptions (toy model):
    - begin_ts/end_ts define the validity interval for committed versions.
    - uncommitted versions are "intents" and are only visible to their creator.
    - chain may be unsorted.
    """
    if snapshot_ts < 0:
        raise ValueError("snapshot_ts must be non-negative")

    # Newest-to-oldest by begin_ts. Real systems also use per-key sequence numbers.
    for v in sorted(chain, key=lambda x: x.begin_ts, reverse=True):
        # Ignore other transactions' intents.
        if not v.committed and v.created_by != txn_id:
            continue
        if v.begin_ts <= snapshot_ts < v.end_ts:
            return v.value
    return None

# Usage example
if __name__ == "__main__":
    INF = 2**63 - 1
    versions = [
        Version("v1", begin_ts=10, end_ts=20),
        Version("v2", begin_ts=20, end_ts=35),
        Version("intent_v3", begin_ts=35, end_ts=INF, created_by="t9", committed=False),
    ]
    print(mvcc_point_read(versions, snapshot_ts=27, txn_id="t1"))  # -> v2
    print(mvcc_point_read(versions, snapshot_ts=40, txn_id="t1"))  # -> v2 (intent ignored)
    print(mvcc_point_read(versions, snapshot_ts=40, txn_id="t9"))  # -> intent_v3 (own intent)
```

What this code should make you feel:

- MVCC visibility is simple locally.
- Distributed MVCC complexity comes from assigning timestamps and resolving intents.

---

##  Distributed snapshot read coordination (toy coordinator)

**Corrections/clarifications:**
- The coordinator must read *per-shard safe time* and choose `S <= min(safeTime)`.
- In real systems, `keys` are routed to the owning shard; broadcasting all keys to all shards is wasteful.
- You need to handle partial failures and retries carefully to avoid thundering herds.

```javascript
// Implementing distributed snapshot coordinator (min safeTime + parallel reads)
const net = require('net');

async function rpc(host, port, payload, timeoutMs = 500) {
  return await new Promise((resolve, reject) => {
    const sock = net.createConnection({ host, port }, () => sock.end(JSON.stringify(payload)));
    const t = setTimeout(() => { sock.destroy(); reject(new Error('rpc timeout')); }, timeoutMs);
    let buf = '';
    sock.on('data', d => (buf += d.toString('utf8')));
    sock.on('error', err => { clearTimeout(t); reject(err); });
    sock.on('close', () => { clearTimeout(t); resolve(JSON.parse(buf || '{}')); });
  });
}

async function snapshotRead(shards, keys, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const safe = await Promise.all(shards.map(s => rpc(s.host, s.port, { op: 'safeTime' })));
    const S = Math.min(...safe.map(r => r.safeTime ?? -1));
    if (S < 0) throw new Error('invalid safeTime from shard');

    // NOTE: toy broadcast. Real systems route keys to shards.
    const reads = await Promise.all(shards.map(s => rpc(s.host, s.port, { op: 'read', keys, ts: S })));
    if (reads.every(r => r.ok)) {
      return { ts: S, rows: Object.assign({}, ...reads.map(r => r.rows || {})) };
    }
  }
  throw new Error('snapshotRead failed after retries');
}

// Usage example (requires shard servers that implement {safeTime} and {read})
// snapshotRead([{host:'127.0.0.1',port:9001},{host:'127.0.0.1',port:9002}], ['K','T']).then(console.log);
```

What to watch for:

- If one shard is behind, the snapshot gets older.
- If safe times don’t advance (e.g., due to unresolved intents), reads stall.

---

##  Unresolved intents and read stalls

### Scenario
A write transaction creates intents on shard 1 and shard 2.

- Shard 1 receives commit decision.
- Shard 2 is partitioned and never resolves.

Now a read-only transaction wants a snapshot `S` that includes the commit.

### [PAUSE AND THINK]
What happens?

A) Reader ignores shard 2 and proceeds.
B) Reader blocks or retries because shard 2’s safe time can’t advance past the unresolved intent.
C) Reader sees partial commit.

---

###
**B** is the usual behavior in strongly consistent systems.

The system must not serve snapshots that might miss a commit <= S.

So unresolved intents can:

- block GC
- block safe time advancement
- increase read latency

Systems mitigate with:

- transaction heartbeats
- intent resolution queues
- pushing transactions (force abort/commit)
- bounded staleness reads (serve older snapshot)

###
In distributed MVCC, **transaction cleanup is part of the read path**.

---

##  “Read-only transactions are always fast”

Read-only transactions can be slow if:

- they require a fresh snapshot
- shards are lagging
- unresolved intents prevent safe time advancement
- read spans many shards

###
Read latency is often driven by **coordination for freshness**, not by local MVCC lookup.

---

##  MVCC and isolation levels in distributed systems

Different products expose different semantics:

- **Read Committed** with MVCC: each statement sees a fresh snapshot.
- **Repeatable Read / SI**: transaction sees one snapshot.
- **Serializable**: additional checks.

###  Isolation on MVCC

| Isolation | Snapshot behavior | Typical anomaly risk | How it’s implemented on MVCC |
|---|---|---|---|
| Read Committed | Snapshot per statement | phantoms, write skew, etc. | pick snapshot each statement |
| Snapshot Isolation | Snapshot per transaction | write skew | snapshot + write-write conflict |
| Serializable | As-if serial | none (within model) | SSI/predicate locks/validation |

###
If your system offers SI, what must your application still do?

Pause.

---

###
Model invariants carefully. If invariants involve multiple rows, you may need:

- explicit locking
- serializable isolation
- redesign invariants

###
Isolation level selection is an **application correctness decision**.

---

##  MVCC with consensus replication (log-based)

Some systems tie commit timestamps to the replication log.

- Commit order is log order.
- Snapshot `S` corresponds to a log index or timestamp.

Pros:

- strong ordering
- simpler external consistency

Cons:

- commit latency includes consensus
- cross-shard transactions still need coordination

[IMAGE: A Raft log with entries labeled with commit_ts; snapshot corresponds to applied index; readers can read at applied index.]

###
Consensus can provide “time” (ordering), but cross-shard atomicity remains hard.

---

##  MVCC and clock-based time (TrueTime/HLC)

If commit timestamps are derived from physical time (or hybrid time), you get benefits:

- monotonic ordering aligned with wall time
- easier “as of time” queries

But you must handle uncertainty:

- TrueTime uses bounded uncertainty and commit wait.
- HLC preserves causality-ish ordering without strict bounds, but you still need safe time frontiers.

###
Why might a system do a “commit wait” before making a transaction visible?

Pause.

---

###
To ensure external consistency: if a transaction commits at timestamp `t`, the system waits until real time is definitely past `t` before acknowledging, so no later transaction can appear to commit “in the past.”

###
Some MVCC systems pay latency to align timestamps with real-world time.

---

##  MVCC components to distributed responsibilities

Match the MVCC component to the distributed concern it stresses most:

| Component | Concern |
|---|---|
| Timestamp assignment | A) Storage bloat and compaction |
| Intent resolution | B) Availability under failures |
| Version GC | C) Read freshness and consistency |
| Safe time / resolved timestamp | D) Cleanup and liveness |

Pause and match.

---

###
- Timestamp assignment -> **B** (failover, fencing, monotonicity)
- Intent resolution -> **D**
- Version GC -> **A**
- Safe time / resolved timestamp -> **C**

###
Distributed MVCC is a web of *correctness + liveness + performance* constraints.

---

##  Which statement is true? (advanced)

1) If you have MVCC, you can serve linearizable reads from any replica.
2) If you have MVCC and a resolved timestamp, you can serve consistent historical reads from followers up to that timestamp.
3) If you have MVCC, write skew is impossible.
4) If you have MVCC, 2PC is unnecessary.

Pause.

---

###
- (1) False. Linearizable reads require additional coordination or lease-based leadership.
- (2) True (assuming resolved timestamp means completeness).
- (3) False.
- (4) False for multi-shard atomic transactions.

---

##  What will bite you in production

### 1) Hot keys and intent contention
Even if readers don’t block writers, **writers block writers**.

Hot keys lead to:

- lock queues / intent queues
- transaction retries
- tail latency

### 2) Long-running transactions
They pin old versions and blow up compaction.

### 3) Secondary index maintenance
Can dominate transaction cost.

### 4) Schema changes
Often require locks or careful backfills, temporarily breaking “no blocking.”

### 5) Observability
You need to see:

- oldest active snapshot
- intent counts and ages
- resolved timestamp per range/shard
- GC lag (bytes/version count)

[IMAGE: A dashboard mockup showing min_active_snapshot, resolved_ts per shard, intent age histogram, and MVCC GC bytes.]

###
MVCC correctness bugs are subtle; MVCC performance bugs are loud.

---

##  Diagnose a latency spike

### Scenario
At 14:05, p99 read latency jumps from 20ms to 800ms. Writes are steady.

Metrics show:

- resolved timestamp stops advancing on shard 7
- intent count increases on shard 7
- GC bytes pending increase cluster-wide

###
What’s the most plausible root cause?

A) A long-running read transaction started.
B) A transaction coordinator crashed leaving intents unresolved.
C) A follower fell behind.

---

###
**B** is most directly suggested: intents piling up and resolved timestamp stuck.

A long read would pin GC but doesn’t necessarily create intents.
A lagging follower doesn’t necessarily stop resolved timestamp on a shard (depends on definition), but the intent buildup is a strong signal of unresolved transactional state.

###
When safe time stops, look for **unresolved transactional metadata**.

---

##  Choosing MVCC semantics for your distributed system

### Pattern 1: Strong reads, fresh reads (expensive coordination)
- Linearizable reads
- Fresh snapshots
- Often read from leaders

### Pattern 2: Consistent but slightly stale reads (fast)
- Serve snapshot reads from followers up to resolved timestamp
- Great for analytics and user-facing “good enough” reads

### Pattern 3: Eventual consistency + MVCC-like history
- Keep versions for auditing
- Reads may not be transactionally consistent across shards

###
Which pattern fits a shopping cart service?

Pause.

---

###
Often:

- cart reads should be consistent for a user session (read-your-writes)
- global serializability may be unnecessary
- follower reads can work with session tokens

###
MVCC is a knob: you can trade freshness, latency, and availability.

---

##  Design MVCC for a global delivery platform

You’re building a global delivery platform with:

- orders (write-heavy)
- order tracking (read-heavy)
- inventory (needs invariants)
- multi-region deployment

### Requirements
- Users must see their own updates immediately.
- Tracking pages can be slightly stale (<= 5s).
- Inventory must not go negative.
- System must tolerate region partitions.

###  Design worksheet
Answer these progressively:

1) **Timestamps:** centralized oracle, HLC, or consensus-derived?
2) **Reads:** which queries can use follower snapshot reads?
3) **Isolation:** where do you need serializable vs SI?
4) **Failure handling:** how do you resolve intents when coordinators die?
5) **GC:** how do you prevent long-running analytics from pinning versions?

Write down your choices.

---

###  One plausible solution
1) **HLC + resolved timestamps** to avoid a single global bottleneck; ensure fencing via lease-based leadership per range.
2) **Follower reads** for tracking pages using resolved timestamp; **leader reads** for cart/session read-your-writes.
3) **Serializable** (or explicit locking) for inventory decrement; SI for most order updates.
4) Durable **transaction record** replicated; background intent resolution; push/abort on timeouts.
5) Separate analytics store or use bounded-staleness snapshots; enforce max txn duration; run analytics on follower snapshots.

###
A good MVCC design is not “turn it on.” It’s a set of explicit choices about **time, coordination, cleanup, and semantics**.

---

##  Take-home questions

1) Name two reasons MVCC read latency can spike even when CPU is fine.
2) Explain why resolved timestamp is about *completeness*, not *freshness*.
3) Give one application invariant that SI can violate.
4) Describe how you’d monitor MVCC health in production.

---
