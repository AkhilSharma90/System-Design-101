# Design a URL Shortener

Design a system like TinyURL or Bit.ly that converts long URLs into short aliases and redirects users when the short URL is visited.

---

## Step 1: Requirements

### Functional Requirements
- Given a long URL, generate a short URL (e.g., `algo.rq/abc123`)
- Redirect users from the short URL to the original long URL
- Optional: custom aliases (user-chosen short codes)
- Optional: URL expiration after a configurable time period
- Optional: analytics (click counts, referrers, geographies)

### Non-Functional Requirements
- High availability: redirects must work 99.99% of the time
- Low latency: redirects should complete in < 100ms
- Not predictable: short codes should not be guessable sequentially
- Scale: 100 million new URLs created per month

### Out of Scope
- User accounts / login (keep it simple)
- URL preview / safety scanning

---

## Step 2: Back-of-Envelope Estimation

```
Write (URL creation):
  100M URLs/month = 100,000,000 / (30 × 24 × 3600) ≈ 40 writes/sec

Read (URL redirection):
  Assuming 10:1 read-to-write ratio
  400 reads/sec average; 2,000 reads/sec peak

Storage:
  Each URL record: ~500 bytes (short URL + long URL + metadata)
  100M records/month × 500 bytes = 50 GB/month
  10 years → 6 TB total

Short URL length:
  Base62 characters: [a-z A-Z 0-9] = 62 symbols
  6 characters: 62^6 = 56.8 billion combinations (plenty for decades)
  7 characters: 62^7 = 3.5 trillion combinations
```

---

## Step 3: API Design

```
Create short URL:
  POST /api/urls
  Body: { "long_url": "https://...", "custom_alias": "optional", "expires_at": "optional" }
  Response: { "short_url": "https://algo.rq/abc123", "long_url": "...", "expires_at": "..." }

Redirect:
  GET /{short_code}
  Response: 301/302 Redirect to long_url

Get URL info:
  GET /api/urls/{short_code}
  Response: { "short_url": "...", "long_url": "...", "click_count": 42, "created_at": "..." }

Delete URL:
  DELETE /api/urls/{short_code}
```

**301 vs 302:**
- **301 Permanent Redirect:** Browser caches the redirect. Subsequent visits go directly to the long URL — no load on our servers. But we lose analytics (clicks not counted).
- **302 Temporary Redirect:** Browser always hits our servers first. We get all analytics. Slightly higher latency.

For analytics: use **302**. For pure performance: use **301**.

---

## Step 4: Data Model

```sql
-- URL mapping
CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(10) UNIQUE NOT NULL,   -- "abc123"
  long_url    TEXT NOT NULL,                  -- original URL
  user_id     BIGINT,                         -- optional creator
  created_at  TIMESTAMP DEFAULT NOW(),
  expires_at  TIMESTAMP,                      -- NULL = no expiration
  click_count BIGINT DEFAULT 0
);

-- Index for fast lookups (the hot path)
CREATE INDEX idx_urls_short_code ON urls(short_code);

-- Click analytics (high write volume → separate table or event stream)
CREATE TABLE clicks (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(10) NOT NULL,
  clicked_at  TIMESTAMP DEFAULT NOW(),
  ip_address  INET,
  referrer    TEXT,
  user_agent  TEXT
);
```

---

## Step 5: High-Level Architecture

```
                                Users
                                  │
                    ┌─────────────┼──────────────┐
                    ▼             ▼              ▼
             Short URL         Create URL      Analytics
             Redirect          Request         Dashboard
             GET /abc123       POST /urls      GET /stats

                                  │
                         ┌────────▼────────┐
                         │  Load Balancer  │
                         └────────┬────────┘
                                  │
               ┌──────────────────┼──────────────────┐
               ▼                  ▼                  ▼
        ┌──────────┐       ┌──────────┐       ┌──────────┐
        │App Server│       │App Server│       │App Server│
        │   (1)    │       │   (2)    │       │   (3)    │
        └────┬─────┘       └────┬─────┘       └────┬─────┘
             │                  │                  │
    ┌────────▼──────────────────▼──────────────────▼────────┐
    │                   Cache (Redis)                        │
    │              short_code → long_url                     │
    │              TTL: 24 hours for hot URLs                │
    └────────────────────────────┬───────────────────────────┘
                                 │ miss
                    ┌────────────▼────────────┐
                    │     Database (MySQL/     │
                    │       PostgreSQL)        │
                    │     Primary + Replicas   │
                    └─────────────────────────┘
```

---

## Step 6: Deep Dives

### Deep Dive 1: Key Generation

How do we generate unique 6-character short codes?

**Option A: MD5 of long URL**
```
short_code = MD5(long_url)[:6] in Base62

Problem: Collisions (different URLs → same first 6 chars)
Problem: Same URL always gets same short code (no duplicates allowed for users)
Problem: Predictable — hash of URL is deterministic
```

**Option B: Auto-increment ID + Base62 Encode**
```
id = next auto-increment (1, 2, 3, ...)
short_code = base62_encode(id)

id=1  → "1"
id=100 → "1C"
id=57,000,000 → "abc123"

Problem: Sequential and predictable (users can iterate through all URLs)
Problem: Exposes business metrics (can estimate total URL count)
```

**Option C: Key Generation Service (KGS) — Best Approach**

Pre-generate random keys, store unused ones in a database. App servers fetch and use keys.

```
┌─────────────────┐       ┌─────────────────┐       ┌────────────────┐
│   KGS (Key      │       │    Key DB       │       │    App         │
│  Generation     │──────▶│                 │       │   Servers      │
│   Service)      │       │  unused_keys:   │◀──────│                │
│                 │       │  [abc123,       │ fetch │  Fetch 1000    │
│ Generates &     │       │   xyz789,       │ keys  │  keys at once  │
│ stores random   │       │   ...]          │       │  Store in      │
│ Base62 strings  │       │                 │       │  memory        │
└─────────────────┘       │  used_keys:     │       └────────────────┘
                          │  [...]          │
                          └─────────────────┘
```

**Benefits:**
- No hashing, no collisions, no predictability
- App servers batch-fetch keys → very fast
- KGS can be replicated for HA

**Issue:** Two app servers might fetch the same key.
**Solution:** KGS marks keys as "in progress" atomically when handing them out. Use database transactions or a Redis-based distributed lock.

---

### Deep Dive 2: Redirection Flow

```
User visits: https://algo.rq/abc123

1. DNS resolves algo.rq → Load Balancer IP
2. Load Balancer → App Server
3. App Server:
   a. Check Redis: GET "url:abc123"
      → HIT: return 302 Redirect to long_url immediately  (1ms)
   b. MISS: Query database: SELECT long_url FROM urls WHERE short_code='abc123'
      → Found: store in Redis with TTL, return 302 Redirect (20ms)
      → Not Found: return 404
      → Expired: return 410 Gone

4. Browser follows redirect → original website

Total time: 1ms (cache hit) or 20ms (cache miss)
```

---

### Deep Dive 3: Analytics

Click tracking without slowing down the redirect:

```
GET /abc123
   │
   ├──▶ 302 Redirect (immediate, don't wait)
   │
   └──▶ Async: publish to Kafka { short_code, timestamp, ip, user_agent, referrer }
                │
                ▼
         Analytics Workers
         (consume from Kafka, write to ClickHouse/analytics DB)

User gets redirect in <5ms.
Analytics written asynchronously — no impact on redirect latency.
```

---

### Deep Dive 4: Expiration and Cleanup

```
Expiration check on read:
  SELECT long_url, expires_at FROM urls WHERE short_code='abc123';
  If expires_at IS NOT NULL AND expires_at < NOW(): return 410 Gone

Proactive cleanup (background job):
  Every night: DELETE FROM urls WHERE expires_at < NOW() - INTERVAL '7 days';
  (Keep 7-day buffer for any lingering cache entries to expire)
```

---

## Step 7: Scaling

### Database Scaling
- **Read replicas:** Redirect traffic is ~99% reads. Route to read replicas.
- **Caching:** Hot URLs (viral links) cached in Redis with 24h TTL. Cache hit rate should be >95%.
- **Sharding by short_code:** If needed, hash the short_code to a shard.

### Cache Strategy
```
Hot URL (visited 1M+ times/day):  Cache hit rate ~100% after first visit
Warm URL (visited 100+ times/day): Usually in cache
Cold URL (visited rarely):        May not be cached
```

80% of traffic likely goes to 20% of URLs (power law). Cache the hot 20% → serve 80% of traffic from cache.

### Rate Limiting
- Limit URL creation to 100/hour per IP for anonymous users
- Prevent DDoS on the redirect endpoint (per-IP rate limiting at the load balancer)

---

## Step 8: Trade-offs and Alternatives

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Key generation | KGS pre-generated keys | Hash-based | No collisions, not predictable |
| Redirect type | 302 | 301 | Analytics visibility |
| Analytics | Async Kafka | Sync DB write | No impact on redirect latency |
| Cache | Redis | Application memory | Shared across all app servers |

---

## Key Takeaways

1. **The redirect path is the hottest path** — optimize it relentlessly (Redis cache + CDN)
2. **KGS pre-generation avoids collisions and sequential IDs** — the right key generation approach
3. **302 enables analytics; 301 improves performance** — choose based on requirements
4. **Separate analytics writes from the redirect path** — async Kafka consumers for click tracking
5. **Expiration needs both lazy deletion (on read) and active cleanup** — two strategies complement each other

---

*Previous: [Event-Driven Architecture ←](02-event-driven-architecture.md) · Next: [Design a Chat System →](04-design-chat-system.md)*
