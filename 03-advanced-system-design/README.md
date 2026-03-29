# Advanced System Design

This is where everything comes together. In this section, you'll design complete, production-grade systems — the same problems engineers face at Google, Meta, Stripe, and Netflix.

## What You'll Learn

By the end of this section, you'll be able to:

- Understand distributed consensus and why it's the hardest problem in distributed systems
- Design event-driven architectures using CQRS and event sourcing
- Design 8 real-world systems from requirements to architecture:
  - URL Shortener, Chat System, News Feed, Video Streaming
  - Distributed Cache, Payment System, Notification System, Web Crawler
- Make and justify architectural trade-offs under real constraints
- Handle edge cases, failure modes, and scaling challenges

## Prerequisites

You should be comfortable with all concepts in both [Introduction](../01-introduction-to-system-design/README.md) and [Intermediate](../02-intermediate-system-design/README.md) sections.

## Articles

| # | Article | Type | Key Concepts |
|---|---------|------|-------------|
| 1 | [Distributed Consensus](01-distributed-consensus.md) | Concept | Raft, Paxos, leader election, split-brain |
| 2 | [Event-Driven Architecture](02-event-driven-architecture.md) | Concept | CQRS, event sourcing, saga pattern |
| 3 | [Design a URL Shortener](03-design-url-shortener.md) | Design | Key generation, redirection, analytics |
| 4 | [Design a Chat System](04-design-chat-system.md) | Design | WebSockets, presence, message ordering |
| 5 | [Design a News Feed](05-design-news-feed.md) | Design | Fan-out, ranking, push vs pull |
| 6 | [Design a Video Streaming Platform](06-design-video-streaming.md) | Design | Transcoding, adaptive bitrate, CDN |
| 7 | [Design a Distributed Cache](07-design-distributed-cache.md) | Design | Consistent hashing, eviction, replication |
| 8 | [Design a Payment System](08-design-payment-system.md) | Design | Idempotency, double-entry, reconciliation |
| 9 | [Design a Notification System](09-design-notification-system.md) | Design | Multi-channel delivery, priority queues, dedup |
| 10 | [Design a Web Crawler](10-design-web-crawler.md) | Design | URL frontier, politeness, deduplication |

## How to Use This Section

Each design article follows a consistent structure:

1. **Requirements** — Functional and non-functional
2. **Estimation** — Back-of-envelope calculations
3. **High-Level Design** — Architecture overview
4. **Deep Dives** — Detailed component design
5. **Scaling** — How to evolve the system
6. **Trade-offs** — Key decisions and alternatives

Try designing each system yourself before reading the solution. The learning happens in the struggle.
