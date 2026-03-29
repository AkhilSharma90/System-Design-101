# Scalability

Scalability is a system's ability to handle increased load by adding resources. It's the difference between a system that works for 1,000 users and one that works for 1,000,000.

---

## Why Scalability Matters

Every successful application eventually faces a scaling challenge. The question isn't *if* you'll need to scale — it's *when* and *how*. A system that can't scale becomes the ceiling on your product's growth.

Consider this: Twitter went from 20,000 tweets per day in 2007 to 500 million per day by 2013. No single server on earth could handle that growth without fundamental architectural changes.

---

## Vertical Scaling (Scale Up)

Vertical scaling means adding more power to your existing machine — more CPU, RAM, faster SSDs.

```
   Before                    After
┌──────────┐          ┌──────────────┐
│  Server  │          │    Server    │
│          │          │              │
│  4 CPU   │   ──▶    │   32 CPU     │
│  8 GB    │          │   256 GB     │
│  256 SSD │          │   4 TB SSD   │
└──────────┘          └──────────────┘
```

### Advantages
- **Simple** — no code changes required
- **No distributed system complexity** — no data consistency issues
- **Strong consistency** — single machine, single source of truth
- **Low latency** — no network calls between components

### Limitations
- **Hardware ceiling** — even the most powerful server has limits
- **Single point of failure** — one machine goes down, everything goes down
- **Cost curve** — doubling capacity often more than doubles cost
- **Downtime for upgrades** — swapping hardware usually requires downtime

### When to Scale Vertically
- Early-stage applications with predictable growth
- Database servers where strong consistency is critical
- Applications with complex in-memory state
- When engineering time is more expensive than hardware

---

## Horizontal Scaling (Scale Out)

Horizontal scaling means adding more machines to your pool of resources.

```
   Before                    After
┌──────────┐          ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Server  │          │ Server 1 │ │ Server 2 │ │ Server 3 │
│          │   ──▶    │          │ │          │ │          │
│  4 CPU   │          │  4 CPU   │ │  4 CPU   │ │  4 CPU   │
│  8 GB    │          │  8 GB    │ │  8 GB    │ │  8 GB    │
└──────────┘          └──────────┘ └──────────┘ └──────────┘
                              │          │          │
                      ────────┴──────────┴──────────┘
                                    │
                           ┌────────┴────────┐
                           │  Load Balancer  │
                           └─────────────────┘
```

### Advantages
- **No hardware ceiling** — add machines as needed
- **Fault tolerance** — if one server dies, others continue serving
- **Cost-effective** — use commodity hardware
- **Geographic distribution** — place servers closer to users

### Challenges
- **Distributed system complexity** — data consistency, network failures
- **Stateless requirement** — servers can't rely on local state
- **Data partitioning** — deciding how to split data across machines
- **Operational overhead** — more machines to deploy, monitor, and maintain

---

## Vertical vs Horizontal Scaling

| Factor | Vertical | Horizontal |
|--------|----------|------------|
| Complexity | Low | High |
| Max capacity | Hardware limits | Virtually unlimited |
| Failure tolerance | Single point of failure | Redundant |
| Cost at scale | Exponential | Linear |
| Downtime for scaling | Usually required | Zero downtime |
| Data consistency | Simple | Complex |
| Best for | Databases, small apps | Web servers, APIs |

**The rule of thumb:** Start vertical, go horizontal when you hit limits. Most real systems use both.

---

## Stateless vs Stateful Architecture

The key to horizontal scaling is making your servers **stateless**.

### Stateful Server (Hard to Scale)

Each server stores user session data locally. If a user's request goes to a different server, their session is lost.

```
┌──────────┐     ┌──────────────────────┐
│          │────▶│ Server 1             │
│  User A  │     │ Session: {userA: ..} │
│          │     └──────────────────────┘
└──────────┘
                 ┌──────────────────────┐
                 │ Server 2             │
                 │ Session: {userB: ..} │  ← User A can't go here!
                 └──────────────────────┘
```

### Stateless Server (Easy to Scale)

Session data is stored externally (Redis, database). Any server can handle any request.

```
┌──────────┐     ┌──────────────┐
│          │────▶│  Server 1    │──┐
│  User A  │     │  (no state)  │  │     ┌─────────────────┐
│          │     └──────────────┘  ├────▶│ Session Store    │
└──────────┘     ┌──────────────┐  │     │ (Redis)         │
                 │  Server 2    │──┘     │                 │
                 │  (no state)  │        │ userA: {..}     │
                 └──────────────┘        │ userB: {..}     │
                                         └─────────────────┘
```

**Key principle:** If you can swap out any server without users noticing, your architecture is stateless.

---

## Database Scaling Strategies

The database is usually the hardest component to scale. Here's a quick overview (each gets its own article later):

### Read Replicas
Offload read queries to copies of the database.

```
┌──────────┐     Writes     ┌──────────────┐
│   App    │───────────────▶│   Primary    │
│  Server  │                │   Database   │
│          │     Reads      ├──────────────┤
│          │◀───────────────│   Replica 1  │
│          │◀───────────────│   Replica 2  │
└──────────┘                └──────────────┘
```

### Caching
Reduce database load by caching frequently accessed data.

### Sharding
Split data across multiple databases based on a shard key.

### Denormalization
Trade storage space for read performance by duplicating data.

---

## Auto-Scaling

Modern cloud platforms can automatically adjust capacity based on demand.

```
                    CPU Usage Over Time
   100% ┤
    80% ┤          ╭──╮     Scale up triggered
    60% ┤       ╭──╯  ╰──╮
    40% ┤    ╭──╯        ╰──╮
    20% ┤ ╭──╯              ╰──╮   Scale down triggered
     0% ┤─╯                    ╰──────
        └────────────────────────────────
         6am    12pm    6pm    12am
```

### Key Auto-Scaling Metrics
- **CPU utilization** — scale when average CPU > 70%
- **Memory usage** — scale before OOM kills start
- **Request queue depth** — scale when requests are waiting
- **Custom metrics** — business-specific (orders/sec, active games)

### Auto-Scaling Policies
| Policy | Description | Use Case |
|--------|-------------|----------|
| Target tracking | Maintain metric at target value | Keep CPU at 50% |
| Step scaling | Add/remove fixed number of instances at thresholds | Add 2 instances when CPU > 80% |
| Scheduled scaling | Scale based on known patterns | Double capacity before Black Friday |
| Predictive scaling | ML-based prediction of future load | Anticipate traffic spikes |

### Cooldown Periods
After scaling, wait before making another decision to avoid oscillation. Typical cooldown: 3–5 minutes.

---

## Common Scalability Bottlenecks

### 1. Database
The most common bottleneck. Solutions: read replicas, caching, sharding, query optimization.

### 2. Single Points of Failure
Any component without redundancy. Solutions: replicate everything, use load balancers.

### 3. Synchronous Processing
Doing everything in the request path. Solutions: use message queues for non-critical tasks.

```
   Synchronous (slow)              Asynchronous (fast)
┌──────┐  ┌──────┐  ┌──────┐    ┌──────┐  ┌───────┐
│ API  │─▶│Email │─▶│ Log  │    │ API  │─▶│ Queue │
│      │  │Send  │  │Write │    │      │  │       │
│200ms │  │500ms │  │100ms │    │200ms │  │  5ms  │
└──────┘  └──────┘  └──────┘    └──────┘  └───┬───┘
Total: 800ms                    Total: 205ms  │
                                         ┌────▼────┐
                                         │ Workers │
                                         │(async)  │
                                         └─────────┘
```

### 4. Hot Keys / Hot Partitions
A small number of keys receiving disproportionate traffic. Solutions: key splitting, caching, request coalescing.

### 5. Network Bandwidth
Transferring too much data between services. Solutions: compression, pagination, protocol buffers.

---

## Real-World Scaling Examples

### Netflix
- **800+ microservices** handling billions of API requests daily
- Horizontal scaling with AWS Auto Scaling Groups
- EVCache (Memcached-based) handles 30+ million requests/second
- Multiple AWS regions for global availability

### Uber
- Started as a monolithic PHP app
- Migrated to 4,000+ microservices
- Uses Google's S2 geometry library for geographic partitioning
- Real-time dispatch processes millions of location updates/second

### Instagram
- Sharded PostgreSQL across dozens of machines
- Memcached with 6+ TB of cache
- Django application servers behind load balancers
- Scaled from 0 to 1 billion users with incremental improvements

---

## Key Takeaways

1. **Start simple, scale incrementally** — don't over-engineer day one
2. **Vertical scaling is a valid starting point** — simpler and often sufficient
3. **Horizontal scaling requires stateless design** — externalize all state
4. **The database is usually the bottleneck** — plan for it early
5. **Auto-scaling handles variable load** — but set appropriate limits
6. **Measure before you optimize** — profile to find real bottlenecks
7. **Every scaling decision is a trade-off** — complexity vs capacity vs cost

---

*Previous: [How to Approach System Design Problems ←](02-how-to-approach-system-design.md) · Next: [Latency and Throughput →](04-latency-and-throughput.md)*
