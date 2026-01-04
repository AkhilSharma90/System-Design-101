---
slug: bloom-filters
title: Bloom Filters
readTime: 20 min
orderIndex: 9
premium: false
---



## Bloom Filters in Distributed Systems - An Interactive Deep Dive (Final Reviewed)

Audience: advanced distributed systems engineers and architects

Goal: build an intuition you can operate - choose parameters, predict failure modes, and design safe integrations.

> Reviewer notes (what changed vs draft):
> - Fixed a wire-format bug in the Node.js serialization example (header sizing/offsets were inconsistent).
> - Tightened correctness language: "no false negatives" is conditional on invariants (superset + hashing contract + monotonicity).
> - Added distributed-systems rigor: consistency models, CAP implications, network assumptions, and safe fallbacks.
> - Added production insights: saturation detection, rebuild strategies, CPU/cache locality, and operational runbooks.
> - Expanded failure scenarios (partial rollout, split-brain metadata, corruption/truncation, retries/idempotency for counting filters).
> - Ensured visual completeness: every major concept/variant/pattern has a diagram placeholder.

---

## Table of Contents (choose your path)
1. CHALLENGE: The expensive lookup problem
2. MENTAL MODEL: Maybe yes, definitely no
3. DEEP DIVE: How Bloom filters work (bits + hashes)
4. WORKSHOP: Sizing math you actually use (n, m, k, p)
5. DECISION GAME: which statement is true?
6. COMMON MISCONCEPTIONS (and why they hurt in prod)
7. DISTRIBUTED PATTERNS: where Bloom filters live
8. MENTAL MODEL: Caching, CDNs, and negative caching
9. DEEP DIVE: Distributed databases and LSM trees: SSTable filters
10. FAILURE SCENARIOS: drift, staleness, and adversarial inputs
11. WORKSHOP: Variants: counting, scalable, partitioned, blocked
12. MATCHING EXERCISE: choose the right filter for the job
13. OPERATIONS: Observability and measuring false positives
14. SYNTHESIS: design a Bloom-filter-backed system

---

## 1) CHALLENGE: The Coffee Shop With Too Many Questions

### Scenario
You run a busy coffee shop that also sells pastries. Customers constantly ask:

- Do you have almond croissants?
- Do you have gluten-free muffins?
- Do you have oat milk?

Your barista keeps running to the storeroom to check. The storeroom is far away, and each trip costs time.

In distributed systems terms:
- The storeroom is a remote service / database / shard.
- The barista is your API server.
- Running to check is a network hop + disk IO + CPU.

Now imagine 90% of questions are for items you do not have.

### Interactive question (pause and think)
If most lookups are negative, what is the best optimization?

- A) Speed up the storeroom checks
- B) Avoid going to the storeroom for negatives
- C) Add more baristas

Pause and think.

### Progressive reveal: Answer
B) Avoid going to the storeroom for negatives.

If you can cheaply determine definitely not in stock, you save the expensive trip. That is exactly what a Bloom filter gives you: a fast probabilistic membership test.

### Key insight box
> A Bloom filter is a negative-lookup accelerator: it is best when most queries are for keys that are absent and the backing lookup is expensive.

### Visual
[IMAGE: Coffee shop counter -> "Bloom filter sign" -> storeroom. Most queries stop at the sign with "NO".]

### Challenge question
If your workload is 90% positive lookups, will a Bloom filter help much? Why or why not?

---

## 2) MENTAL MODEL: Maybe Yes, Definitely No

### Scenario
You post a laminated sheet at the counter:

- If the sheet says NO, you trust it and do not check the storeroom.
- If the sheet says YES, it might be wrong; you still check.

That laminated sheet is your Bloom filter.

### Interactive question (pause and think)
Which of these can a standard Bloom filter do?

1) Return false positives (say YES when item is not there)
2) Return false negatives (say NO when item is there)
3) Support deletions

Pause and think.

### Progressive reveal: Answer
- (1) False positives: yes, by design.
- (2) False negatives: no - if implemented correctly and the filter represents a superset of inserted items under the same hashing contract.
- (3) Deletions: not in the standard Bloom filter (use counting Bloom filters or rebuild/rotate).

### Explanation with analogy
The laminated sheet is made by punching holes (setting bits). You cannot un-punch a hole without knowing whether that hole was also needed for some other item.

### Key insight box
> Bloom filters trade certainty for speed and compactness: definitely not is reliable; maybe yes is a hint.

### Visual
[IMAGE: Two outcomes: "NO -> stop" and "YES -> check source of truth".]

### Challenge question
If a Bloom filter cannot delete, what happens in a system where keys naturally expire (sessions, caches, TTL data)?

---

## 3) DEEP DIVE: How Bloom Filters Work (Bits + Hashes)

### Scenario
You have a row of m light switches (bits), all off. To insert an item, you flip k switches based on hash functions.

- Insert key x
- Compute k hashes: h1(x), h2(x), ..., hk(x)
- Map each hash to an index in [0, m-1]
- Set those bits to 1

To query membership:
- Compute the same k indices
- If any bit is 0 -> definitely not present
- If all bits are 1 -> maybe present (possible false positive)

### Visual
[IMAGE: Bit array length m. Insert "latte" sets k positions. Query "mocha" finds a 0 -> definitely not. Query "espresso" finds all 1 -> maybe.]

### Interactive question (pause and think)
Why do we use multiple hashes (k > 1) instead of just one?

Pause and think.

### Progressive reveal: Answer
Multiple hashes reduce false positives by spreading each item across multiple bits. With one hash, collisions concentrate and the filter saturates faster.

### Production note: hashing strategy
In practice you rarely compute k independent hashes. You use double hashing:

- idx_i = (h1 + i*h2) mod m

This is standard, fast, and has good empirical behavior.

### Key insight box
> Bloom filters work because absence is easy to prove: one missing bit disproves membership.

### Challenge question
What happens to the false positive rate as more keys are inserted into a fixed-size Bloom filter?

---

## 4) WORKSHOP: Sizing Math You Actually Use (n, m, k, p)

### Scenario
You are designing a distributed cache fronting a database. You want a Bloom filter to avoid DB hits for missing keys.

You need to choose:
- n: expected number of inserted elements
- m: number of bits
- k: number of hash functions
- p: desired false positive probability

### The core formulas
For a Bloom filter with m bits, k hashes, and n inserted items:

- Probability a particular bit is still 0 after inserts:
  (1 - 1/m)^(k*n) is approximately exp(-k*n/m)

- Probability a query yields a false positive:
  p is approximately (1 - exp(-k*n/m))^k

- Optimal number of hashes:
  k_opt = (m/n) * ln(2)

- Bits needed for target false positive rate:
  m = -(n * ln(p)) / (ln(2)^2)

### Interactive exercise (pause and think)
You expect n = 10^7 keys. You want p = 1%.

1) Roughly how many bits per element do you need?
2) What is k_opt roughly?

Pause and think.

### Progressive reveal: Answer
Bits per element b = m/n is approximately -ln(p) / (ln(2)^2)

- For p = 0.01, -ln(0.01) = 4.6052
- (ln(2)^2) is about 0.48045
- b is about 4.6052 / 0.48045 = 9.59 bits/element

So:
- m is about 9.6 * 10^7 bits, about 96,000,000 bits, about 12 MB

Optimal hashes:
- k_opt = b * ln(2) is about 9.59 * 0.693 = 6.64, about 7 hashes

### Key insight box
> Bloom filters are often shockingly small: about 10 bits/element buys you about 1% false positives.

### Performance consideration
- k affects CPU linearly.
- m affects memory footprint and cache locality.
- For very high QPS, blocked/partitioned Bloom filters can outperform "optimal k" designs due to fewer cache misses.

### Challenge question
If you halve m but keep n constant, what happens to the false positive rate? (Qualitative answer.)

---

## 5) DECISION GAME: Which Statement Is True?

### Scenario
Your team is debating Bloom filters for a distributed key-value store.

Choose the true statement(s):

A) Bloom filters reduce latency for negative lookups but can increase CPU due to hashing.

B) Bloom filters guarantee no extra load on the backing store.

C) Bloom filters can be safely merged across nodes by bitwise OR, assuming the same m and k and hash functions.

D) If the Bloom filter says definitely not, you should still check the database just in case.

Pause and think.

### Progressive reveal: Answer
- A is true. Hashing costs CPU; you are trading CPU for fewer network/disk round trips.
- B is false. False positives still hit the backing store; also operational issues (staleness, partial builds) can reduce effectiveness.
- C is true under strict conditions: same parameters and hash functions; OR corresponds to set union.
- D is false for a correctly built filter representing a superset of inserted keys. Definitely not is the point.

### Key insight box
> Bloom filters are probabilistic accelerators, not correctness mechanisms (except for the definitely-not guarantee under invariants).

### Visual
[IMAGE: Two filters with identical parameters -> OR merge -> union set.]

### Challenge question
Why might a distributed system accidentally violate the no-false-negatives property, even if Bloom filters theoretically do not have them?

---

## 6) COMMON MISCONCEPTIONS (That Break Systems)

### Misconception 1: Bloom filters are caches
Reality: A Bloom filter does not store values, only a probabilistic set membership summary.

- A cache answers: Here is the value.
- A Bloom filter answers: Worth checking the store?

Failure mode: treating a Bloom filter hit as authoritative and returning found without verifying.

Key insight box
> Always treat maybe present as go check the source of truth.

### Misconception 2: False positives are harmless
Reality: false positives translate into wasted backend work.

If your system is designed so that filter says maybe triggers:
- a cross-AZ read,
- a disk seek,
- or a quorum read,

then false positives can be expensive.

Key insight box
> Choose p by pricing the cost of a false positive vs the memory/CPU budget.

### Misconception 3: Bloom filters never have false negatives
Reality: the data structure does not - your distributed pipeline might.

You can introduce false negatives via:
- building the filter from incomplete data,
- race conditions (query before insert propagated),
- filter resets during rollouts,
- hash mismatch across versions,
- corrupted bitsets (bad serialization, truncation, bit rot).

Key insight box
> The no-false-negatives property is conditional on operational invariants.

### Misconception 4: Just pick k=10; more hashes is better
Reality: too many hashes can increase false positives for a fixed m and n (more bit setting -> faster saturation) and wastes CPU.

Key insight box
> Use k_opt = (m/n) * ln(2) as a starting point, then benchmark.

### Visual
[IMAGE: Graph of false positive rate vs k for fixed m,n showing a minimum near k_opt.]

### Challenge questions
1) Which is more dangerous in a distributed system: false positives or false negatives? Why?
2) When might you intentionally accept a higher false positive rate?

---

## 7) DISTRIBUTED PATTERNS: Where Bloom Filters Live

### Scenario
You run a globally distributed service with:
- edge caches,
- regional microservices,
- a sharded database.

Where do you place Bloom filters?

### Pattern 1: Client-side / edge Bloom filter
Use case: prevent requests for missing objects from reaching origin.

- The edge stores a Bloom filter of objects that exist in the origin.
- Query at edge:
  - If filter says no, respond 404 quickly.
  - If maybe, forward to origin.

Trade-off: staleness can cause system-level false negatives if you treat "no" as authoritative.

Key insight box
> Edge Bloom filters are powerful but require careful update semantics to preserve correctness.

### Pattern 2: Per-shard Bloom filters in a sharded DB
Use case: route reads to the right shard or avoid shard reads.

- Each shard maintains a filter of keys it stores.
- A coordinator tests filters to decide where to query.

Catch: stale filters can cause missed keys (operational false negatives) or misrouting.

### Pattern 3: On-disk structures (LSM/SSTable) Bloom filters
Use case: avoid disk reads for keys not in a file.

- Each SSTable stores a Bloom filter for its keys.
- On lookup, check filter before reading the file.

This is canonical and battle-tested.

### Pattern 4: Service-to-service negative lookup firewall
Use case: a high-QPS service calls another service for user IDs, product IDs, etc.

- Maintain a Bloom filter of valid IDs.
- Reject obviously invalid requests early.

Security note: be careful with adversarial inputs; see failure scenarios.

### Visual
[IMAGE: Four placement diagrams: Edge, Coordinator+Shards, LSM levels, Service A -> Bloom gate -> Service B.]

### Challenge question
Which pattern is most sensitive to staleness: edge filters, shard routing filters, or SSTable filters? Why?

---

## 8) MENTAL MODEL: Bloom Filters for Caching and Negative Caching

### Scenario
You operate a cache-aside system:

1) Read request arrives
2) Check cache
3) On miss, read DB
4) Populate cache

But your DB is getting hammered by requests for keys that do not exist.

### Idea
Maintain a Bloom filter of existing keys (or key prefixes) so you can skip DB hits when the key is definitely absent.

### Interactive question (pause and think)
If the Bloom filter says definitely not present, should you cache that negative response?

Pause and think.

### Progressive reveal: Answer
Often yes - but carefully.

- If the key might appear later (eventual consistency, delayed writes), negative caching can cause stale not found.
- If keys are immutable once absent (e.g., random UUIDs for objects), negative caching is safer.

### Safer approach than negative caching (when keys can appear later)
- Use a short TTL for negative cache entries.
- Or use write-through invalidation (on create, invalidate negative cache).
- Or use two-tier filters: base snapshot + delta for recent writes, and treat "no" as authoritative only if the request is older than the snapshot cut.

### Visual
[IMAGE: Read path: Bloom -> Cache -> DB, plus optional negative cache with TTL.]

### Challenge question
What is a safer approach than negative caching when keys can appear later?

---

## 9) DEEP DIVE: Bloom Filters in LSM Trees and SSTables (Why They Are Everywhere)

### Scenario
You are debugging read amplification in an LSM-tree-based store (RocksDB, LevelDB, Cassandra, HBase).

A point lookup might check many SSTables across levels. Without filters, each check can cause:
- index lookup,
- disk read,
- decompression,
- and lots of wasted work.

### How SSTable Bloom filters help
Each SSTable has a Bloom filter summarizing its keys.

Lookup flow:
1) For each candidate SSTable, check its Bloom filter.
2) If filter says no, skip reading that SSTable.
3) If maybe, consult index/data blocks.

### Visual
[IMAGE: LSM tree with levels L0/L1/L2; per-SSTable Bloom checks; most "NO" skip IO; one "MAYBE" reads blocks.]

### Interactive question (pause and think)
Why is a small false positive rate still acceptable here?

Pause and think.

### Progressive reveal: Answer
Because the alternative is far worse: without Bloom filters you may do many disk reads per lookup. A few extra reads due to false positives is a good trade.

### Compaction note
If compaction rewrites SSTables, their Bloom filters are rebuilt for the new files. This is one reason SSTable filters are operationally safe: the filter lifecycle is tied to immutable file creation.

### Key insight box
> Bloom filters are a primary tool to reduce read amplification in LSM trees.

---

## 10) FAILURE SCENARIOS in Distributed Environments

A Bloom filter is simple locally, but distributed systems add time, versions, and partial failure.

### Network assumptions (state them explicitly)
- Messages can be delayed, duplicated, reordered.
- Nodes can restart and lose in-memory state.
- Partitions can isolate producers from consumers.
- You may have mixed versions during rollouts.

### CAP / consistency framing
Bloom filters are typically used as an optimization layer. Under partitions you must choose:
- Correctness-first: treat filter as advisory; on "no" you may still check the source of truth (higher cost).
- Availability/latency-first: treat "no" as authoritative; you risk system-level false negatives if the filter is stale.

Bloom filters do not change CAP, but they can move where you pay the CAP trade-off.

---

### Failure 1: Stale filters causing operational false negatives

#### Scenario
You publish a Bloom filter to edge nodes every 10 minutes. An object is created at t=0, but the edge does not receive the updated filter until t=10m.

Edge behavior:
- Query for the new object at t=1m
- Filter says definitely not
- Edge returns 404

That is a false negative at the system level, even though the data structure did not lie - it was stale.

#### What invariant did you violate?
The filter must represent a superset of actually-present keys.

#### Mitigations
- Monotonic filters: only add keys (never remove). Correct for existence, but deletions are not reflected -> false positives grow.
- Versioned filters + fallback window: if filter says no, optionally check origin for a short window after writes.
- Write-through propagation: push updates on writes (hard at scale).
- Two-tier filters: stable base + small delta filter for recent writes.

#### Visual
[IMAGE: Timeline showing write at t0, edge update at t10m, request at t1m incorrectly blocked.]

Challenge question: If you must serve correct 404/200 at the edge, is a Bloom filter appropriate? Under what constraints?

---

### Failure 2: Parameter mismatch across services (hashing contract drift)

#### Scenario
Team A builds the filter with:
- m = 96M bits, k = 7, hash seeds S1

Team B queries with:
- m = 96M bits, k = 6, hash seeds S2

Now queries can produce apparent false negatives.

#### Why?
Membership is defined by the exact set of bit positions computed by the hash functions. If you compute different positions, you are checking the wrong bits.

#### Mitigations
- Include filter metadata: m, k, hash algorithm, seeds, version.
- Use a shared library.
- Add canary tests: insert known sentinel keys and verify they test positive.

#### Rollout strategy (production)
- Support dual-read during migration: query both old and new filters; treat "maybe" if either says maybe.
- Or ship a metadata service that pins clients to a compatible filter version.

#### Visual
[IMAGE: Two services with different hash versions; same key maps to different bit positions.]

---

### Failure 3: Filter saturation and performance collapse

#### Scenario
You sized for n=10M but actually insert n=50M.

Now many bits become 1, and p skyrockets. The filter becomes a "maybe for everything" machine.

#### Symptom
Backend load rises back toward baseline (or worse), while CPU remains higher due to hashing. Latency and cost increase.

#### Mitigations
- Use scalable Bloom filters (grow as needed).
- Monitor fill ratio / estimate n from bit density.
- Rebuild periodically.

#### What to alert on
- Estimated false positive rate (from bit density) crossing a threshold.
- Bit density (fraction of 1s) crossing a threshold.
- Avoided lookup rate dropping unexpectedly.

#### Visual
[IMAGE: Bit array gradually filling; curve of p rising sharply after saturation.]

---

### Failure 4: Adversarial inputs and hash flooding

#### Scenario
You expose an API endpoint that checks a Bloom filter before doing heavier work.

An attacker sends many keys designed to maximize collisions or force expensive hashing.

#### Mitigations
- Use fast, well-distributed hashes (xxHash/Murmur3) for benign environments.
- If adversarial, use keyed hashing (e.g., SipHash or BLAKE2 with a secret key) to prevent chosen-input attacks.
- Rate limit and apply request shaping.
- Do not use Bloom filter results as a security boundary.

#### Visual
[IMAGE: Attacker traffic -> Bloom gate -> backend; rate limiter and keyed hash shown.]

---

### Failure 5: Partial rollout / split-brain metadata

#### Scenario
You roll out a new filter version. Half the fleet uses v1, half uses v2. A load balancer routes requests arbitrarily.

If v2 is missing keys (bad build) and you treat "no" as authoritative, you get correctness regressions that appear as intermittent 404s.

#### Mitigations
- Make filter publication atomic: publish bits + metadata together, then flip a pointer.
- Make existence filters monotonic add-only when possible.
- Prefer fail-open behavior (treat errors as "maybe") when correctness matters.

#### Visual
[IMAGE: Fleet split: v1/v2; requests bouncing; intermittent failures.]

---

### Failure 6: Corruption / truncation in transit or at rest

#### Scenario
A filter blob is truncated in object storage, or a network transfer drops bytes. Consumers load a shorter bitset.

This can create false negatives (checking bits that are implicitly 0).

#### Mitigations
- Include length checks, checksums (CRC32C) or cryptographic hashes.
- Store in content-addressed form (hash in filename) and verify.
- Use defensive parsing and fail-open.

#### Visual
[IMAGE: Serialized blob with checksum; consumer verifies before use.]

---

## 11) WORKSHOP: Variants You Will Actually Need

Standard Bloom filters are insertion-only. Real systems often need more.

### Counting Bloom Filters (support deletions)
A counting Bloom filter replaces each bit with a small counter (e.g., 4 bits).

- Insert: increment k counters
- Delete: decrement k counters
- Query: if all counters > 0 -> maybe present

Trade-offs:
- More memory (counters vs bits)
- Risk of counter overflow if too small
- New failure modes: underflow/overflow, non-idempotent deletes

Distributed retry warning: if deletes are applied twice due to retries, counters can underflow and create false negatives.

Mitigations:
- Make updates idempotent (hard for counting filters without per-key state).
- Prefer epoch-based rebuild/rotation for TTL workloads.

### Visual
[IMAGE: Standard bit array vs counting array; insert increments; delete decrements; show underflow risk.]

---

### Scalable Bloom Filters (grow with n)
A scalable Bloom filter is a sequence of Bloom filters with increasing size and decreasing false positive targets.

- When the current filter saturates, add a new one.
- Query checks from newest to oldest.

Trade-offs:
- Slightly more CPU (multiple filters)
- Operationally friendlier when n is uncertain

### Visual
[IMAGE: Stack of filters F1, F2, F3; query checks newest->oldest.]

---

### Partitioned and Blocked Bloom Filters (cache-line friendly)
In high-performance systems, random memory access is expensive.

- Blocked Bloom filters arrange bits so that all k bits for an item likely fall into the same cache line.
- Great for high-QPS in-memory checks.

Trade-off:
- Slightly worse theoretical false positive rate for the same m in some designs, but often faster in practice.

### Visual
[IMAGE: Bit array divided into blocks; each key maps to one block; k bits inside block.]

---

## 12) MATCHING EXERCISE: Choose the Right Filter

Options:
1) Standard Bloom filter
2) Counting Bloom filter
3) Scalable Bloom filter
4) Blocked Bloom filter

Requirements:
A) We need deletions (TTL expiry) and can afford more memory.
B) We do not know how many elements we will store; must maintain target p.
C) We have fixed n and want the simplest, smallest structure.
D) We need extremely high throughput; CPU cache misses are the bottleneck.

### Answer
- A -> 2) Counting Bloom filter (with strong caveats about retries/idempotency)
- B -> 3) Scalable Bloom filter
- C -> 1) Standard Bloom filter
- D -> 4) Blocked Bloom filter

### Production clarification for TTL caches
For TTL-based eviction at scale, many teams prefer:
- rotating standard Bloom filters (time-bucketed) or
- stable Bloom filters (probabilistic aging)

instead of counting filters, because counting filters are fragile under retries and concurrency.

### Visual
[IMAGE: Decision tree mapping requirements to variants.]

---

## 13) OPERATIONS: Implementation Notes (with Code Markers)

You need:
- stable hashing
- deterministic serialization
- metadata for compatibility
- fast bit operations

### Python: Bloom filter (double hashing + sizing)
```python
import hashlib
import math
from dataclasses import dataclass

@dataclass(frozen=True)
class BloomMeta:
    m: int
    k: int
    seed: int
    algo: str = "blake2b-64-double"
    bit_order: str = "LSB0"  # bit i uses (byte=i>>3, mask=1<<(i&7))

class BloomFilter:
    def __init__(self, n: int, p: float, seed: int = 0):
        if n <= 0 or not (0 < p < 1):
            raise ValueError("n must be > 0 and p must be in (0,1)")

        m = int(math.ceil(-(n * math.log(p)) / (math.log(2) ** 2)))
        k = max(1, int(round((m / n) * math.log(2))))

        self.meta = BloomMeta(m=m, k=k, seed=seed)
        self.bits = bytearray((m + 7) // 8)

    def _indexes(self, key: bytes):
        # Two independent 64-bit hashes (domain separated) -> double hashing
        h1 = int.from_bytes(hashlib.blake2b(key, digest_size=8, person=b"bf1").digest(), "little")
        h2 = int.from_bytes(hashlib.blake2b(key, digest_size=8, person=b"bf2").digest(), "little") | 1
        m = self.meta.m
        for i in range(self.meta.k):
            yield (h1 + i * h2 + self.meta.seed) % m

    def add(self, key: str) -> None:
        if not isinstance(key, str) or not key:
            raise ValueError("key must be a non-empty string")
        b = key.encode("utf-8")
        for idx in self._indexes(b):
            self.bits[idx >> 3] |= (1 << (idx & 7))

    def might_contain(self, key: str) -> bool:
        # Fail-open is often safer for correctness (avoid operational false negatives)
        try:
            b = key.encode("utf-8")
            for idx in self._indexes(b):
                if (self.bits[idx >> 3] & (1 << (idx & 7))) == 0:
                    return False
            return True
        except Exception:
            return True

# Usage example
bf = BloomFilter(n=10_000_000, p=0.01, seed=42)
for k in ("latte", "espresso"):
    bf.add(k)
print(bf.might_contain("mocha"), bf.might_contain("latte"))
```

### Node.js: deterministic wire serialization (fixed)
The draft had inconsistent header sizing/offsets. Below is a correct, self-consistent format.

Wire format:
- Magic: 4 bytes: BFv1
- Header (14 bytes):
  - m u32 LE
  - k u16 LE
  - seed u32 LE
  - bitsLen u32 LE
- Payload: bitsLen bytes

```javascript
const net = require('net');
const crypto = require('crypto');

function packFilter({ m, k, seed, bits }) {
  if (!Number.isInteger(m) || m <= 0) throw new Error('bad m');
  if (!Number.isInteger(k) || k <= 0) throw new Error('bad k');
  if (!Number.isInteger(seed)) throw new Error('bad seed');
  if (!Buffer.isBuffer(bits)) throw new Error('bits must be a Buffer');

  const magic = Buffer.from('BFv1');
  const header = Buffer.alloc(14);
  header.writeUInt32LE(m, 0);
  header.writeUInt16LE(k, 4);
  header.writeUInt32LE(seed >>> 0, 6);
  header.writeUInt32LE(bits.length, 10);

  return Buffer.concat([magic, header, bits]);
}

function unpackFilter(buf) {
  if (buf.length < 4 + 14) throw new Error('truncated');
  if (buf.subarray(0, 4).toString('ascii') !== 'BFv1') throw new Error('bad magic');

  const m = buf.readUInt32LE(4);
  const k = buf.readUInt16LE(8);
  const seed = buf.readUInt32LE(10);
  const len = buf.readUInt32LE(14);

  const start = 18;
  const end = start + len;
  if (buf.length < end) throw new Error('truncated payload');

  const bits = buf.subarray(start, end);
  return { m, k, seed, bits };
}

// Example: send a serialized filter to a peer
const server = net.createServer((sock) => {
  sock.on('data', (d) => {
    try {
      console.log(unpackFilter(d));
    } catch (e) {
      sock.destroy(e);
    }
  });
});

server.listen(9000, '127.0.0.1', () => {
  const client = net.connect(9000, '127.0.0.1');
  const bits = crypto.randomBytes(128); // placeholder bitset
  client.end(packFilter({ m: 1024, k: 7, seed: 42, bits }));
});
```

### Node.js: cache-aside read path with Bloom gating + observability
```javascript
class Metrics {
  constructor(){
    this.avoided = 0; // filter said NO
    this.db = 0;      // DB lookups performed
    this.fp = 0;      // filter said MAYBE but DB said not found
    this.errors = 0;
  }
}

async function cacheAsideGet({ key, bloomMightContain, cacheGet, cacheSet, dbGet, metrics }) {
  // Correctness-first default: if bloom check fails, fail-open.
  let might;
  try {
    might = bloomMightContain(key);
  } catch {
    metrics.errors++;
    might = true;
  }

  if (!might) {
    metrics.avoided++;
    return null;
  }

  const cached = await cacheGet(key).catch(() => null);
  if (cached != null) return cached;

  metrics.db++;
  const val = await dbGet(key);
  if (val == null) {
    metrics.fp++;
    return null;
  }

  await cacheSet(key, val).catch(() => undefined);
  return val;
}
```

### Think about it
If you ship a filter over the network, what endian/bit-order assumptions can break compatibility?

### Progressive reveal: Answer
You must define:
- how bit index maps to byte + bit position (LSB-first vs MSB-first)
- integer endianness in metadata
- hash algorithm + seeds + domain separation
- versioning and compatibility rules

### Visual
[IMAGE: Bit index i -> (byte=i>>3, bit=i&7) mapping; show LSB0 vs MSB0 difference.]

---

## 14) OPERATIONS: Observability and Measuring False Positives

### Metrics to track
- Filter query rate (QPS)
- Filter negative rate: fraction returning definitely not
- Backend lookup rate: how often you still hit DB
- False positive rate estimate:
  - count cases where filter says maybe but DB says not found
- CPU cost: hashing time, cache misses
- Memory footprint
- Version distribution: percent of fleet on each filter version

### Measuring false positives without bias
Danger: measuring false positives only on maybe queries you send to DB can be biased if you have other gates.

Approaches:
- Sampling: for a small fraction of definitely-not results, still check DB to validate invariants (should be zero; if not, you have staleness/contract bugs).
- Shadow reads: asynchronously verify a sample of decisions.
- Bit-density-based estimate: estimate p from fraction of 1 bits (useful for saturation detection).

### Visual
[IMAGE: Metrics dashboard: avoided%, fp%, db qps, bit density, version skew.]

---

## Trade-offs: A Comparison Table

| Dimension | Bloom Filter Benefit | Bloom Filter Cost | Distributed Systems Twist |
|---|---|---|---|
| Latency | Avoids expensive negative lookups | Hashing overhead | Network hops dominate, so it often wins |
| Memory | Compact set representation | Needs RAM for bit array | Replication multiplies memory footprint |
| Correctness | Definitely not is reliable if up-to-date | Maybe requires verification | Staleness/rollouts can create operational false negatives |
| Scalability | Easy to shard/merge (OR) | Parameter coordination required | Versioning across fleets is hard |
| Security | Can shed invalid traffic | Can be abused if adversarial | Keyed hashes/rate limiting may be needed |

Key insight box
> Bloom filters are a systems trade: memory+CPU to save IO and cross-service work.

---

## SYNTHESIS CHALLENGE: Design a Distributed Inventory Existence Service

### Scenario
You run a multi-region commerce platform. Clients frequently ask if a product ID exists before rendering pages.

Constraints:
- 100k QPS globally
- Product catalog: 200M IDs
- Writes: 1k QPS (new products, deletions rare)
- Must avoid hammering the catalog DB with invalid IDs (bots + typos)
- Correctness requirement:
  - For existing products: must not incorrectly return does not exist
  - For non-existing: can return exists occasionally (will be corrected by later fetch)

You propose a Bloom filter.

### Step 1 - Choose placement
Options:
- A) Only in the catalog DB
- B) In each regional API cluster
- C) At the CDN/edge
- D) In clients (mobile/web)

Answer:
- B is typically best: close to request handling, controllable rollout, avoids edge staleness correctness traps.
- C can be powerful but risks incorrect 404 if stale.
- D creates update and abuse complexity.

### Step 2 - Update strategy
Options:
- A) Rebuild hourly from DB snapshot
- B) Stream inserts to the filter (monotonic adds)
- C) Use a base snapshot + delta filter for recent writes
- D) Do not update; rebuild weekly

Answer:
- C is robust: snapshot provides completeness; delta covers recent writes.
- If deletions are rare, monotonic adds are safe for correctness (but may increase false positives over time).

### Step 3 - Parameter sizing
You have n = 200M elements. Choose p.

Suppose p = 0.5%.

- -ln(0.005) = 5.298
- bits/element is about 5.298 / 0.48045 = 11.0

Memory:
- m is about 200M * 11 bits = 2.2B bits = 275MB

If deployed per region across 10 regions: about 2.7GB RAM total (before replication).

### Step 4 - Failure game
Which failure is most likely to page you at 2 AM?

A) Filter saturation due to underestimating n
B) Hash mismatch after a library upgrade
C) Stale edge filter causing incorrect 404
D) Counting filter counter underflow due to retry bugs

Answer (contextual):
- B is often the most dangerous because it silently introduces false negatives.
- A is common and shows up as load regressions.
- C depends on edge placement.
- D is why counting filters require careful idempotency.

### Visual
[IMAGE: System diagram: Clients -> Regional API -> Bloom check -> Cache -> Catalog DB; filter distribution + metadata/version service; base+delta filters.]

---

## FINAL: What You Should Be Able To Do Now

### You can now
- Explain Bloom filters as maybe yes, definitely no and know when that is valuable.
- Size a filter (m, k) for a target n and p.
- Identify when distributed realities (staleness, version mismatch) create operational false negatives.
- Choose variants (counting, scalable, blocked) based on system constraints.
- Instrument and monitor real-world false positive rates and saturation.

### Final synthesis challenge (whiteboard)
Design a Bloom-filter-assisted read path for a multi-tenant distributed cache in front of a sharded DB.

Requirements:
- Tenants have different key cardinalities.
- Some tenants require strong not found correctness.
- Others prioritize cost.
- You must support rolling upgrades.

Questions:
1) Do you use one filter per tenant or shared?
2) How do you prevent one tenant from saturating the filter?
3) Where do you store metadata and how do you version it?
4) How do you measure false positives per tenant?
5) What is your fallback when the filter is unavailable?

### Visual
[IMAGE: Multi-tenant diagram: per-tenant filters, versioned metadata, safe fail-open path.]

---

## Appendix: Quick Reference

### Parameter cheat sheet
- m = - (n * ln(p)) / (ln(2)^2)
- k = (m/n) * ln(2)
- Bits/element b = m/n = -ln(p) / (ln(2)^2)

### Operational invariants (print these)
- Filter must represent a superset of present keys to avoid operational false negatives.
- Hash function + seeds + bit indexing must be consistent across producers/consumers.
- Treat Bloom filters as an optimization layer: define fail-open vs fail-closed explicitly.
- Monitor saturation and rebuild/grow before collapse.
- Version and validate filter blobs (length + checksum) to prevent corruption-induced false negatives.
