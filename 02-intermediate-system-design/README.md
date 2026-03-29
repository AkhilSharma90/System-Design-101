# Intermediate System Design

This section takes you deeper into the building blocks that power production systems at scale. You'll learn how companies like Netflix, Uber, and Slack build systems that handle millions of concurrent users.

## What You'll Learn

By the end of this section, you'll understand:

- How databases scale beyond a single machine using sharding and replication
- Why consistent hashing is everywhere in distributed systems
- How message queues decouple services and improve reliability
- Rate limiting strategies that protect your systems from abuse
- How CDNs serve content to billions of users worldwide
- The microservices vs monolith debate — with real trade-offs, not dogma
- How search engines find results in milliseconds
- How blob storage works at massive scale
- How to monitor and observe complex distributed systems

## Prerequisites

You should be comfortable with all concepts in [Introduction to System Design](../01-introduction-to-system-design/README.md) before starting this section.

## Articles

| # | Article | Key Concepts |
|---|---------|-------------|
| 1 | [Database Sharding](01-database-sharding.md) | Horizontal partitioning, shard keys, resharding strategies |
| 2 | [Consistent Hashing](02-consistent-hashing.md) | Hash rings, virtual nodes, minimal disruption |
| 3 | [Message Queues](03-message-queues.md) | Pub/sub, Kafka, RabbitMQ, exactly-once delivery |
| 4 | [Database Replication](04-database-replication.md) | Leader-follower, multi-leader, quorum reads/writes |
| 5 | [Rate Limiting](05-rate-limiting.md) | Token bucket, sliding window, distributed rate limiting |
| 6 | [CDN and Edge Computing](06-cdn-and-edge-computing.md) | Push vs pull CDN, edge functions, cache hierarchies |
| 7 | [Microservices vs Monolith](07-microservices-vs-monolith.md) | Service boundaries, data ownership, migration patterns |
| 8 | [Search Systems](08-search-systems.md) | Inverted indexes, tokenization, relevance scoring |
| 9 | [Blob Storage and Object Storage](09-blob-storage.md) | S3 architecture, chunking, replication, CDN integration |
| 10 | [Monitoring and Observability](10-monitoring-and-observability.md) | Metrics, logs, traces, alerting, SLOs/SLIs/SLAs |

## Next Steps

After completing this section, you're ready for [Advanced System Design](../03-advanced-system-design/README.md) — where you'll design complete systems like a chat application, a payment platform, and a video streaming service.
