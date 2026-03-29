# Latency and Throughput

Latency and throughput are the two fundamental metrics for measuring system performance. Understanding them — and the tension between them — is essential for designing systems that feel fast and can handle load.

---

## What is Latency?

**Latency** is the time it takes for a single request to travel from client to server and back. It's what users *feel* — the delay between clicking a button and seeing a result.

```
┌──────────┐                              ┌──────────┐
│          │ ─── Request (t=0ms) ───────▶ │          │
│  Client  │                              │  Server  │
│          │ ◀── Response (t=120ms) ───── │          │
└──────────┘                              └──────────┘

              Latency = 120ms
```

### Components of Latency

```
Total Latency
├── Network latency (request travel time)
├── Server processing time
│   ├── Application logic
│   ├── Database query
│   └── External service calls
└── Network latency (response travel time)
```

**Example breakdown:**
| Component | Time |
|-----------|------|
| DNS lookup | 10 ms |
| TCP handshake | 20 ms |
| TLS handshake | 30 ms |
| Request to server | 15 ms |
| Server processing | 35 ms |
| Database query | 25 ms |
| Response to client | 15 ms |
| **Total** | **150 ms** |

---

## Why Averages Lie: Percentile Latencies

Average latency hides the worst-case experience. If 99 requests take 50ms and 1 request takes 5,000ms, the average is 99.5ms — but one user waited 5 seconds.

**Percentiles give a better picture:**

| Metric | Meaning | What It Tells You |
|--------|---------|-------------------|
| p50 (median) | 50% of requests are faster | Typical user experience |
| p95 | 95% of requests are faster | Most users' worst experience |
| p99 | 99% of requests are faster | Edge cases and tail latency |
| p99.9 | 99.9% of requests are faster | Extreme outliers |

```
  Request Latency Distribution

  Count
   ▲
   │  ╭──╮
   │ ╭╯  ╰╮
   │╭╯    ╰─╮
   ││       ╰──╮
   ││          ╰────╮
   ││               ╰──────────────────╮  ← "long tail"
   └┴───────────────────────────────────┴──▶ Latency
     p50    p95   p99              p99.9
    (50ms) (200ms)(800ms)        (3000ms)
```

### Why Tail Latency Matters

Users making multiple requests experience the *slowest* call. If a page makes 10 API calls, the probability of hitting a p99 latency is:

```
P(at least one slow call) = 1 - (0.99)^10 = 9.6%
```

Nearly 1 in 10 page loads will be slow! This is why Amazon obsesses over p99.9.

---

## What is Throughput?

**Throughput** is the number of operations a system can handle per unit of time. While latency measures the speed of one request, throughput measures total capacity.

Common units:
| Metric | Stands For | Example |
|--------|-----------|---------|
| QPS | Queries Per Second | Database: 50,000 QPS |
| RPS | Requests Per Second | API: 10,000 RPS |
| TPS | Transactions Per Second | Payment: 500 TPS |
| Mbps/Gbps | Megabits/Gigabits per second | Network: 10 Gbps |

---

## Latency vs Throughput: The Relationship

Low latency and high throughput are not the same thing, and optimizing for one can hurt the other.

**Analogy: Highway traffic**
- **Latency** = how long it takes *one car* to travel from A to B
- **Throughput** = how many cars pass a point per hour

A single sports car (low latency) doesn't mean high throughput. A highway packed bumper-to-bumper has high throughput but each car moves slowly (high latency).

```
   Low Latency, Low Throughput        High Throughput, Higher Latency
   (one car, open road)               (many cars, some congestion)

   🚗─────────────────▶               🚗🚗🚗🚗🚗──────────▶
                                       🚗🚗🚗🚗🚗──────────▶
   Fast, but only 1 car               Slower, but 10 cars/minute
```

### The Trade-off in Practice

| Optimization | Improves | May Hurt |
|-------------|----------|----------|
| Batching requests | Throughput | Latency (waiting to fill batch) |
| Connection pooling | Both | Memory usage |
| Caching | Both | Consistency |
| Compression | Bandwidth throughput | CPU latency |
| Parallel processing | Throughput | Latency (coordination overhead) |

---

## Latency Numbers Every Engineer Should Know

These numbers help you make quick, informed decisions about architecture:

| Operation | Latency | Notes |
|-----------|---------|-------|
| L1 cache reference | 0.5 ns | On the CPU die |
| L2 cache reference | 7 ns | Still on chip |
| Main memory (RAM) reference | 100 ns | ~200x slower than L1 |
| SSD random read | 150 μs | 1,500x slower than RAM |
| HDD random read | 10 ms | 67x slower than SSD |
| Send 1 KB within datacenter | 250 μs | Network hop |
| Read 1 MB sequentially from SSD | 1 ms | |
| Read 1 MB sequentially from HDD | 20 ms | |
| Round trip within datacenter | 0.5 ms | Same availability zone |
| Round trip: US East ↔ US West | 40 ms | Cross-country |
| Round trip: US ↔ Europe | 80 ms | Across Atlantic |
| Round trip: US ↔ Asia | 150 ms | Across Pacific |

### Key Insights from These Numbers

1. **Memory is 1,000x faster than SSD** — cache aggressively
2. **SSD is 100x faster than HDD** — use SSDs for databases
3. **Network within datacenter is fast** — microservices are viable
4. **Cross-continent is slow** — use CDNs and regional deployments
5. **Sequential reads are much faster than random reads** — design for sequential access when possible

---

## Optimization Techniques for Latency

### 1. Caching
The most impactful optimization. Store computed results closer to the user.

```
Without cache:  Client → Server → Database (150ms)
With cache:     Client → Server → Cache (15ms)
                                    ↓ miss
                                  Database (150ms)
```

### 2. Connection Pooling
Reuse database connections instead of creating new ones per request.

```
Without pooling:  TCP handshake + TLS + auth = 50ms per query
With pooling:     Reuse existing connection = 2ms per query
```

### 3. Geographic Distribution
Move servers closer to users.

| Deployment | US User Latency | Asia User Latency |
|-----------|----------------|-------------------|
| US-only | 20 ms | 170 ms |
| US + Asia regions | 20 ms | 20 ms |

### 4. Async Processing
Move non-critical work out of the request path.

```
   Synchronous (all in request):
   ┌─────┐─▶┌──────┐─▶┌───────┐─▶┌──────┐  Response: 350ms
   │Parse│  │Save  │  │Send   │  │Update│
   │ 20ms│  │150ms │  │Email  │  │Stats │
   └─────┘  └──────┘  │ 100ms │  │ 80ms │
                       └───────┘  └──────┘

   Async (email + stats in background):
   ┌─────┐─▶┌──────┐─▶┌───────┐  Response: 175ms
   │Parse│  │Save  │  │Enqueue│
   │ 20ms│  │150ms │  │  5ms  │  (email + stats processed later)
   └─────┘  └──────┘  └───────┘
```

### 5. Data Compression
Trade CPU cycles for network bandwidth.

### 6. Prefetching
Predict what the user will need next and load it before they ask.

---

## Optimization Techniques for Throughput

### 1. Horizontal Scaling
Add more machines behind a load balancer.

### 2. Batching
Group multiple operations into one.

```
Without batching:  100 individual INSERTs → 100 round trips
With batching:     1 bulk INSERT of 100 rows → 1 round trip
```

### 3. Load Shedding
When overloaded, reject some requests gracefully rather than slowing down for everyone.

### 4. Backpressure
Signal upstream systems to slow down when you can't keep up.

```
   Producer → Queue (filling up) → Consumer (slow)
                    │
                    └── "Slow down!" → Producer
```

### 5. Read/Write Separation
Route reads to replicas, writes to primary.

---

## Measuring Latency and Throughput

### Tools
| Tool | What It Measures |
|------|-----------------|
| `ping` | Network round-trip time |
| `traceroute` | Latency per network hop |
| `wrk` / `hey` | HTTP throughput and latency distribution |
| Prometheus + Grafana | Continuous monitoring with percentiles |
| Jaeger / Zipkin | Distributed tracing across services |

### What to Monitor in Production
- **p50, p95, p99 latency** — not averages
- **Requests per second** — total and per endpoint
- **Error rate** — failures under load indicate capacity limits
- **Saturation** — how close are you to maximum capacity?

---

## Key Takeaways

1. **Latency is per-request speed; throughput is total capacity** — optimize both, but know which matters more for your use case
2. **Use percentiles, not averages** — p99 reveals the real user experience
3. **Tail latency compounds** — one slow dependency slows everything
4. **Cache everything you can** — it's the single biggest latency win
5. **Move work out of the request path** — async processing improves both latency and throughput
6. **Know your latency numbers** — they inform architecture decisions instantly
7. **Measure in production** — synthetic benchmarks lie about real-world performance

---

*Previous: [Scalability ←](03-scalability.md) · Next: [CAP Theorem →](05-cap-theorem.md)*
