---
slug: lsm-trees-compaction-strategies
title: Lsm Trees Compaction Strategies
readTime: 20 min
orderIndex: 10
premium: false
---



## LSM Tree Compaction Strategies (Size-Tiered vs Leveled) —

> Audience: engineers who already know what an LSM tree is, and now want to reason about compaction strategies under distributed workloads, failures, and operational constraints.

---

## [CHALLENGE] Your database is “fast”… until compaction day

You run a distributed key-value store backed by an LSM-tree engine. Writes are blazing fast. Then:

- read latency spikes at 2pm every day,
- background IO saturates disks,
- tail latencies explode during node rebuilds,
- and your on-call gets paged: “p99 read latency > 500ms”.

You look at metrics and see the same culprit every time: **compaction**.

### Pause and think
If compaction is “just background housekeeping,” why does it dominate user-visible performance?

Take 10 seconds.

...

### Reveal
Compaction is not just housekeeping; it’s **the mechanism that defines your read amplification, write amplification, space amplification, and failure recovery behavior**. In distributed systems, compaction also interacts with:

- replication and quorum reads
- anti-entropy and repair
- rebalancing and node replacement
- multi-tenant IO scheduling
- tail latency under load

Compaction isn’t a background detail. It’s a **core part of your storage protocol**.

**Key insight**
> Choosing a compaction strategy is choosing a performance envelope and a failure mode profile.

---

## [MENTAL MODEL] Compaction as a coffee shop’s “table management”

Imagine a busy coffee shop.

- Customers (writes) arrive constantly.
- The barista takes orders quickly and stacks tickets on the counter (memtable -> flushed SSTables).
- If you never clean or organize tables, customers can’t find seats (reads become expensive).

Compaction is the staff periodically:

- merging piles of tickets,
- throwing away canceled orders (tombstones),
- organizing tickets by time or category,
- and keeping the counter from overflowing.

Different compaction strategies are like different restaurant policies:

- **Size-Tiered Compaction (STC)**: “When we have enough small piles, merge them into a bigger pile.”
- **Leveled Compaction (LC)**: “We keep the counter organized into shelves (levels) with strict size limits, continuously moving tickets down.”

**Key insight**
> Compaction is the trade-off engine: it converts write-optimized ingestion into read-optimized layout—at a cost.

---

## [DEEP DIVE] What compaction actually does (beyond “merge files”)

Compaction typically:

1. **Merges sorted runs** (SSTables) into fewer, larger runs.
2. **Drops overwritten values** (keeping newest by sequence/ts).
3. **Applies tombstones** to delete keys.
4. **Rewrites indexes and bloom filters**.
5. **Reclaims space** and maintains invariants (non-overlap per level, size ratios, etc.).

### Decision game: Which is true?

**A.** Compaction is required mostly to reclaim disk space.

**B.** Compaction is required mostly to reduce the number of files and thus reduce read amplification.

**C.** Compaction is required mostly to maintain invariants that make point lookups and range scans efficient.

Pause and think.

...

**Answer reveal**: **B and C are the key reasons**, and A is only sometimes. Many systems can tolerate temporary space bloat but cannot tolerate unbounded read amplification.

**Key insight**
> Space reclamation is a side effect; the main job is controlling read amplification and maintaining layout invariants.

---

## [CHALLENGE] Two compaction strategies, one SLA

You must choose compaction strategy defaults for a distributed store with:

- replication factor 3
- mixed workload: 90% writes, 10% reads
- reads are mostly point lookups, but range scans exist (analytics)
- key distribution is skewed (hot keys)
- nodes can fail and be replaced frequently

You’re deciding between:

- **Size-Tiered Compaction (STC)**
- **Leveled Compaction (LC)**

### Pause and think
Which one would you pick if your top priority is **consistent read latency**? Which if your top priority is **max write throughput**?

...

### Reveal (high-level)
- **Consistent read latency** -> usually **Leveled**.
- **Max write throughput** -> often **Size-tiered**.

But distributed reality complicates this. We’ll build up the reasoning.

**Key insight**
> In distributed systems, compaction choice affects not just local performance but also replication lag, repair cost, and operational stability.

---

## [EXERCISE] Matching: Map “amplifications” to the strategy

Match each statement to STC or LC.

1. “Fewer bytes rewritten per ingested byte (usually).”
2. “Point reads typically touch fewer SSTables.”
3. “More space overhead due to duplicates across levels/runs.”
4. “Better for range scans, because data is more uniformly organized.”

Pause and think.

...

**Answer reveal**
1 -> **STC** (lower write amplification in many cases)
2 -> **LC** (lower read amplification)
3 -> **STC** (more duplicates across overlapping runs)
4 -> **LC** (more predictable range scans)

**Key insight**
> STC trades space (and read amplification) for write efficiency; LC trades write amplification for read predictability.

---

## [SECTION] Size-Tiered Compaction (STC)

### [CHALLENGE] The delivery service that bundles packages

You run a delivery hub. Small packages (fresh SSTables) arrive constantly. To reduce clutter, you wait until you have, say, 4 similarly sized bundles, then you **merge them into a larger bundle**.

That’s STC.

### How it works (core idea)
- SSTables are grouped by size “tiers”.
- When a tier has enough files (fan-in, e.g., 4 or 10), those files are compacted into a larger file in the next tier.
- Files in the same tier **overlap in key ranges**.

### Pause and think
If SSTables overlap, what happens to point reads?

...

**Answer reveal**
Point reads may need to check **multiple SSTables** (and their bloom filters) because the key could be in any overlapping run. Read amplification can grow with the number of runs.

**Key insight**
> Overlap is STC’s defining feature: it preserves write efficiency but makes reads less predictable.

---

### [MISCONCEPTION] “STC means compaction is rare”

Reality: STC compaction can be **bursty**.

- You wait for a tier to fill.
- Then you compact multiple files at once.
- That creates IO spikes.

In distributed systems, IO spikes can align across nodes (same workload patterns), causing correlated latency events.

**Key insight**
> STC’s burstiness can create synchronized compaction storms across a cluster.

---

### [DEEP DIVE] Distributed systems angle: STC under replication

Consider a replicated LSM store:

- each replica compacts independently
- compaction changes physical layout but not logical content

Independent compaction can lead to:

- **divergent SSTable boundaries** across replicas
- different bloom filter false-positive rates
- different read amplification per replica

#### Pause and think
If your coordinator does hedged reads (send to multiple replicas), which replica is likely to win?

...

**Answer reveal**
The replica with **lower read amplification** (fewer SSTables to consult, better cache locality) wins more often, becoming “hotter,” potentially causing load imbalance.

**Key insight**
> Compaction strategy can indirectly create replica load skew via tail-latency winners.

---

### [FAILURE] Tombstones + STC = long-lived ghosts

In STC, because overlapping runs persist, tombstones may take longer to be fully purged.

- a tombstone must “meet” older values during compaction to delete them
- if older values are spread across many overlapping runs, the tombstone may survive across multiple tiers

#### Decision game: Which statement is true?

1) “STC tends to purge tombstones quickly because it merges many files at once.”

2) “STC can keep tombstones around longer because overlap delays full key history convergence.”

Pause and think.

...

**Answer reveal**: **2**.

Distributed impact:
- long-lived tombstones increase read cost
- increase repair cost (tombstones replicate)
- complicate TTL-heavy workloads

**Key insight**
> STC can be painful for delete-heavy or TTL-heavy workloads unless tuned (or paired with time-window compaction).

---

### [IMAGE: Diagram of STC tiers with overlapping SSTables in each tier; arrows showing periodic merge of N similarly sized files into one larger file; annotate read path needing to check multiple overlapping runs]

---

## [SECTION] Leveled Compaction (LC)

### [CHALLENGE] A restaurant with labeled shelves (strict organization)

Instead of piles of tickets on the counter, you install shelves:

- Shelf 0: small, messy, many tickets (L0)
- Shelf 1: bigger, organized
- Shelf 2: bigger still

Rule: except for L0, each shelf contains files that **do not overlap** in key range.

When a shelf exceeds its size limit, you move some files down by merging them with overlapping files in the next shelf.

### How it works (core idea)
- L0 may have overlapping SSTables (flushes land here)
- L1..Ln are size-limited levels with **non-overlapping** files
- compaction picks a file (or range) from level i and merges it into level i+1, rewriting overlapping data

### Pause and think
If levels don’t overlap, what happens to point reads?

...

**Answer reveal**
A point read typically consults:

- some number of L0 files (overlapping)
- **at most one file per level** (because non-overlapping)

So read amplification is bounded and predictable.

**Key insight**
> LC buys predictable reads by enforcing non-overlap—at the cost of rewriting data many times.

---

### [MISCONCEPTION] “Leveled compaction is always better”

LC can be worse when:

- write-heavy workloads dominate
- disks are slow or IO budget is small
- you have high churn / frequent node replacement (rebuild + compaction)
- you rely on cheap ingestion and accept higher read cost

**Key insight**
> LC is not “better,” it’s more read-optimized and more write-expensive.

---

### [DEEP DIVE] Distributed systems angle: LC and predictable quorum reads

With LC, each replica tends to have:

- bounded number of SSTables to consult per key
- similar read amplification if configured similarly

This reduces variance across replicas and stabilizes quorum read latency.

#### Pause and think
If you do quorum reads (R=2 of 3) and you pick the fastest two responses, how does LC help tail latency?

...

**Answer reveal**
By reducing per-replica variance, LC reduces the chance that one replica is dramatically slower due to checking many overlapping runs. That tightens the distribution, improving p99.

**Key insight**
> LC’s biggest distributed advantage is variance reduction—not just lower average read cost.

---

### [IMAGE: Diagram of leveled compaction with L0 overlapping files and L1..Ln non-overlapping sorted ranges; show compaction of one L1 file into L2 overlapping range]

---

## [MENTAL MODEL] Compaction as “pay now vs pay later”

- **STC**: Pay later (reads pay by checking more runs; space pays by duplicates; occasional big merges).
- **LC**: Pay now (writes pay by rewriting data at multiple levels; steady background work; reads are cheaper).

Analogy:

- restaurant that cleans continuously (LC) vs cleans in big nightly batches (STC)
- delivery hub that bundles packages only when piles get big (STC) vs continuously sorts by route (LC)

**Key insight**
> Compaction strategy is a billing model: you decide whether to spend IO budget on writes or reads.

---

## [COMPARISON] STC vs LC (single-node + distributed consequences)

| Dimension | Size-Tiered (STC) | Leveled (LC) | Distributed consequence |
|---|---|---|---|
| Read amplification (point) | Higher, variable | Lower, bounded | Tail latency variance affects quorum/hedged reads |
| Write amplification | Lower (often) | Higher (rewrite across levels) | Replication lag if compaction steals IO from WAL/fsync |
| Space amplification | Higher (duplicates across runs) | Lower (less duplication) | More disk pressure -> more rebalancing events |
| Compaction pattern | Bursty | More continuous | Cluster-wide IO storms vs steady background |
| Tombstone GC | Can be slower | Typically faster/more predictable | Repair traffic and TTL workloads |
| Range scans | Often worse (many overlaps) | Better (organized ranges) | Analytics queries less disruptive |
| Operational tuning | Simpler knobs but tricky bursts | More knobs; predictable once tuned | SLO management and capacity planning |

**Key insight**
> In a cluster, variance and correlation matter as much as averages.

---

## [CHALLENGE] Why compaction is a distributed systems problem

Compaction is local work with global consequences. It interacts with:

- **replication**: IO budget competition, replica divergence
- **consistency**: tombstones, read-repair
- **rebalancing**: streaming SSTables, bootstrap, hinted handoff
- **multi-tenancy**: noisy neighbor compaction
- **failure recovery**: node rebuild triggers compaction debt

### Decision game: Which is the most dangerous combination?

A) STC + heavy deletes + frequent repairs

B) LC + write-heavy workload + small IO budget

C) Either strategy + no compaction throttling

Pause and think.

...

**Answer reveal**
All can be dangerous, but **C** is the cluster killer: unthrottled compaction can starve foreground IO and replication.

**Key insight**
> Compaction must be treated as a first-class resource consumer in your distributed scheduler.

---

## [DEEP DIVE] Amplification math (just enough to reason)

Goal: directional reasoning, not exact formulas.

### Write amplification (WA)
WA ~= total bytes written to storage / bytes ingested.

- LC rewrites data at each level multiple times.
- STC merges fewer times (bigger merges, fewer effective rewrite stages).

### Read amplification (RA)
RA ~= number of SSTables consulted per read (plus IO per consult).

- LC: bounded by (#levels + L0 files checked).
- STC: can grow with number of runs per tier.

### Space amplification (SA)
SA ~= physical bytes stored / logical live bytes.

- STC: higher due to duplicates across overlapping runs.
- LC: lower because duplicates converge faster.

#### Decision game
If your workload is 95% point reads and you have strict p99 latency SLO, which amplification dominates your choice?

...

**Answer reveal**
Read amplification dominates—so LC often wins, or you must heavily tune STC and caching.

**Key insight**
> For SLO-driven systems, the tail behavior of RA is often the deciding factor.

---

## [EXERCISE] Pick a strategy for each workload

Match the workload to the likely better default:

Workloads:

1) Write-heavy time-series ingestion, mostly recent reads, TTL deletes, large scans rare.
2) User profile store, mixed reads/writes, strict p99 reads, small values.
3) Log analytics store, heavy range scans, occasional point reads.
4) IoT metadata store on slow disks, limited IO budget, writes dominate.

Strategies:

- STC
- LC
- “It depends; consider time-window compaction”

Pause and think.

...

**Answer reveal (typical)**
1 -> It depends; consider TWCS (or STC tuned for TTL)
2 -> LC
3 -> LC (range scans benefit)
4 -> STC (lower WA; but must throttle)

**Key insight**
> Real systems often use hybrids (TWCS, tiered+leveled) to match time-based access patterns.

---

## [SECTION] Compaction selection and scheduling (where theory meets pager)

### [CHALLENGE] Two tenants, one disk

Tenant A: write-heavy ingestion.
Tenant B: latency-sensitive reads.

Compaction steals IO from reads and writes. If you don’t schedule it, the disk becomes a battleground.

### Pause and think
If you throttle compaction too much, what happens?

...

**Answer reveal**
You accumulate **compaction debt**:

- more SSTables -> higher read amplification
- more tombstones -> more CPU and IO
- eventually compaction must catch up, causing a bigger storm

**Key insight**
> Throttling compaction is like deferring maintenance on a delivery fleet: debt compounds.

---

### Common scheduling knobs (conceptual)

- max background compactions
- compaction IO rate limit
- L0 stop/slowdown triggers (LC especially)
- target file size and level size multiplier
- compaction priority (foreground vs background)
- subcompactions/parallelism

#### Decision game
Which knob is most directly tied to preventing write stalls in LC?

A) bloom filter bits-per-key

B) L0 slowdown/stop triggers

C) compression algorithm

...

**Answer reveal**: **B**.

**Key insight**
> LC turns compaction into a backpressure mechanism: L0 is the emergency brake.

---

## [FAILURE] Distributed failure scenarios: What compaction changes (and what it doesn’t)

Compaction preserves semantics, but failures happen mid-compaction.

### [CHALLENGE] Node crashes mid-compaction

Pause and think: what could go wrong?

...

**Answer reveal**
Engines treat compaction output as new immutable files and atomically install them via a manifest/version edit.

Failure modes:

- crash before install: old files remain; compaction output may be orphaned and later cleaned
- crash after install but before deleting old files: duplicates remain temporarily; safe but uses space
- manifest corruption / missing fsync: recovery may lose track of newest state (rare but catastrophic)

Distributed consequence:
- restarting node may have different compaction state than peers, affecting read latency and repair behavior

**Key insight**
> Compaction relies on atomic metadata updates; correctness is easy, but post-crash performance can drift.

---

### [CHALLENGE] Replica divergence + read repair

Compaction affects the *cost* of reconciliation:

- STC: more overlapping runs -> more work to locate key histories during repair scans
- LC: more organized layout -> more predictable iteration

Pause and think: does compaction reduce the amount of data that must be repaired across replicas?

...

**Answer reveal**
Not directly. Repair is about logical divergence, not physical layout. Compaction changes scan efficiency and tombstone retention.

**Key insight**
> Compaction doesn’t fix inconsistency; it changes the cost profile of detecting and healing it.

---

## [MISCONCEPTION] “Compaction is purely local, so it can’t affect cluster stability”

Reality: compaction is local work with global consequences.

- correlated disk saturation shrinks effective cluster capacity
- write stalls grow replication queues
- delayed tombstone purge increases repair traffic

**Key insight**
> Local IO contention becomes global tail latency through coordinated request fan-out.

---

## [SECTION] Real-world usage patterns

### Cassandra
STCS (size-tiered), LCS (leveled), TWCS (time-window).

### RocksDB
Often leveled compaction for predictable reads.

### HBase
Minor/major compactions; major compaction can be disruptive.

**Key insight**
> Strict latency SLOs often push systems toward leveled; time-series pushes toward time-window.

---

## [MENTAL MODEL] L0 is your “waiting room”

In LC:

- L0 accumulates flushed SSTables
- L0 overlap makes reads expensive
- compaction keeps L0 small

Analogy: clinic waiting room overflow slows the whole clinic.

**Key insight**
> Keeping L0 under control is the difference between stable latency and meltdown.

---

## [GAME] Which statement is true about range scans?

1) STC is always better for range scans because it rewrites less.

2) LC is generally better for range scans because non-overlapping levels reduce redundant reads.

3) Range scans are unaffected by compaction strategy because data is sorted in all SSTables.

...

**Answer reveal:** **2**.

**Key insight**
> Sorted runs aren’t enough; overlap determines redundant scanning.

---

## [SECTION] Compaction and caching: Why your cache hit rate lies

### [CHALLENGE] Cache looks great, p99 still bad

Hit rate is 95%, p99 still spikes.

Pause and think: how?

...

**Answer reveal**
Tail reads may touch many SSTables, causing extra metadata lookups and cache misses on indexes/filters.

**Key insight**
> Compaction shapes the working set of metadata, not just data blocks.

---

## [SECTION] Bootstrap and rebalancing

### [CHALLENGE] Adding a new node (streaming SSTables)

Compaction strategy affects data movement:

- LC: non-overlapping ranges help efficient streaming
- STC: overlap complicates file selection; you may stream extra bytes

Pause and think: which strategy is more likely to cause extra network transfer during range movement?

...

**Answer reveal:** **STC**.

**Key insight**
> Overlap increases data movement amplification during rebalancing.

---

## [SECTION] Anti-entropy (repair) scans

LC can reduce redundant reads during scans, but repair cost is still dominated by data size + network + hashing.

**Key insight**
> LC improves scan efficiency; it doesn’t make repair free.

---

## [EXERCISE] Simulate read amplification in your head

Toy model:

- L0 has 6 overlapping SSTables
- LC has 6 levels beyond L0 (L1..L6), non-overlapping

Question: for a point lookup, what’s the maximum SSTables you might consult?

...

**Answer reveal:** roughly 12 (6 from L0 + 1 per level).

Now STC: 4 tiers with 10 overlapping runs each -> worst case ~40.

**Key insight**
> Bounded worst-case is a major reason LC is used for latency-sensitive systems.

---

## [SECTION] Tuning knobs: What matters differently for STC vs LC

### STC tuning themes
- fan-in / min threshold
- max size per tier
- tombstone compaction
- smoothing and IO throttling to avoid bursts

### LC tuning themes
- level size multiplier
- target file size
- L0 triggers (slowdown/stop)
- compaction concurrency and subcompactions

Decision game: You see frequent write stalls in LC. Which change is most likely to help without changing hardware?

A) Increase L0 stop trigger

B) Increase compaction parallelism / background threads

C) Decrease bloom filter false positive rate

...

**Answer reveal:** often **B** (if headroom exists) or **A** (if you can tolerate more L0 overlap). C helps reads but not compaction throughput.

**Key insight**
> Write stalls mean compaction throughput < ingestion rate.

---

## [CODE: Python, simulate read amplification and write amplification with simplified models for STC vs LC]

```python
# Implementing simplified STC vs LC amplification simulation
from __future__ import annotations

from dataclasses import dataclass

@dataclass(frozen=True)
class Model:
    l0_files: int = 6
    levels: int = 6          # L1..L6
    stc_tiers: int = 4
    stc_runs_per_tier: int = 10

def estimate(model: Model, ingest_mb: float, size_ratio: int = 10, fan_in: int = 4) -> dict:
    # Read amplification: LC checks all L0 + at most 1 per level; STC checks all overlapping runs.
    ra_lc = model.l0_files + model.levels
    ra_stc = model.stc_tiers * model.stc_runs_per_tier

    # Write amplification: LC rewrites each byte ~once per level (very rough); STC fewer stages.
    wa_lc = max(1.0, float(model.levels))
    wa_stc = max(1.0, (model.stc_tiers / max(1, fan_in)) * (1.0 + 1.0 / max(1, size_ratio)))

    return {
        "ra_lc_sstables": ra_lc,
        "ra_stc_sstables": ra_stc,
        "lc_bytes_written_mb": ingest_mb * wa_lc,
        "stc_bytes_written_mb": ingest_mb * wa_stc,
    }

# Usage example
if __name__ == "__main__":
    try:
        print(estimate(Model(), ingest_mb=1024, size_ratio=10, fan_in=4))
    except Exception as e:
        raise SystemExit(f"simulation failed: {e}")
```

---

## [SECTION] Compaction with snapshots and long-running readers

### [CHALLENGE] Analytics query holds a snapshot for 30 minutes

Compaction cannot drop overwritten versions/tombstones visible to an active snapshot.

Effects:
- higher space amplification
- delayed tombstone purge

Distributed consequence: one long-running query on one replica can increase disk usage and trigger rebalancing/throttling.

**Key insight**
> Snapshots turn compaction from garbage collection into version management.

---

## [SECTION] Hot keys, skew, and compaction fairness

Hot partitions flush more, create more L0 files, and can trigger compaction hotspotting.

Pause and think: which strategy is more sensitive to hot partitions causing write stalls?

...

**Answer reveal:** often **LC** due to L0 stop/slowdown behavior.

**Key insight**
> LC enforces invariants continuously; hot spots violate them quickly.

---

## [SECTION] Operational playbook: Recognizing compaction pathologies

| Symptom | Likely cause | Strategy association |
|---|---|---|
| Periodic IO spikes, p99 spikes at same time daily | bursty compaction | STC more common |
| Sustained high disk write throughput | continuous compaction | LC |
| Write stalls / backpressure | L0 too large, compaction behind | LC |
| Disk usage keeps growing | deletes + insufficient compaction + snapshots | both; STC often worse |
| Quorum reads often wait for 1 slow replica | replica variance in RA | STC |

**Key insight**
> Graph L0 file count, pending compactions, compaction bytes/sec, and disk util together.

---

## [CHALLENGE] Compaction storms during node replacement

New node bootstraps data and then triggers compaction to reach steady-state (“compaction aftershock”).

Pause and think: which strategy tends to make bootstrap more painful?

...

**Answer reveal:** often **LC**, because it must build leveled invariants (rewriting data multiple times) unless you have a bulk-load path.

**Key insight**
> Bulk load + leveled invariants needs special handling to avoid rewriting the dataset multiple times.

---

## [CODE: Go, demonstrate a bulk-load ingestion path that places SSTables directly into lower levels (conceptual), vs naive L0 ingestion]

```go
// Implementing conceptual bulk-load placement vs naive L0 ingestion
package main

import (
	"errors"
	"fmt"
)

type SSTable struct{ Path string; Level int }

type Manifest struct{ tables []SSTable }

func (m *Manifest) Add(t SSTable) error {
	if t.Path == "" || t.Level < 0 { return errors.New("invalid sstable") }
	m.tables = append(m.tables, t) // In real engines: fsync + atomic version edit.
	return nil
}

func bulkLoad(m *Manifest, paths []string, targetLevel int) error {
	for _, p := range paths { if err := m.Add(SSTable{Path: p, Level: targetLevel}); err != nil { return err } }
	return nil
}

func naiveIngestToL0(m *Manifest, paths []string) error { return bulkLoad(m, paths, 0) }

func main() {
	m := &Manifest{}
	if err := bulkLoad(m, []string{"/data/s1.sst", "/data/s2.sst"}, 3); err != nil { panic(err) }
	fmt.Println("installed tables:", m.tables)
}
```

---

## [SECTION] Hybrid strategies and why they exist

- Time-window compaction (TWCS): compact within time buckets; older windows become immutable
- Universal compaction: tiered-like with different heuristics
- Tiered+leveled hybrids: tiering up top, leveling down low

**Key insight**
> Hybrids exist because workloads are not stationary: recent and cold data have different access patterns.

---

## [MENTAL MODEL] Temperature layers in storage

Recent data: optimize ingestion, accept overlap.
Old data: optimize scans/reads, enforce order.

**Key insight**
> The best strategy is often “different strategies for different temperatures.”

---

## [LAB] Design a compaction policy for a 3-region deployment

### [CHALLENGE]
3-region active-active, local quorum reads, cross-region latency high.

Question: would you choose the same strategy everywhere?

...

**Answer reveal**
Not necessarily: read-heavy regions may prefer LC; ingestion-heavy regions may prefer STC/TWCS. But this increases operational complexity (repair, disk usage differences, streaming efficiency).

**Key insight**
> Geo deployments can use compaction policy as specialization, but it complicates ops.

---

## [QUIZ] Pick the true statements (multi-select)

1) In LC, non-overlapping levels mean a point read consults at most one SSTable per level.
2) In STC, merging more files at once always reduces read amplification.
3) Compaction can impact hedged read behavior by changing per-replica variance.
4) Tombstone purge depends on compaction encountering older versions.

...

**Answer reveal:** true: 1, 3, 4. False: 2.

**Key insight**
> Read amplification is about how many overlapping runs remain, not just merge size.

---

## [GUIDE] Choosing between STC and LC

### If you lean STC
Use when:
- write throughput is paramount
- you can tolerate variable read latency
- you have disk headroom

Mitigate with: bloom filters, metadata caching, smoothing/throttling, TWCS for TTL.

### If you lean LC
Use when:
- strict read latency SLO
- range scans matter
- you can afford higher WA

Mitigate with: adequate compaction throughput, IO isolation, careful L0 triggers, bulk-load paths.

**Key insight**
> Choose based on what you can afford under failure and rebuild, not just steady state.

---

## [MISCONCEPTION] “We can just add more hardware to fix compaction”

Hardware helps, but storms can saturate any disk; correlated behavior still breaks SLOs.

Better: compaction budgets, IO isolation, and planning for rebuild/repair capacity.

**Key insight**
> Capacity planning must include compaction + repair + rebuild, not just foreground QPS.

---

## [SYNTHESIS CHALLENGE] You are the compaction scheduler

### Scenario (progressive reveal)
200-node cluster, RF=3, 80% writes, p99 read SLO 50ms, NVMe shared, ingestion peaks at noon, TTL=7 days, weekly node replacements.

Choose:
1) STC or LC as default
2) two key tuning knobs
3) one operational safeguard

Pause and think (write down your answer).

...

**Answer reveal (one reasonable solution)**

- Strategy: LC (or TWCS+LC if time-series dominates)
- Two knobs: (a) L0 triggers + compaction concurrency, (b) compaction IO rate limit / prioritization
- Safeguard: bulk-load/bootstrap path to avoid compaction aftershock + alerting on compaction debt

**Key insight**
> The best design is the one whose failure behavior you can operate.

---

## [CLOSING] Challenge questions

1) If p99 spikes correlate with compaction, is the root cause always “too much compaction”? What else might it be?
2) In your system, what dominates tail reads: L0 overlap, bloom filter false positives, or disk queueing?
3) During node bootstrap, how many times do you rewrite the dataset before steady-state? Can you measure it?
4) If you switch from STC to LC, what new failure mode becomes more likely?

---

### [IMAGE: Dashboard mock showing L0 file count, pending compaction bytes, compaction throughput, p99 read latency, and disk utilization; annotate how to spot compaction debt and storms]

### [IMAGE: Cluster diagram showing request fan-out to replicas, hedged reads, and how per-replica compaction variance affects tail latency]

### [CODE: PromQL snippets, alert on compaction debt, write stalls, and disk saturation correlated with compaction]

```javascript
// Implementing PromQL alert rule generation (Node.js) for compaction debt & stalls
const fs = require('fs/promises');

async function writeAlerts(outFile) {
  // Keep expressions generic: adapt metric names to your engine (RocksDB/Cassandra/etc.).
  const rules = `groups:\n- name: lsm_compaction\n  rules:\n  - alert: CompactionDebtHigh\n    expr: sum(compaction_pending_bytes) / sum(node_disk_bytes_total) > 0.10\n    for: 10m\n    labels: { severity: "page" }\n    annotations: { summary: "Compaction debt >10% of disk" }\n  - alert: L0FilesHigh\n    expr: max(l0_sstable_files) > 50\n    for: 5m\n    labels: { severity: "page" }\n    annotations: { summary: "L0 overlap high; write stalls likely" }\n  - alert: DiskSaturatedWithCompaction\n    expr: (avg(rate(node_disk_io_time_seconds_total[5m])) > 0.8) and (sum(rate(compaction_bytes_written_total[5m])) > 0)\n    for: 10m\n    labels: { severity: "page" }\n    annotations: { summary: "Disk saturated while compaction active" }\n`;

  await fs.writeFile(outFile, rules, { encoding: 'utf8' });
}

// Usage example
writeAlerts('./lsm_compaction_alerts.yaml').catch((e) => {
  console.error('failed to write alert rules:', e);
  process.exitCode = 1;
});
```
