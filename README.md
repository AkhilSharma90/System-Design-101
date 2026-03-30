# System Design 101

A comprehensive system design curriculum for engineers — from core fundamentals to designing production-grade distributed systems at scale. Built for the [Algoroq](https://algoroq.com) learning platform.

---

## What You'll Learn

This repository is structured as a complete learning path across three levels:

| Level | Focus |
|-------|-------|
| **Introduction** | Core principles, building blocks, how the web works |
| **Intermediate** | Distributed patterns, scalability techniques, real-world architectures |
| **Advanced** | Complex distributed systems, design problems, production engineering |

---

## Introduction to System Design

Fundamentals every engineer needs before diving into distributed systems.

| Section | Topics |
|---------|--------|
| [System Design Foundationals](introduction-to-system-design/system-design-foundationals-topics/) | HLD vs LLD, back-of-envelope estimation, scaling, load balancing, latency/throughput, CAP theorem |
| [Caching Fundamentals](introduction-to-system-design/caching-fundamentals/) | What is caching, cache hits/misses, eviction policies, CDN, browser vs server cache |
| [Database Fundamentals](introduction-to-system-design/database-fundamentals/) | SQL vs NoSQL, CRUD, ACID, indexing, transactions, backups |
| [Computer Networking](introduction-to-system-design/computer-networking-protocols-basics/) | TCP, UDP, IP, DNS, TLS, networking protocols |
| [REST API Fundamentals](introduction-to-system-design/fundamentals-of-rest-api/) | REST APIs, HTTP status codes, authentication, pagination, versioning, rate limiting |
| [Storage Fundamentals](introduction-to-system-design/storage-fundamentals/) | Disk vs memory, hot/warm/cold data, storage tiers |
| [Basics of Computing](introduction-to-system-design/basics-of-computing/) | How computers work, binary, memory, CPU |

---

## Intermediate System Design

Patterns and techniques for building scalable, reliable systems.

| Section | Topics |
|---------|--------|
| [Database Patterns](system-design-intermediate/database/) | Sharding, replication, consistent hashing, master-slave architecture, distributed databases |
| [Caching Patterns](system-design-intermediate/caching-patterns/) | Cache-aside, write-through, read-through, write-behind, comparison |
| [Event-Driven Architecture](system-design-intermediate/event-driven-architecture/) | Event streams, Kafka, RabbitMQ, MQTT, pub-sub, dead letter queues |
| [Software Architecture Patterns](system-design-intermediate/software-architecture-patterns/) | Client-server, stateful vs stateless, async vs sync, proxies, webhooks, microservices |
| [API Protocols](system-design-intermediate/api-communication-protocols-and-architectural-styles/) | GraphQL, gRPC, Protocol Buffers, WebRTC, SMTP |
| [Search & Storage Systems](system-design-intermediate/search-and-storage/) | Search systems, inverted indexes, blob/object storage, CDN deep dive |
| [Reliability Patterns](system-design-intermediate/reliability-patterns/) | Rate limiting (token bucket, sliding window), circuit breakers |
| [Software Testing](system-design-intermediate/fundamentals-of-software-testing/) | Smoke, functional, integration, API, system, E2E, stress, chaos testing |
| [Monitoring & Observability](system-design-intermediate/software-monitoring%20-and-observibility/) | Logging, observability, health checks |

---

## Advanced System Design

Deep distributed systems theory and real-world system design problems.

| Section | Topics |
|---------|--------|
| [Distributed Algorithms](system-design-advanced/distributed-algorithms/) | Vector clocks, BFT, Raft/Paxos consensus, 2PC/3PC, quorum, gossip, Merkle trees, hinted handoff |
| [Event-Driven Architecture](system-design-advanced/event-driven-architecture/) | Event sourcing at scale, complex event processing, outbox/inbox patterns, event mesh, Kappa vs Lambda |
| [Architectural Patterns](system-design-advanced/architectural-patterns/) | Saga patterns, strangler fig, BFF, API gateway vs service mesh, multi-tenancy, hexagonal architecture |
| [Data Management](system-design-advanced/data-management/) | CRDTs, MVCC, locking strategies, database federation, polyglot persistence, LSM trees, bloom filters |
| [Caching & Performance](system-design-advanced/caching-and-performance/) | Thundering herd, distributed locks, cache coherence, edge computing |
| [Observability](system-design-advanced/observability/) | Tail-based sampling, OpenTelemetry, high-cardinality data |
| [Reliability & Resilience](system-design-advanced/reliability-and-resilience/) | Game days, blast radius, SLIs/SLOs, multi-region active-active, cascading failure prevention |
| [System Design Problems](system-design-advanced/system-design-problems/) | URL Shortener, Chat System, News Feed, Video Streaming, Distributed Cache, Payment System, Notification System, Web Crawler |

---

## Repository Structure

```
System-Design-101/
├── introduction-to-system-design/     # Beginner fundamentals
├── system-design-intermediate/        # Intermediate patterns
├── system-design-advanced/            # Advanced distributed systems
└── CONTRIBUTING.md
```

---

*Continue learning interactively at [algoroq.com](https://algoroq.com) — structured learning paths, progress tracking, and hands-on exercises.*
