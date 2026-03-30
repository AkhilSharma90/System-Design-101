# Rate Limiting

Rate limiting controls how many requests a client can make to your system in a given time window. It's your first line of defense against abuse, your protection against accidental traffic spikes, and a tool for enforcing fair usage across all your customers.

---

## Why Rate Limiting?

Without rate limiting, a single misbehaving client can:
- **DDoS your system** (intentional or accidental)
- **Exhaust your database connections** with a runaway script
- **Run up your cloud bill** by hammering expensive endpoints
- **Degrade service for all other users** by consuming shared resources

Rate limiting protects everyone.

---

## Token Bucket Algorithm

The most widely used algorithm. Imagine a bucket that holds tokens. Tokens are added at a fixed rate. Each request consumes a token. If the bucket is empty, the request is rejected.

```
                  Rate: 10 tokens/second
                            │
                            ▼
                   ┌────────────────┐
                   │   Token Bucket │
Token added/sec ──▶│   ○ ○ ○ ○ ○   │ Capacity: 20 tokens
                   │   ○ ○ ○ ○ ○   │
                   └────────┬───────┘
                            │ consume 1 token per request
                            ▼
                     Request allowed?
                    ○ Token available → ✅ Allow
                    ○ Bucket empty   → ❌ Reject (429)
```

**Key property:** Allows bursting up to `capacity` requests, then enforces the sustained rate.

```
Capacity=20, Rate=10/sec:
t=0: Bucket full (20 tokens). Client sends 20 requests → all allowed (burst!)
t=1: 10 tokens added. Client sends 15 requests → 10 allowed, 5 rejected
t=2: 10 tokens added. Client sends 5 requests  → all 5 allowed
```

**Pros:** Simple, allows controlled bursting, memory-efficient (just store token count + last refill time)
**Cons:** Doesn't smooth out traffic perfectly within the window

---

## Leaky Bucket Algorithm

Requests enter a bucket (queue). The bucket "leaks" at a constant rate — processing requests at a fixed pace regardless of input rate.

```
Requests ──▶ ┌──────────────┐
(bursty)     │   Bucket     │──── Process 1 req/100ms ────▶
             │   (queue)    │                              Smooth output
             └──────────────┘
                   │ overflow
                   ▼
               ❌ Reject
```

**Pros:** Smooths out bursty traffic, consistent output rate — great for protecting downstream systems
**Cons:** Bursts are queued and delayed, not served fast; queue can build up

**Use case:** Rate-limiting outbound requests (e.g., calling a third-party API with strict limits).

---

## Fixed Window Counter

Count requests in fixed time windows (e.g., per minute).

```
Window: [12:00:00 – 12:00:59]   Limit: 100 requests
  12:00:00: counter = 1
  12:00:30: counter = 85
  12:00:55: counter = 100 → subsequent requests rejected
  12:01:00: NEW WINDOW → counter resets to 0 ✅

Boundary problem:
  12:00:55 – 12:00:59: 100 requests allowed
  12:01:00 – 12:01:05: 100 more requests allowed
  → 200 requests in 10 seconds! Double the limit at window boundaries.
```

**Pros:** Simple to implement and understand
**Cons:** Boundary burst problem — 2x the limit can occur at window transitions

---

## Sliding Window Log

Track the timestamp of every request. On each new request, count how many requests fall within the last window.

```
Request log: [12:00:05, 12:00:23, 12:00:44, 12:00:58, 12:01:02]
New request at 12:01:10, window = 60 seconds:

  Requests in [12:00:10 – 12:01:10]:
  [12:00:23, 12:00:44, 12:00:58, 12:01:02] = 4 requests

  If limit = 5 → allowed!
```

**Pros:** No boundary problem; perfectly accurate
**Cons:** Memory-heavy — must store every request timestamp; for 1M req/min, that's a lot of data

---

## Sliding Window Counter

Hybrid of fixed window + sliding window. Use two counters (current and previous window) with a weighted estimate.

```
Limit: 100/minute

Previous window (12:00): 80 requests
Current window (12:01):  20 requests (so far, at 12:01:30 = 50% through)

Estimated count = (80 × 0.5) + 20 = 60 requests

60 < 100 → allow the request
```

**Pros:** Memory-efficient (only 2 counters), close approximation of sliding window, no boundary spikes
**Cons:** Approximation — slightly off (~0.003% error in practice, from Cloudflare's analysis)

**This is what most production systems use**, including Cloudflare.

---

## Algorithm Comparison

| Algorithm | Memory | Allows Burst | Accuracy | Complexity |
|-----------|--------|-------------|----------|------------|
| Token Bucket | Low (1 state) | Yes | Good | Low |
| Leaky Bucket | Medium (queue) | No (smoothed) | Exact | Low |
| Fixed Window | Low (1 counter) | Yes (boundary) | Good | Lowest |
| Sliding Window Log | High | No | Exact | Medium |
| Sliding Window Counter | Low (2 counters) | No | ~Exact | Low |

---

## Where to Implement Rate Limiting

```
Internet
    │
    ▼
┌──────────────────┐  ← DNS / CDN level (Cloudflare)
│  CDN / WAF       │    Coarse-grained, block obvious abuse
└────────┬─────────┘
         │
    ▼
┌──────────────────┐  ← API Gateway level (Kong, AWS API Gateway)
│  API Gateway     │    Per-route, per-client limits
└────────┬─────────┘
         │
    ▼
┌──────────────────┐  ← Application level
│  App Server      │    Business logic rate limiting
└────────┬─────────┘
         │
    ▼
┌──────────────────┐  ← Database level
│  Database        │    Connection pool limits
└──────────────────┘
```

---

## Distributed Rate Limiting

Single-server rate limiting is easy. Distributed rate limiting is hard — your counters must be shared across all instances.

```
Without shared state:
┌─────────┐  req 1-100 → allowed   ← counter: 100
│ Server 1│
└─────────┘  req 1-100 → allowed   ← counter: 100
┌─────────┐
│ Server 2│
└─────────┘
Total: 200 requests served — double the limit!

With Redis (shared state):
┌─────────┐  req 1-50  → allowed   ┌─────────────────┐
│ Server 1│ ─────────────────────▶ │ Redis Counter   │
└─────────┘  req 51-100 → allowed  │ client:alice=100│
┌─────────┐ ─────────────────────▶ │                 │
│ Server 2│  req 101   → rejected  └─────────────────┘
└─────────┘
Correct! 100 total.
```

### Redis-Based Rate Limiting

```
# Using Redis atomic INCR + EXPIRE:
count = INCR rate_limit:{user_id}:{window}
if count == 1:
    EXPIRE rate_limit:{user_id}:{window} window_seconds
if count > limit:
    reject request
```

For sliding window with sorted sets:
```
# Add timestamp, remove old entries, count remaining:
ZADD rate_limit:{user_id} now now
ZREMRANGEBYSCORE rate_limit:{user_id} 0 (now - window)
count = ZCARD rate_limit:{user_id}
```

---

## Rate Limiting Strategies

| Strategy | Granularity | Use Case |
|----------|------------|---------|
| Per IP | IP address | DDoS protection, anonymous users |
| Per user | User/API key | Authenticated API access |
| Per endpoint | Route + user | Expensive endpoints cost more |
| Per tenant | Customer account | SaaS tier enforcement |
| Global | Whole API | Protect backend capacity |

**Tiered limits (common for APIs):**
```
Free tier:    100 req/day
Pro tier:   10,000 req/day
Enterprise:  Unlimited (contractual SLA)
```

---

## Rate Limit Headers

Always communicate your limits to clients:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 567
X-RateLimit-Reset: 1700001600

HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 30 seconds.",
  "retry_after": 30
}
```

---

## Graceful Degradation

Instead of hard rejecting, some systems queue requests or serve degraded responses:

```
Normal:      Request → Full response (10ms)
Overloaded:  Request → Cached/stale response (1ms) or
             Request → Queue → Process when capacity available
Rate limited: Request → 429 with Retry-After
```

---

## Real-World Examples

| Company | Limits | Strategy |
|---------|--------|---------|
| GitHub API | 5,000 req/hour (authenticated) | Token bucket per user |
| Stripe API | 100 req/sec (live mode) | Per-API-key rate limiting |
| Twitter API | 300 req/15 min per app | Fixed window |
| Cloudflare | Configurable | Sliding window counter |
| AWS API Gateway | 10,000 req/sec (default, adjustable) | Token bucket |

---

## Key Takeaways

1. **Rate limiting protects your system from abuse, runaway clients, and traffic spikes**
2. **Token bucket is the most practical algorithm** — allows bursting, simple to implement
3. **Sliding window counter is the most accurate and efficient** for distributed systems
4. **Use Redis for distributed rate limiting** — atomic operations ensure correctness across instances
5. **Rate limit at multiple layers** — CDN, API gateway, and application levels catch different threats
6. **Always return meaningful 429 responses** — include `Retry-After` so clients back off correctly
7. **Make limits tiered by customer plan** — enforcing SaaS pricing through rate limits is standard

---

*Previous: [Database Replication ←](04-database-replication.md) · Next: [CDN and Edge Computing →](06-cdn-and-edge-computing.md)*
