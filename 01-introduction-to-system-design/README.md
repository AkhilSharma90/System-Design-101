# Introduction to System Design

This section builds your foundation in system design. If you're new to thinking about how large-scale systems work, start here.

## What You'll Learn

By the end of this section, you'll understand:

- What system design is and why it matters for your career
- A repeatable framework for approaching any design problem
- Core concepts: scalability, latency, throughput, and the CAP theorem
- Essential building blocks: load balancers, caches, databases, and APIs
- How the internet actually works under the hood

## Articles

| # | Article | Key Concepts |
|---|---------|-------------|
| 1 | [What is System Design?](01-what-is-system-design.md) | Functional vs non-functional requirements, system properties |
| 2 | [How to Approach System Design Problems](02-how-to-approach-system-design.md) | 4-step framework, requirement gathering, estimation |
| 3 | [Scalability](03-scalability.md) | Vertical vs horizontal scaling, stateless design |
| 4 | [Latency and Throughput](04-latency-and-throughput.md) | P99 latency, QPS, bandwidth, optimization techniques |
| 5 | [CAP Theorem](05-cap-theorem.md) | Consistency, availability, partition tolerance trade-offs |
| 6 | [Load Balancing](06-load-balancing.md) | Round robin, least connections, L4 vs L7 balancing |
| 7 | [Caching Fundamentals](07-caching-fundamentals.md) | Cache-aside, write-through, eviction policies |
| 8 | [Databases — SQL vs NoSQL](08-databases-sql-vs-nosql.md) | ACID, BASE, relational vs document vs key-value stores |
| 9 | [Networking Essentials](09-networking-essentials.md) | DNS resolution, TCP handshake, HTTP/HTTPS, WebSockets |
| 10 | [API Design](10-api-design.md) | REST principles, GraphQL queries, gRPC and protobufs |

## Prerequisites

No prior system design knowledge required. Basic programming experience is helpful.

## Next Steps

Once you've completed this section, move on to [Intermediate System Design](../02-intermediate-system-design/README.md) to learn about production-grade building blocks like sharding, replication, and message queues.
