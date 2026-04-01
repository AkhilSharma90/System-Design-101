# System Design 101

**149 articles** covering everything from DNS resolution to designing payment systems at scale. Free and open source.

> **Learn interactively** — structured courses, progress tracking, and hands-on exercises at **[algoroq.io](https://algoroq.io)**

---

## Who This Is For

- **Interview prep** — FAANG/top-tech system design rounds
- **Backend engineers** — going deeper on distributed systems
- **New engineers** — building a solid mental model from scratch

---

## Learning Paths

### Path 1 — Interview Prep (4–6 weeks)

```
Week 1-2: Foundations
  → Scalability, CAP Theorem, Load Balancing, Caching, Databases

Week 3-4: Distributed Patterns
  → Sharding, Consistent Hashing, Replication, Message Queues, Rate Limiting

Week 5-6: System Design Problems
  → URL Shortener → Chat System → News Feed → Video Streaming
  → Distributed Cache → Payment System → Notification System → Web Crawler
```

### Path 2 — Deep Dive for Engineers (ongoing)

```
Distributed Algorithms → Data Management → Event-Driven Architecture
→ Architectural Patterns → Reliability & Resilience → Observability
```

---

## Quick Reference — Numbers Every Engineer Should Know

### Latency Numbers

| Operation | Latency |
|-----------|---------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| RAM access | 100 ns |
| Read 1MB from RAM | 250 µs |
| SSD random read | 150 µs |
| Read 1MB from SSD | 1 ms |
| Round trip within same datacenter | 0.5 ms |
| Read 1MB from HDD | 20 ms |
| HDD seek | 10 ms |
| Round trip US → Europe | 150 ms |

**Rule of thumb:** RAM is ~1,000× faster than SSD. SSD is ~100× faster than HDD. Network inside a datacenter is fast (~0.5ms); cross-continent is slow (~150ms).

---

### Powers of 2 Reference

| Power | Value | Human readable |
|-------|-------|----------------|
| 2^10 | 1,024 | ~1 thousand (1 KB) |
| 2^20 | 1,048,576 | ~1 million (1 MB) |
| 2^30 | 1,073,741,824 | ~1 billion (1 GB) |
| 2^40 | ~1 trillion | ~1 TB |
| 2^50 | ~1 quadrillion | ~1 PB |

---

### Availability Nines

| Availability | Downtime / year | Downtime / month | Downtime / week |
|-------------|-----------------|------------------|-----------------|
| 99% | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.99% | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% | 5.26 minutes | 26.3 seconds | 6.05 seconds |
| 99.9999% | 31.5 seconds | 2.6 seconds | 0.6 seconds |

---

### Back-of-Envelope Conversions

```
1 million req/day   =  ~12 req/sec
1 billion req/day   =  ~11,600 req/sec  (~12K rps)
1 million req/sec   =  86.4 billion req/day

Storage:
  1 byte  × 1 billion users = 1 GB
  1 KB    × 1 billion users = 1 TB
  1 MB    × 1 billion users = 1 PB

Read/Write:
  Average web page   ≈  100 KB
  Average photo      ≈  300 KB
  Average tweet      ≈  300 bytes
  Average video/min  ≈  50 MB (720p)
```

---

### Common System Limits (Rule of Thumb)

| Component | Typical throughput |
|-----------|-------------------|
| Single SQL DB (reads) | ~10,000 QPS with index |
| Single SQL DB (writes) | ~1,000–5,000 QPS |
| Redis | 100,000–1,000,000 ops/sec |
| Kafka | 1,000,000+ msgs/sec (partitioned) |
| Single HTTP server | ~10,000–50,000 req/sec |
| CDN (global edge) | Effectively unlimited |

---

## What's Inside

### Introduction to System Design

Fundamentals every engineer needs before diving into distributed systems.

| Section | Articles |
|---------|----------|
| [System Design Foundationals](introduction-to-system-design/system-design-foundationals-topics/) | HLD vs LLD · Back-of-envelope estimation · Vertical vs horizontal scaling · Load balancing · Single point of failure · Latency and throughput · **CAP theorem** |
| [Caching Fundamentals](introduction-to-system-design/caching-fundamentals/) | What is caching · Cache hits vs misses · Eviction policies · CDN · Browser vs server cache |
| [Database Fundamentals](introduction-to-system-design/database-fundamentals/) | SQL vs NoSQL · CRUD · ACID · Primary/foreign keys · Transactions · Indexing · Read/write patterns · Backups |
| [Computer Networking](introduction-to-system-design/computer-networking-protocols-basics/) | Networking protocols · IP · TCP · UDP · TCP vs UDP · TLS · DNS |
| [REST API Fundamentals](introduction-to-system-design/fundamentals-of-rest-api/) | REST APIs · HTTP status codes · Authentication · Path & query params · Pagination · Versioning · API gateway · Rate limiting |
| [Storage Fundamentals](introduction-to-system-design/storage-fundamentals/) | Where your data lives · Disk vs memory · Hot/warm/cold data |
| [Basics of Computing](introduction-to-system-design/basics-of-computing/) | How computers work · Binary system · Memory · CPU basics |

> **Take the full interactive Introduction course → [algoroq.io/courses/introduction-to-system-design](https://algoroq.io)**

---

### Intermediate System Design

Patterns and techniques for building scalable, reliable systems.

| Section | Articles |
|---------|----------|
| [Database Patterns](system-design-intermediate/database/) | Sharding · Replication · Master-slave architecture · Distributed databases · **Consistent hashing** |
| [Caching Patterns](system-design-intermediate/caching-patterns/) | Cache-aside (lazy loading) · Write-through · Read-through · Write-behind · Comparison |
| [Event-Driven Architecture](system-design-intermediate/event-driven-architecture/) | Event streams · Stream patterns · Technologies · Stream processing · Pub-sub · Kafka · RabbitMQ · MQTT · Dead letter queues |
| [Software Architecture Patterns](system-design-intermediate/software-architecture-patterns/) | Client-server · Stateful vs stateless · Async vs sync · Forward/reverse proxy · Webhooks · Monolith vs microservices |
| [API Protocols](system-design-intermediate/api-communication-protocols-and-architectural-styles/) | GraphQL · Protocol Buffers · gRPC · WebRTC · SMTP · Website metrics |
| [Search & Storage Systems](system-design-intermediate/search-and-storage/) | **Search systems & inverted indexes** · **Blob/object storage (S3)** · **CDN & edge computing deep dive** |
| [Reliability Patterns](system-design-intermediate/reliability-patterns/) | **Rate limiting: token bucket, leaky bucket, sliding window, Redis implementation** |
| [Software Testing](system-design-intermediate/fundamentals-of-software-testing/) | Smoke · Functional · Integration · API · System · E2E · Stress · Chaos testing |
| [Monitoring & Observability](system-design-intermediate/software-monitoring%20-and-observibility/) | Logging · Observability · Health checks & heartbeats |

> **Take the full interactive Intermediate course → [algoroq.io/courses/system-design-intermediate](https://algoroq.io)**

---

### Advanced System Design

Deep distributed systems theory and real-world system design problems.

| Section | Articles |
|---------|----------|
| [Distributed Algorithms](system-design-advanced/distributed-algorithms/) | Vector clocks & Lamport timestamps · Byzantine fault tolerance · X3DH · Double ratchets · **Distributed consensus (Raft/Paxos)** · 2PC & 3PC · Quorum & sloppy quorum · Gossip protocol · Merkle trees · Anti-entropy · Hinted handoff · Split-brain · Linearizability · Causal consistency |
| [Event-Driven Architecture](system-design-advanced/event-driven-architecture/) | Event sourcing at scale · Complex event processing · Transactional outbox · Transactional inbox · Event mesh · Kappa vs Lambda architecture |
| [Architectural Patterns](system-design-advanced/architectural-patterns/) | Saga patterns · Strangler fig · Backend for frontend · API gateway vs service mesh · Multi-tenancy · Hexagonal architecture |
| [Data Management](system-design-advanced/data-management/) | CRDTs · MVCC · Optimistic vs pessimistic locking · Database federation · Polyglot persistence · Zero-downtime migration · Hot partition detection · Time-series DB optimization · Bloom filters · LSM trees |
| [Caching & Performance](system-design-advanced/caching-and-performance/) | Thundering herd problem · Adaptive caching with ML · Distributed locks with Redis · Cache coherence in multi-region · Edge computing & edge caching |
| [Observability](system-design-advanced/observability/) | Tail-based sampling · OpenTelemetry & distributed tracing · High-cardinality data management · Observability-driven development |
| [Reliability & Resilience](system-design-advanced/reliability-and-resilience/) | Game days & DIRT testing · Blast radius & failure domain isolation · SLIs/SLOs/error budgets · Multi-region active-active · Global load balancing · Cascading failure prevention |
| [**System Design Problems**](system-design-advanced/system-design-problems/) | See below |

> **Take the full interactive Advanced course → [algoroq.io/courses/system-design-advanced](https://algoroq.io)**

---

### System Design Problems

End-to-end system design walkthroughs in interview format: requirements → estimation → architecture → deep dives → key takeaways.

| # | Problem | Key Concepts |
|---|---------|-------------|
| 1 | [Design a URL Shortener](system-design-advanced/system-design-problems/01-design-url-shortener.md) | Key generation service, 301 vs 302 redirect, Redis caching, expiration |
| 2 | [Design a Chat System](system-design-advanced/system-design-problems/02-design-chat-system.md) | WebSockets, Kafka fan-out, Cassandra message storage, presence with heartbeats |
| 3 | [Design a News Feed](system-design-advanced/system-design-problems/03-design-news-feed.md) | Fan-out on write vs read vs hybrid, ML ranking signals, cursor pagination |
| 4 | [Design a Video Streaming Platform](system-design-advanced/system-design-problems/04-design-video-streaming.md) | Chunked multipart upload, DAG transcoding, HLS adaptive bitrate, storage tiering |
| 5 | [Design a Distributed Cache](system-design-advanced/system-design-problems/05-design-distributed-cache.md) | Hash slots, LRU eviction O(1), hot key mitigation, online resharding |
| 6 | [Design a Payment System](system-design-advanced/system-design-problems/06-design-payment-system.md) | Idempotency keys, double-entry bookkeeping, PSP integration, reconciliation |
| 7 | [Design a Notification System](system-design-advanced/system-design-problems/07-design-notification-system.md) | APNs/FCM/SendGrid/Twilio, Redis deduplication, priority queues, retry strategies |
| 8 | [Design a Web Crawler](system-design-advanced/system-design-problems/08-design-web-crawler.md) | URL frontier, Bloom filter deduplication, SimHash, robots.txt, spider traps |

---

## CAP Theorem at a Glance

One of the most important concepts in distributed systems. Every database makes this trade-off.

```
A distributed system can guarantee only 2 of these 3 properties:

         Consistency
              ▲
             / \
            /   \
           / CP  \   ZooKeeper, etcd, HBase
          /       \
         /─────────\
Availability       Partition Tolerance
     AP                  (mandatory)

  Cassandra, DynamoDB, DNS
```

**The real choice:** Partition tolerance is not optional in real networks. The choice is **C vs A when a partition occurs**.

- **CP** — returns an error rather than stale data (banks, inventory, auth)
- **AP** — returns possibly stale data rather than an error (shopping carts, social feeds, DNS)

→ [Full article: CAP Theorem](introduction-to-system-design/system-design-foundationals-topics/07-cap-theorem.md)

---

## Consistent Hashing at a Glance

How distributed systems route data to nodes without mass re-shuffling when servers change.

```
Naive hashing:  hash(key) % N
  Add 1 server to N=3 → N=4:  ~75% of keys move  ← catastrophic

Consistent hashing:
  Map both keys and servers onto a ring.
  Add 1 server: only ~1/N keys move.

              0
           ╭────╮
        ╭──╯    ╰──╮
  A(45°)│            │B(135°)
        │            │
         ╰──╮    ╭──╯
           ╰────╯
           C(260°)

Key → assigned to next server clockwise
```

Used by: Redis Cluster (16,384 hash slots), Cassandra (256 vnodes), DynamoDB, CDNs.

→ [Full article: Consistent Hashing](system-design-intermediate/database/05-consistent-hashing.md)

---

## Repository Structure

```
System-Design-101/
├── introduction-to-system-design/
│   ├── system-design-foundationals-topics/   (7 articles)
│   ├── caching-fundamentals/                 (5 articles)
│   ├── database-fundamentals/                (8 articles)
│   ├── computer-networking-protocols-basics/ (7 articles)
│   ├── fundamentals-of-rest-api/             (8 articles)
│   ├── storage-fundamentals/                 (3 articles)
│   └── basics-of-computing/                  (4 articles)
│
├── system-design-intermediate/
│   ├── database/                             (5 articles)
│   ├── caching-patterns/                     (5 articles)
│   ├── event-driven-architecture/            (9 articles)
│   ├── software-architecture-patterns/       (6 articles)
│   ├── api-communication-protocols.../       (6 articles)
│   ├── search-and-storage/                   (3 articles)
│   ├── reliability-patterns/                 (1 article)
│   ├── fundamentals-of-software-testing/     (8 articles)
│   └── software-monitoring-and-observibility/(3 articles)
│
└── system-design-advanced/
    ├── distributed-algorithms/               (14 articles)
    ├── event-driven-architecture/            (6 articles)
    ├── architectural-patterns/               (6 articles)
    ├── data-management/                      (10 articles)
    ├── caching-and-performance/              (5 articles)
    ├── observability/                        (4 articles)
    ├── reliability-and-resilience/           (6 articles)
    └── system-design-problems/               (8 articles)
```

**Total: 149 articles**

---

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on article format, structure, and the review process.

---

## License

This content is open source. See the repository license for details.

---

<p align="center">
  <strong>Found this useful? Give it a star — it helps others find it too.</strong>
  <br><br>
  Continue learning interactively at <a href="https://algoroq.io"><strong>algoroq.io</strong></a>
  <br>
  Structured courses · Progress tracking · Hands-on exercises
</p>
