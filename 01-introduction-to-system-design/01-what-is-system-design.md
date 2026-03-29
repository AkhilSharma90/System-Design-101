# What is System Design?

System design is the process of defining the architecture, components, and data flow of a system to satisfy a set of requirements. It's how engineers take a vague problem like "build Twitter" and turn it into a concrete plan with servers, databases, caches, and communication protocols.

---

## Why System Design Matters

Every application you use — from Instagram to your banking app — is backed by a system that was designed by engineers making deliberate trade-offs. As software grows in users and complexity, the design of the underlying system becomes the single biggest factor in whether it works reliably or falls apart.

**System design matters because:**

- A poorly designed system will fail under load, no matter how clean the code is
- Architectural decisions are expensive to change later — they affect every layer of the stack
- It's the primary evaluation criteria for senior engineering roles at top companies
- It bridges the gap between "I can write code" and "I can build products that scale"

---

## Functional vs Non-Functional Requirements

Every system design starts by separating two types of requirements:

### Functional Requirements

These describe **what the system should do** — the features and behaviors.

> *Example for a URL shortener:*
> - Users can submit a long URL and receive a shortened URL
> - When someone visits the short URL, they are redirected to the original
> - Users can optionally set a custom alias
> - Links expire after a configurable time period

### Non-Functional Requirements

These describe **how the system should perform** — the quality attributes.

> *Example for a URL shortener:*
> - The system should handle 100 million URLs created per month
> - URL redirection should happen in under 100ms (p99)
> - The system should be highly available (99.99% uptime)
> - Shortened URLs should not be predictable (security)

```
┌─────────────────────────────────────────────────────┐
│                  System Requirements                │
├──────────────────────┬──────────────────────────────┤
│   Functional         │   Non-Functional             │
│                      │                              │
│   • Features         │   • Performance              │
│   • User actions     │   • Scalability              │
│   • Data operations  │   • Availability             │
│   • Business rules   │   • Security                 │
│                      │   • Consistency               │
│                      │   • Maintainability           │
│   "What it does"     │   "How well it does it"      │
└──────────────────────┴──────────────────────────────┘
```

---

## The Building Blocks of System Design

At a high level, every system is composed of a few fundamental building blocks:

### Clients and Servers

The most basic architecture — a client (browser, mobile app) sends requests to a server that processes them and returns responses.

```
┌──────────┐         HTTP Request          ┌──────────┐
│          │ ──────────────────────────────▶│          │
│  Client  │                               │  Server  │
│          │◀────────────────────────────── │          │
└──────────┘         HTTP Response          └──────────┘
```

### Databases

Persistent storage for your application's data. The choice between SQL and NoSQL databases is one of the most consequential early decisions.

### Caches

Fast, temporary storage (usually in-memory) that sits between your application and the database to speed up reads.

### Load Balancers

Distribute incoming traffic across multiple servers so no single server gets overwhelmed.

### Message Queues

Enable asynchronous communication between services — a producer sends a message, and a consumer processes it later.

### CDNs (Content Delivery Networks)

Geographically distributed servers that cache and serve static content (images, CSS, JavaScript) close to users.

```
┌─────────────────────────────────────────────────────────────┐
│                    Typical Web Architecture                  │
│                                                             │
│   ┌────────┐     ┌──────────────┐     ┌──────────────┐     │
│   │  CDN   │     │    Load      │     │   App Server │     │
│   │        │     │   Balancer   │────▶│   Instance 1 │──┐  │
│   └───┬────┘     │              │     └──────────────┘  │  │
│       │          │              │     ┌──────────────┐  │  │
│       │          │              │────▶│   App Server │  │  │
│       │          │              │     │   Instance 2 │──┤  │
│       │          └──────┬───────┘     └──────────────┘  │  │
│       │                 │             ┌──────────────┐  │  │
│       │                 └────────────▶│   App Server │  │  │
│       │                               │   Instance 3 │──┤  │
│       │                               └──────────────┘  │  │
│       │                                                 │  │
│       │          ┌──────────────┐     ┌──────────────┐  │  │
│       │          │    Cache     │     │   Database   │  │  │
│       │          │   (Redis)   │◀────│  (Primary)   │◀─┘  │
│       │          └──────────────┘     └──────┬───────┘     │
│       │                                      │             │
│       │                               ┌──────┴───────┐     │
│       │                               │   Database   │     │
│       │                               │  (Replica)   │     │
│       │                               └──────────────┘     │
│  Users│                                                     │
│◀──────┘                                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Properties of a Well-Designed System

When evaluating any system design, these are the properties you should think about:

### Scalability
Can the system handle growth — more users, more data, more requests? A scalable system can grow without a complete redesign.

### Reliability
Does the system continue to work correctly even when things go wrong? Hardware fails, networks partition, bugs ship — a reliable system anticipates these failures.

### Availability
What percentage of time is the system operational? This is typically measured in "nines":

| Availability | Downtime per Year | Downtime per Month |
|-------------|-------------------|-------------------|
| 99% (two nines) | 3.65 days | 7.3 hours |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.38 minutes |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds |

### Maintainability
Can engineers understand, modify, and extend the system without introducing bugs? This includes code quality, documentation, and operational simplicity.

### Cost Efficiency
Does the system use resources (compute, storage, bandwidth) efficiently? Over-engineering is as wasteful as under-engineering.

---

## System Design vs. Coding

System design and coding are complementary but different skills:

| Aspect | Coding | System Design |
|--------|--------|---------------|
| Focus | Correctness of algorithms and logic | Architecture and trade-offs |
| Scale | Single machine | Multiple machines and services |
| Constraints | Time/space complexity | Availability, latency, cost |
| Output | Working code | Architecture diagrams and plans |
| Change cost | Refactor a function | Potentially rewrite entire services |

You need both. But as you grow in your career, the ability to design systems becomes more important than the ability to write individual functions.

---

## What's Next

Now that you understand what system design is and why it matters, the next article covers [How to Approach System Design Problems](02-how-to-approach-system-design.md) — a repeatable framework you can use for any design question.

---

*Next: [How to Approach System Design Problems →](02-how-to-approach-system-design.md)*
