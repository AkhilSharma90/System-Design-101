# Load Balancing

A load balancer is the traffic cop of your infrastructure. It sits in front of your servers, receives all incoming requests, and distributes them intelligently so no single server becomes a bottleneck.

---

## Why Load Balancing?

Without a load balancer, you have a single server handling all traffic. This creates two problems:

1. **Capacity limit** — one server can only handle so much
2. **Single point of failure** — if it dies, everything dies

```
   Without Load Balancer          With Load Balancer

   ┌─────────┐                   ┌─────────┐
   │ Users   │──▶ ┌──────────┐   │ Users   │──▶ ┌────────────┐
   │ (many)  │    │  Server  │   │ (many)  │    │    Load    │
   │         │    │(overload)│   │         │    │  Balancer  │
   └─────────┘    └──────────┘   └─────────┘    └─────┬──────┘
                  Server dies                         │
                  = 100% outage              ┌────────┼────────┐
                                             ▼        ▼        ▼
                                          ┌──────┐┌──────┐┌──────┐
                                          │ Srv1 ││ Srv2 ││ Srv3 │
                                          └──────┘└──────┘└──────┘
                                          Srv1 dies = ~33% capacity lost
```

---

## Where Load Balancers Live

Load balancers can appear at multiple layers of your architecture:

```
Internet
    │
    ▼
┌──────────────────┐
│   DNS / GeoDNS   │  ← Layer: Routes between datacenters
└────────┬─────────┘
         │
    ▼
┌──────────────────┐
│  Global LB (L4)  │  ← Layer: TCP/UDP — fast, low overhead
└────────┬─────────┘
         │
    ▼
┌──────────────────┐
│   App LB (L7)    │  ← Layer: HTTP — content-aware routing
└────────┬─────────┘
         │
   ┌─────┴──────┐
   ▼            ▼
┌──────┐    ┌──────┐
│ App  │    │ App  │
│Srv 1 │    │Srv 2 │
└──────┘    └──────┘
   │            │
   └─────┬──────┘
         ▼
┌──────────────────┐
│   DB Load Bal.   │  ← Layer: Routes reads to replicas, writes to primary
└──────────────────┘
```

---

## Load Balancing Algorithms

### 1. Round Robin
Requests are distributed sequentially across all servers.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (cycle repeats)
```

**Pro:** Simple, even distribution
**Con:** Ignores server load — a slow server still gets equal requests

### 2. Weighted Round Robin
Servers get requests proportional to their capacity.

```
Server A (weight 3): handles 3 requests per cycle
Server B (weight 1): handles 1 request per cycle

Cycle: A → A → A → B → A → A → A → B ...
```

**Pro:** Accounts for different server capacities
**Con:** Static weights don't adapt to runtime conditions

### 3. Least Connections
Send each new request to the server with the fewest active connections.

```
Server A: 10 active connections
Server B: 3 active connections  ← new request goes here
Server C: 7 active connections
```

**Pro:** Adapts to varying request durations
**Con:** Slightly more overhead tracking connection counts

### 4. Least Response Time
Routes to the server with the lowest combination of active connections and average response time.

**Pro:** Best for heterogeneous servers with varying performance
**Con:** More complex to implement

### 5. IP Hash
Hash the client's IP address to determine which server handles their requests.

```
SHA256(client_ip) % num_servers = server_index
```

**Pro:** Same client always hits same server (session affinity without cookies)
**Con:** Uneven distribution if client IPs cluster; reshuffles on server changes

### 6. Random
Pick a server at random. Works surprisingly well at scale.

**Pro:** No coordination overhead, zero state
**Con:** Can create imbalance with small server counts

---

## Algorithm Comparison

| Algorithm | Session Affinity | Adapts to Load | Complexity | Best For |
|-----------|-----------------|----------------|------------|----------|
| Round Robin | No | No | Lowest | Stateless, uniform requests |
| Weighted Round Robin | No | Partially | Low | Mixed capacity servers |
| Least Connections | No | Yes | Medium | Long-lived connections (WebSocket) |
| Least Response Time | No | Yes | Medium | Heterogeneous workloads |
| IP Hash | Yes | No | Low | Session-heavy apps |
| Random | No | No | Lowest | Large, homogeneous fleets |

---

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer)
Operates on IP addresses and TCP/UDP ports. Forwards packets without inspecting content.

```
┌──────────────────────────────────────────────────────┐
│              L4 Load Balancer                        │
│                                                      │
│  Sees: src_ip, dst_ip, src_port, dst_port, protocol  │
│  Doesn't see: HTTP headers, URLs, cookies            │
│                                                      │
│  Decision: route based on IP + port only             │
│  Speed: Very fast (no packet inspection)             │
└──────────────────────────────────────────────────────┘
```

**Use L4 when:** You need maximum throughput and lowest latency. TCP/UDP services that aren't HTTP.

### Layer 7 (Application Layer)
Inspects HTTP content. Can route based on URL, headers, cookies, request body.

```
┌──────────────────────────────────────────────────────┐
│              L7 Load Balancer                        │
│                                                      │
│  Sees: Full HTTP request — method, path, headers,    │
│        cookies, body                                 │
│                                                      │
│  /api/users    → User Service cluster                │
│  /api/payments → Payment Service cluster             │
│  /static/*     → CDN / Object Storage                │
│                                                      │
│  Speed: Slower (inspects every request)              │
│  Capability: SSL termination, A/B testing, canary    │
└──────────────────────────────────────────────────────┘
```

**Use L7 when:** Microservices routing, A/B testing, canary deployments, SSL termination.

### L4 vs L7 Comparison

| Feature | L4 | L7 |
|---------|----|----|
| Protocol awareness | TCP/UDP only | HTTP, gRPC, WebSocket |
| Content-based routing | No | Yes |
| SSL termination | No (pass-through) | Yes |
| Performance | Higher | Slightly lower |
| Canary/A-B testing | No | Yes |
| Health check sophistication | TCP connection | HTTP response + body |
| Examples | AWS NLB, HAProxy (TCP) | AWS ALB, Nginx, Envoy |

---

## Health Checks

Load balancers continuously verify that backend servers are healthy before sending traffic.

### Active Health Checks
The load balancer proactively sends probe requests to each server.

```
Load Balancer                   Server
      │                            │
      │── GET /health ────────────▶│
      │◀── 200 OK (healthy) ───────│
      │                            │
      │   (every 10 seconds)       │
      │── GET /health ────────────▶│
      │◀── 503 Error ──────────────│
      │                            │
      │  Mark server as UNHEALTHY  │
      │  Stop sending traffic      │
```

### Passive Health Checks
Monitor actual traffic for failures.

```
Load Balancer notices:
  5 consecutive 500 responses from Server B
  → Mark Server B as UNHEALTHY
  → Remove from rotation
  → Re-check after cooldown period
```

### Good Health Check Endpoints
```
GET /health → 200 OK
{
  "status": "healthy",
  "db": "connected",
  "cache": "connected",
  "version": "1.4.2"
}
```

Not just "is the process alive" but "can it actually serve requests."

---

## Sticky Sessions (Session Affinity)

Sticky sessions ensure a client always reaches the same server — useful for stateful servers that store session data locally.

```
Client A (cookie: srv=server1) ──▶ Server 1 always
Client B (cookie: srv=server2) ──▶ Server 2 always
```

**Problems with sticky sessions:**
- Uneven load distribution (some users are heavier than others)
- If a server dies, those users lose their session
- Makes rolling deployments harder

**Better approach:** Make servers stateless — store sessions in Redis/database and remove the need for sticky sessions entirely.

---

## Redundant Load Balancers

The load balancer itself can become a single point of failure. The solution: run two load balancers in active-passive or active-active mode.

```
   Active-Passive (Failover):          Active-Active:

   ┌──────────────┐                   ┌──────────────┐
   │    Load      │  Primary          │    Load      │◄──┐
   │  Balancer 1  │──────▶ Servers   │  Balancer 1  │   │
   └──────┬───────┘                   └──────────────┘   │ DNS
          │ (heartbeat)                      ▲            │ Round
   ┌──────▼───────┐                   ┌──────────────┐   │ Robin
   │    Load      │  Standby          │    Load      │◄──┘
   │  Balancer 2  │  (takes over      │  Balancer 2  │
   └──────────────┘   on failure)     └──────────────┘
```

---

## Global Server Load Balancing (GSLB)

GSLB routes users to the nearest or healthiest datacenter.

```
       User in Asia ──▶ DNS query ──▶ Return Asia DC IP
       User in US   ──▶ DNS query ──▶ Return US DC IP

                   ┌───────────────┐
                   │  GeoDNS /     │
                   │  GSLB Layer   │
                   └──────┬────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  US East DC │  │  EU West DC │  │  Asia DC    │
  └─────────────┘  └─────────────┘  └─────────────┘
```

**Routing strategies:** Geographic proximity, lowest latency (measured), datacenter health, compliance requirements.

---

## Real-World Load Balancers

| Product | Type | Notes |
|---------|------|-------|
| AWS ALB | L7 (software) | Path-based routing, WebSocket, gRPC support |
| AWS NLB | L4 (software) | Ultra-low latency, static IPs, preserves client IP |
| Nginx | L4 + L7 | Open source, extremely popular, doubles as web server |
| HAProxy | L4 + L7 | High performance, excellent observability |
| Envoy | L7 | Service mesh, used in Istio, built for microservices |
| Cloudflare | L7 (global) | Global anycast network, DDoS protection included |

---

## Key Takeaways

1. **Load balancers enable horizontal scaling and eliminate single points of failure**
2. **Round Robin works well for stateless, homogeneous services**
3. **Least Connections is better for variable-duration requests like WebSockets**
4. **L4 is faster; L7 is smarter** — use L7 for HTTP-based services needing content routing
5. **Health checks are not optional** — without them, traffic goes to dead servers
6. **Avoid sticky sessions** — they create coupling; prefer stateless design + external session store
7. **Load balancers need redundancy too** — active-passive or active-active setups
8. **GSLB routes globally** — use GeoDNS or Anycast for multi-region deployments

---

*Previous: [CAP Theorem ←](05-cap-theorem.md) · Next: [Caching Fundamentals →](07-caching-fundamentals.md)*
