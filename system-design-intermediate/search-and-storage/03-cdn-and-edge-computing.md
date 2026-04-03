---
slug: cdn-and-edge-computing
title: CDN and Edge Computing — Deep Dive
readTime: 20 min
orderIndex: 3
premium: false
---

# CDN and Edge Computing

A Content Delivery Network (CDN) is a geographically distributed network of servers that caches and serves content close to your users. It's one of the highest-leverage infrastructure investments you can make — dramatically reducing latency, offloading your origin servers, and improving global availability.

---

## The Problem CDNs Solve

Without a CDN, all users connect to your origin server regardless of location:

```
Origin server: US East (Virginia)

Tokyo user    ──── 150ms ────▶ Virginia server
Berlin user   ────  90ms ────▶ Virginia server
São Paulo user ─── 120ms ────▶ Virginia server
LA user       ────  70ms ────▶ Virginia server

With CDN:
Tokyo user    ────  15ms ────▶ Tokyo PoP      ← 10× faster
Berlin user   ────  10ms ────▶ Frankfurt PoP  ← 9× faster
São Paulo user ────  20ms ───▶ São Paulo PoP  ← 6× faster
LA user       ────   8ms ────▶ LA PoP         ← 8× faster
```

> **Pause and think:** If your website serves users globally but all your servers are in Virginia, what happens to the experience for a user in Tokyo? What if a popular image is requested 1 million times per day?

---

## How CDNs Work

### Points of Presence (PoPs)

CDNs operate hundreds of **PoPs** (Points of Presence) — data centers distributed globally. Each PoP has edge servers that cache your content.

```
                    ┌─────────────────────┐
                    │    Origin Server    │
                    │   (your datacenter) │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  PoP: London │    │  PoP: Tokyo  │    │  PoP: São    │
  │  Edge servers│    │  Edge servers│    │  Paulo       │
  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
         │                   │                   │
    UK users            Japan users        Brazil users
```

### Request Flow (Pull CDN)

```
1. User in Tokyo requests image.jpg
2. DNS resolves to Tokyo PoP IP (via GeoDNS)
3. Request hits Tokyo edge server
4. Edge checks local cache:
   a. CACHE HIT  → Return cached image (15ms total)
   b. CACHE MISS → Fetch from origin (150ms), cache it, return

5. Next Tokyo user requesting image.jpg: always cache hit (15ms)
```

---

## Push CDN vs Pull CDN

### Pull CDN (Most Common)

The CDN fetches from your origin on first request (cache miss).

```
You upload to:    origin.yoursite.com
Users request:    cdn.yoursite.com

First request to a PoP:   miss → fetch from origin → cache
All subsequent requests:  hit  → serve from edge
```

**Best for:** General websites, media, APIs with unpredictable access patterns.

### Push CDN

You proactively push content to all PoPs before users request it.

```
You upload video → CDN replicates to all PoPs immediately
Users get cache hit from first request, everywhere

Pros: No cold-start latency, great for known-popular large content
Cons: Must manage what's pushed and when to invalidate
      Pay to store on all PoPs even in low-traffic regions
```

**Best for:** Large predictable content — software releases, game updates, popular video files.

---

## CDN Caching Strategy

### Cache-Control Headers

Your origin server tells the CDN how to cache:

```http
Cache-Control: public, max-age=86400        ← Cache for 24 hours
Cache-Control: public, max-age=31536000     ← Cache for 1 year (static assets)
Cache-Control: no-store                     ← Never cache (user-specific data)
Cache-Control: no-cache                     ← Cache but revalidate every time
Cache-Control: s-maxage=3600                ← Override CDN TTL (not browser)

ETag: "abc123"                              ← Fingerprint for cache validation
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
```

### Cache Busting

For content that must be invalidated immediately after a deploy:

```html
<!-- BAD: unpredictable cache behavior on update -->
<script src="/app.js"></script>

<!-- GOOD: content-hashed filename, cache forever safely -->
<script src="/app.abc123.js"></script>
```

When `app.js` changes, it gets a new hash → new URL → CDN treats as a new file. Old URLs are cached safely forever.

### Cache Invalidation Options

```
Option 1: Wait for TTL to expire     — simple, but delay up to TTL
Option 2: Purge by URL               — instant, API call per URL
Option 3: Purge by surrogate key     — purge all pages using a resource
Option 4: Purge all                  — nuclear option → causes origin storm
```

---

## CDN for Dynamic Content

CDNs aren't just for static files. Modern CDNs cache dynamic content too:

### ESI (Edge Side Includes)

Compose pages at the edge from cached fragments:

```html
<esi:include src="/header.html" />       ← static, cached 1 year
<esi:include src="/recommendations" />   ← dynamic, short TTL
<esi:include src="/footer.html" />       ← static, cached 1 year
```

### Vary Header

Cache different versions of the same URL based on request attributes:

```http
Vary: Accept-Encoding   ← Cache gzip and uncompressed separately
Vary: Accept-Language   ← Cache English and French separately
Vary: Cookie            ← DANGER: nearly disables caching
```

---

## Edge Computing

Edge computing moves computation — not just content — to edge servers, close to users:

```
Traditional:
User ──150ms──▶ Origin (compute here) ──150ms──▶ Response  =  300ms round trip

Edge computing:
User ──15ms──▶ Edge PoP (compute here) ──15ms──▶ Response  =  30ms round trip
```

### Edge Function Platforms

| Platform | Product | Runtime |
|----------|---------|---------|
| Cloudflare | Workers | V8 JavaScript isolates |
| AWS | Lambda@Edge | Node.js, Python |
| Vercel | Edge Functions | Web Crypto, Fetch API |
| Fastly | Compute@Edge | WebAssembly |

### Common Edge Function Use Cases

```
A/B Testing:
  Edge reads experiment cookie → returns variant A or B
  No round trip to origin → zero added latency

Personalization:
  Edge reads user country from IP → returns localized content

Authentication:
  Edge validates JWT → blocks unauthenticated requests before origin

Image Optimization:
  Edge resizes/converts images based on Accept header / device

Bot Detection:
  Edge checks request signatures → blocks at network edge
```

---

## CDN Security

### DDoS Mitigation

```
Attack: 1 Tbps of garbage requests
               │
               ▼
  ┌──────────────────────────┐
  │     CDN Network          │
  │  (100+ PoPs globally)    │
  │  Absorbs and filters     │
  │  attack traffic          │
  └──────────────────────────┘
               │  (only clean traffic passes)
               ▼
     Your origin server (shielded)
```

Cloudflare's network can absorb multi-terabit attacks. Your origin never sees them.

### SSL/TLS Termination at the Edge

```
Without CDN:
User ──────────── TLS handshake ──────────────▶ Origin (150ms handshake)

With CDN:
User ── TLS ──▶ Edge PoP (15ms handshake) ──▶ Origin (fast internal path)
```

TLS termination at the edge cuts handshake latency dramatically for global users.

### WAF (Web Application Firewall)

Block SQL injection, XSS, and OWASP Top 10 attacks at the CDN edge, before requests ever reach your application code.

---

## CDN Providers Comparison

| Provider | Strength | PoPs | Notable Features |
|----------|---------|------|-----------------|
| Cloudflare | Security, edge compute | 300+ | Workers, DDoS, Zero Trust |
| AWS CloudFront | AWS integration | 400+ | Lambda@Edge, deep AWS ecosystem |
| Akamai | Enterprise, media streaming | 4,000+ | Largest network globally |
| Fastly | Developer-friendly, real-time purge | 80+ | Compute@Edge, instant purge |
| Vercel Edge | Frontend / Next.js | Global | Zero-config, developer DX |

---

## When You Don't Need a CDN

- Internal tools with no geographic distribution requirements
- APIs serving only dynamic, user-specific content (no cacheability)
- Very early stage when latency is not the bottleneck
- Development environments

---

## Key Takeaways

1. **CDNs reduce latency by 5–15× for global users** — serving from the nearest PoP instead of a single origin
2. **Pull CDN is simpler** — cached on first access; push CDN pre-warms all edges for known-popular content
3. **Cache-Control headers drive CDN behavior** — long TTLs for static assets, content-hashed URLs for safe forever-caching
4. **Edge functions move compute to the edge** — A/B testing, auth, and personalization with near-zero added latency
5. **CDNs absorb DDoS attacks** — your origin server is shielded behind a distributed global network
6. **TLS termination at the edge** — cuts handshake latency dramatically for geographically distant users
7. **For most web apps, a CDN is one of the highest-ROI infrastructure investments available**
