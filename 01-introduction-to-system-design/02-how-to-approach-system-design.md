# How to Approach System Design Problems

Whether you're in an interview or designing a real system at work, having a structured approach prevents you from getting lost in details too early. This article gives you a repeatable 4-step framework.

---

## The 4-Step Framework

```
┌─────────────────────────────────────────────────────────┐
│              System Design Framework                     │
│                                                         │
│   Step 1: Understand Requirements          (5 min)      │
│   ─────────────────────────────────────────────────     │
│   Step 2: Estimate Scale                   (5 min)      │
│   ─────────────────────────────────────────────────     │
│   Step 3: Design the High-Level Architecture (15 min)   │
│   ─────────────────────────────────────────────────     │
│   Step 4: Deep Dive into Key Components    (15 min)     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Step 1: Understand the Requirements

Never jump straight into designing. First, ask clarifying questions to narrow the problem scope.

### Questions to Ask

**Users and Scale:**
- How many users does the system serve? (1K? 1M? 1B?)
- What's the expected read-to-write ratio?
- What geographies do we need to support?

**Functional Scope:**
- What are the core features we must support?
- What features are explicitly out of scope?
- Are there any specific user workflows we need to optimize for?

**Non-Functional Priorities:**
- What's more important: consistency or availability?
- What are the latency requirements?
- Do we need real-time updates or is eventual consistency acceptable?

### Example: Designing a Paste Service (like Pastebin)

> **Clarifying questions you might ask:**
> - Can pastes contain only text, or also images/files?
> - Is there a size limit per paste?
> - Do pastes expire?
> - Do we need user accounts, or can anyone create a paste?
> - What's the expected traffic — reads vs writes?

After clarification, you might arrive at:

**Functional Requirements:**
- Create a paste with text content and receive a unique URL
- Read a paste by visiting its URL
- Pastes expire after a configurable period (default: 30 days)
- Optional: user accounts for managing pastes

**Non-Functional Requirements:**
- High availability — reads should almost never fail
- Low latency — paste retrieval under 200ms
- Scale: 5M new pastes/day, 25M reads/day (5:1 read-to-write ratio)

---

## Step 2: Estimate Scale

Back-of-the-envelope estimation helps you make informed architecture decisions. You don't need exact numbers — order of magnitude is enough.

### Key Calculations

**Traffic estimates:**
```
Writes: 5M pastes/day
      = 5,000,000 / 86,400 ≈ 58 writes/second

Reads:  25M reads/day
      = 25,000,000 / 86,400 ≈ 290 reads/second
```

**Storage estimates:**
```
Average paste size: 10 KB
Daily storage:  5M × 10 KB = 50 GB/day
Annual storage: 50 GB × 365 = ~18 TB/year
5-year storage: ~90 TB
```

**Bandwidth estimates:**
```
Incoming: 58 writes/sec × 10 KB = 580 KB/sec ≈ 0.6 MB/sec
Outgoing: 290 reads/sec × 10 KB = 2,900 KB/sec ≈ 2.9 MB/sec
```

### Useful Numbers to Memorize

| Operation | Time |
|-----------|------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| RAM reference | 100 ns |
| SSD random read | 150 μs |
| HDD random read | 10 ms |
| Send 1 KB over network (same datacenter) | 250 μs |
| Round trip within same datacenter | 0.5 ms |
| Round trip CA → Netherlands | 150 ms |

| Unit | Exact | Approximation |
|------|-------|---------------|
| 1 Million | 10^6 | 1M |
| 1 Billion | 10^9 | 1B |
| 1 day in seconds | 86,400 | ~100K |
| 1 year in seconds | 31,536,000 | ~30M |

---

## Step 3: Design the High-Level Architecture

Start with the simplest design that satisfies requirements, then evolve it.

### Start Simple

For our paste service, the simplest design is:

```
┌──────────┐       ┌──────────────┐       ┌──────────────┐
│          │       │              │       │              │
│  Client  │──────▶│  App Server  │──────▶│   Database   │
│          │       │              │       │              │
└──────────┘       └──────────────┘       └──────────────┘
```

### Identify Bottlenecks and Evolve

Ask yourself: *What breaks first as we scale?*

1. **Single server can't handle 290 reads/sec?** → Add multiple app servers behind a load balancer
2. **Database becomes a bottleneck?** → Separate reads from writes with replicas
3. **Reads are too slow?** → Add a cache layer for hot pastes
4. **90 TB of storage?** → Use object storage (S3) for paste content, keep only metadata in the database

**Evolved Design:**

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐
│          │     │               │     │   App Server 1   │──┐
│          │     │     Load      │────▶│                  │  │
│ Clients  │────▶│   Balancer   │     └──────────────────┘  │
│          │     │               │     ┌──────────────────┐  │
│          │     │               │────▶│   App Server 2   │──┤
└──────────┘     └───────────────┘     └──────────────────┘  │
                                       ┌──────────────────┐  │
                                       │   App Server N   │──┤
                                       └──────────────────┘  │
                                                             │
                    ┌──────────────────┐                     │
                    │   Cache (Redis)  │◀────────────────────┤
                    └────────┬─────────┘                     │
                             │ miss                          │
                    ┌────────▼─────────┐    ┌─────────────┐  │
                    │   Metadata DB    │    │Object Store  │  │
                    │   (PostgreSQL)   │    │    (S3)      │◀─┘
                    └──────────────────┘    └─────────────┘
```

---

## Step 4: Deep Dive into Key Components

Pick 2–3 components that are most critical and design them in detail.

### What to Deep Dive On

Choose based on what's **unique or challenging** about your system:

| System | Likely Deep Dives |
|--------|------------------|
| URL Shortener | Key generation, redirection flow |
| Chat System | Real-time messaging, presence detection |
| News Feed | Feed generation, ranking algorithm |
| Payment System | Idempotency, transaction handling |

### For Our Paste Service

**Deep Dive 1: Key Generation**

How do we generate unique, short IDs for each paste?

- **Option A: Hash the content** — MD5/SHA256 → Base62 encode → take first 8 chars. Problem: collisions.
- **Option B: Auto-increment ID** — Simple but predictable. Security concern.
- **Option C: Pre-generate keys** — Generate random keys in advance, store in a key database. A Key Generation Service (KGS) picks unused keys. No collisions, no predictability.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  App Server  │────▶│     KGS      │────▶│   Key DB     │
│              │     │  (assigns    │     │  (unused     │
│              │◀────│  unused key) │     │   keys)      │
│              │     └──────────────┘     └──────────────┘
│  "abc123"    │
└──────────────┘
```

**Deep Dive 2: Cleanup / Expiration**

How do we handle paste expiration?

- **Lazy deletion:** Check expiration on read. If expired, delete and return 404. Simple but doesn't free storage proactively.
- **Active cleanup:** Background job scans for expired pastes periodically. Frees storage but adds operational complexity.
- **Best approach:** Both. Lazy deletion for correctness, active cleanup for storage reclamation.

---

## Common Mistakes to Avoid

1. **Jumping to the solution** — Always start with requirements and estimation
2. **Over-engineering** — Don't add microservices, Kafka, and Kubernetes to a system with 100 users
3. **Ignoring trade-offs** — Every decision has pros and cons. State them explicitly
4. **Designing in silence** — In an interview, talk through your reasoning. At work, write it down
5. **Premature optimization** — Design for current scale with a clear path to grow, not for 10x scale on day one

---

## A Checklist Before You're Done

- [ ] Are all functional requirements addressed?
- [ ] Are non-functional requirements (latency, availability, scale) satisfied?
- [ ] Have you identified single points of failure?
- [ ] Can the system scale horizontally?
- [ ] Have you discussed trade-offs for major decisions?
- [ ] Is the data model clear?
- [ ] Are failure scenarios handled (what happens when X goes down)?

---

## What's Next

With this framework in hand, let's start learning the core concepts. Next up: [Scalability](03-scalability.md) — the foundation of thinking about systems that grow.

---

*Previous: [What is System Design? ←](01-what-is-system-design.md) · Next: [Scalability →](03-scalability.md)*
