---
slug: rate-limiting
title: Rate Limiting — Algorithms and Distributed Implementation
readTime: 18 min
orderIndex: 1
premium: false
---

# Rate Limiting

Rate limiting controls how many requests a client can make to your system in a given time window. It's your first line of defense against abuse, your protection against accidental traffic spikes, and a tool for enforcing fair usage across all your customers.

---

## Why Rate Limiting?

Without rate limiting, a single misbehaving client can:
- **DDoS your system** — intentional or from a runaway script
- **Exhaust database connections** — a tight loop with no backoff
- **Run up your cloud bill** — hammering expensive endpoints
- **Degrade service for everyone** — consuming shared resources

### Pause and think

If GitHub didn't rate limit its API, what would happen if one developer ran a script that made 10,000 repo requests per second?

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
                    Token available?  → ✅ Allow
                    Bucket empty?    → ❌ Reject (429)
```

**Key property:** Allows bursting up to `capacity` requests, then enforces sustained rate.

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

Requests enter a queue (the "bucket"). The bucket "leaks" at a constant rate — processing at a fixed pace regardless of input rate.

```
Requests ──▶ ┌──────────────┐
(bursty)     │   Bucket     │──── Process 1 req/100ms ────▶ Smooth output
             │   (queue)    │
             └──────────────┘
                   │ overflow
                   ▼
               ❌ Reject
```

**Pros:** Smooths bursty traffic, consistent output rate — protects downstream systems

**Cons:** Bursts are queued and delayed, not served fast; queue can build up

**Use case:** Rate-limiting outbound requests to a third-party API with strict limits.

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
  12:00:55 – 12:00:59: 100 requests (last 5 sec of window)
  12:01:00 – 12:01:05: 100 more   (first 5 sec of new window)
  → 200 requests in 10 seconds!  2× the limit
```

**Pros:** Simplest to implement

**Cons:** Boundary burst — 2× the allowed rate is possible at window transitions

---

## Sliding Window Log

Track the exact timestamp of every request. On each new request, count how many fall within the last window.

```
Request log: [12:00:05, 12:00:23, 12:00:44, 12:00:58, 12:01:02]
New request at 12:01:10, window = 60 seconds:

  Count requests in [12:00:10 – 12:01:10]:
  [12:00:23, 12:00:44, 12:00:58, 12:01:02] = 4 requests

  Limit = 5 → allow!
```

**Pros:** No boundary problem; perfectly accurate

**Cons:** Memory-heavy — must store every timestamp; for 1M req/min, that's a lot of data

---

## Sliding Window Counter

Hybrid of fixed window + sliding window. Use two counters (current and previous window) with a weighted estimate.

```
Limit: 100/minute

Previous window (12:00): 80 requests
Current window (12:01):  20 requests (at 12:01:30 = 50% through)

Estimated count = (80 × 0.5) + 20 = 60 requests

60 < 100 → allow the request
```

**Pros:** Memory-efficient (only 2 counters), approximates sliding window well, no boundary spikes

**Cons:** Slight approximation (~0.003% error — from Cloudflare's analysis)

**This is what most production systems use**, including Cloudflare.

---

## Algorithm Comparison

| Algorithm | Memory | Allows Burst | Accuracy | Complexity |
|-----------|--------|-------------|----------|------------|
| Token Bucket | Low | Yes (controlled) | Good | Low |
| Leaky Bucket | Medium | No (smoothed) | Exact | Low |
| Fixed Window | Low | Yes (at boundary) | Good | Lowest |
| Sliding Window Log | High | No | Exact | Medium |
| Sliding Window Counter | Low | No | ~Exact | Low |

---

## Where to Implement Rate Limiting

```
Internet
    │
    ▼
┌──────────────────┐  ← DNS / CDN level (Cloudflare)
│  CDN / WAF       │    Coarse-grained, blocks obvious abuse early
└────────┬─────────┘
         │
         ▼
┌──────────────────┐  ← API Gateway level (Kong, AWS API Gateway)
│  API Gateway     │    Per-route, per-client limits
└────────┬─────────┘
         │
         ▼
┌──────────────────┐  ← Application level
│  App Server      │    Business logic and fine-grained limits
└────────┬─────────┘
         │
         ▼
┌──────────────────┐  ← Database level
│  Database        │    Connection pool limits
└──────────────────┘
```

---

## Distributed Rate Limiting with Redis

Single-server rate limiting is easy. Distributed rate limiting requires shared state.

```
Without shared state:
  Server 1: client sends req 1-100 → allowed  (counter: 100)
  Server 2: client sends req 1-100 → allowed  (counter: 100)
  Total: 200 requests served — double the limit!

With Redis (shared state):
  Server 1 + Server 2 both increment the same Redis counter
  After 100 total requests: all subsequent requests rejected ✓
```

### Redis Fixed Window

```python
# Atomic INCR + EXPIRE
count = REDIS.INCR(f"rate_limit:{user_id}:{window_bucket}")
if count == 1:
    REDIS.EXPIRE(f"rate_limit:{user_id}:{window_bucket}", window_seconds)
if count > limit:
    return 429  # Too Many Requests
```

### Redis Sliding Window (Sorted Set)

```python
now = current_timestamp_ms()
window_start = now - window_ms

# Add current request, remove old entries, count remaining
REDIS.ZADD(key, {now: now})
REDIS.ZREMRANGEBYSCORE(key, 0, window_start)
count = REDIS.ZCARD(key)

if count > limit:
    return 429
```

---

## Rate Limiting Strategies

| Strategy | Granularity | Use Case |
|----------|------------|---------|
| Per IP | IP address | DDoS protection, anonymous users |
| Per user | User/API key | Authenticated API access |
| Per endpoint | Route + user | Expensive endpoints cost more quota |
| Per tenant | Customer account | SaaS tier enforcement |
| Global | Whole API | Protect total backend capacity |

**Tiered limits (standard for SaaS APIs):**
```
Free tier:      100 req/day
Pro tier:    10,000 req/day
Enterprise:  Unlimited (contractual SLA)
```

---

## Rate Limit Response Headers

Always communicate limits to clients so they can self-throttle:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 567
X-RateLimit-Reset: 1700001600

---

HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Retry after 30 seconds.",
  "retry_after": 30
}
```

---

## Real-World Rate Limits

| Company | Limit | Algorithm |
|---------|-------|-----------|
| GitHub API | 5,000 req/hour (authenticated) | Token bucket per user |
| Stripe API | 100 req/sec (live mode) | Per-API-key |
| Twitter API | 300 req/15 min per app | Fixed window |
| Cloudflare | Configurable | Sliding window counter |
| AWS API Gateway | 10,000 req/sec default | Token bucket |

---

## Common Mistakes

| Mistake | Why it's wrong | Correct approach |
|---------|---------------|-----------------|
| Rate limiting only at the application level | Requests still reach your servers and consume resources | Implement at CDN/API gateway level too — block abuse before it hits your app |
| Using fixed-window counters | 2x burst possible at window boundaries | Use sliding window counter or token bucket for smoother limiting |
| Not returning Retry-After headers | Clients don't know when to retry, leading to thundering herd on window reset | Always include Retry-After in 429 responses |
| Same limits for all endpoints | Expensive endpoints get hammered at the same rate as cheap ones | Set per-endpoint limits — /search costs more than /health |
| Not whitelisting internal services | Your own services get rate limited during high traffic | Separate rate limit policies for internal vs external traffic |

---

## Key Takeaways

1. **Rate limiting protects systems from abuse, runaway clients, and traffic spikes** — essential for any public API
2. **Token bucket is the most practical** — allows controlled bursting, memory-efficient
3. **Sliding window counter is the best distributed choice** — accurate, low memory, no boundary spikes
4. **Use Redis for distributed rate limiting** — atomic operations ensure correctness across all instances
5. **Implement at multiple layers** — CDN, API gateway, and application each catch different threats
6. **Return meaningful 429 responses with `Retry-After`** — well-behaved clients will back off automatically
7. **Tier limits by customer plan** — rate limiting is how SaaS pricing is enforced in practice
