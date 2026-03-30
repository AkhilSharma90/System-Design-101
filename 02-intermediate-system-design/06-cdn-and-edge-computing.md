# CDN and Edge Computing

A Content Delivery Network (CDN) is a geographically distributed network of servers that caches and serves content close to your users. It's one of the highest-leverage infrastructure investments you can make — dramatically reducing latency, offloading your origin servers, and improving availability worldwide.

---

## The Problem CDNs Solve

Without a CDN, all users — regardless of location — connect to your origin server.

```
Origin server: US East (Virginia)

Tokyo user ──── 150ms ────▶ Virginia server
Berlin user ──── 90ms ────▶ Virginia server
São Paulo user ─ 120ms ───▶ Virginia server
LA user ────────  70ms ───▶ Virginia server

With CDN:
Tokyo user ──── 15ms ────▶ Tokyo PoP ── cached ──▶ user
Berlin user ─── 10ms ────▶ Frankfurt PoP ─ cached ─▶ user
São Paulo user ─ 20ms ───▶ São Paulo PoP ─ cached ──▶ user
LA user ────────  8ms ───▶ LA PoP ── cached ──────────▶ user
```

---

## How CDNs Work

### Points of Presence (PoPs)

CDNs operate hundreds of **PoPs** (Points of Presence) — data centers distributed worldwide. Each PoP has edge servers that cache your content.

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
2. DNS resolves to Tokyo PoP IP (GeoDNS)
3. Request hits Tokyo edge server
4. Edge server checks local cache:
   a. CACHE HIT: Return cached image instantly (15ms)
   b. CACHE MISS: Fetch from origin (150ms), cache it, return to user

5. Next Tokyo user requesting image.jpg: always cache hit (15ms)
```

---

## Push CDN vs Pull CDN

### Pull CDN (Most Common)

The CDN fetches content from your origin server on demand (on first request = cache miss).

```
You upload content to: origin.yoursite.com
CDN URL is:           cdn.yoursite.com

First request to PoP:  PoP fetches from origin, caches
Subsequent requests:   PoP serves from cache

Pros:  Easy to set up, only caches what's actually requested
Cons:  First request to each PoP always hits origin ("cold start")
```

**Best for:** General websites, media, APIs with unpredictable access patterns.

### Push CDN

You proactively push content to all PoPs before users request it.

```
You upload video → CDN replicates to all PoPs immediately
Users get cache hit from first request everywhere

Pros:  No cold-start latency, good for large content you know will be popular
Cons:  You must manage what content is pushed and invalidated
       Cost: you pay to store content on all PoPs (even if few users in some regions)
```

**Best for:** Large, predictable content — software releases, game updates, large video files.

---

## CDN Caching Strategy

### Cache-Control Headers

Your origin server tells the CDN how to cache responses:

```http
Cache-Control: public, max-age=86400        ← Cache for 24 hours
Cache-Control: public, max-age=31536000     ← Cache for 1 year (static assets)
Cache-Control: no-store                     ← Never cache (user-specific data)
Cache-Control: no-cache                     ← Cache but revalidate every time
Cache-Control: s-maxage=3600                ← Override CDN TTL (but not browser)

ETag: "abc123"                              ← Version fingerprint for cache validation
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
```

### Cache Busting

For content that must be invalidated immediately (after a deploy):

```html
<!-- Old approach: unpredictable cache behavior -->
<script src="/app.js"></script>

<!-- New approach: content-hashed filename -->
<script src="/app.abc123.js"></script>

When app.js changes, new hash → new URL → CDN treats as new file
Old URL cached forever (safe) → new URL fetched fresh
```

### Cache Invalidation

```
Option 1: Wait for TTL to expire (simple, delay up to TTL)
Option 2: Purge by URL (instant, costs API call per URL)
Option 3: Purge by tag/surrogate key (purge all pages using a resource)
Option 4: Purge all (nuclear option — causes origin storm)
```

---

## CDN for Dynamic Content

CDNs aren't just for static files. Modern CDNs cache dynamic content too:

### ESI (Edge Side Includes)
Compose pages at the edge by assembling cached fragments:

```html
<!-- CDN stitches together static header + dynamic personalized content -->
<esi:include src="/header.html" />       ← static, cached 1 year
<esi:include src="/recommendations" />   ← dynamic, short TTL or no cache
<esi:include src="/footer.html" />       ← static, cached 1 year
```

### Vary Header
Cache different versions of the same URL based on request headers:

```http
Vary: Accept-Encoding   ← Cache gzip and non-gzip versions separately
Vary: Accept-Language   ← Cache English and French versions separately
Vary: Cookie           ← DANGER: nearly disables caching (unique cookies = unique entries)
```

---

## Edge Computing

Edge computing moves computation — not just content — to the edge servers, closer to users.

```
Traditional:
User request ──150ms──▶ Origin (compute) ──150ms──▶ Response = 300ms RTT

Edge computing:
User request ──15ms──▶ Edge PoP (compute) ──15ms──▶ Response = 30ms RTT
```

### Edge Functions

Run serverless code at the edge:

| Platform | Product | Runtime |
|----------|---------|---------|
| Cloudflare | Workers | V8 JavaScript isolates |
| AWS | Lambda@Edge | Node.js, Python |
| Vercel | Edge Functions | V8 (Web Crypto, Fetch API) |
| Fastly | Compute@Edge | WebAssembly |

### Common Edge Function Use Cases

```
A/B Testing:
  Edge reads experiment cookie → returns variant A or B HTML
  No round trip to origin → zero added latency

Personalization:
  Edge reads user country from IP → returns localized content

Authentication:
  Edge validates JWT → blocks unauthenticated requests before they reach origin

Image Optimization:
  Edge resizes/converts images on-demand based on Accept header / screen size

Bot Detection:
  Edge checks request signatures → blocks bots at the edge
```

---

## CDN Security

CDNs offer substantial security benefits beyond just performance:

### DDoS Mitigation
```
Attack traffic: 1 Tbps of garbage requests
                        │
                        ▼
          ┌──────────────────────────┐
          │     CDN Network          │
          │  (distributed across     │
          │   100+ PoPs globally)    │
          │  Absorbs and filters     │
          │  attack traffic          │
          └──────────────────────────┘
                        │ (only clean traffic passes)
                        ▼
              Your origin server
              (protected, minimal load)
```

Cloudflare's network can absorb multi-terabit attacks. Your origin never sees them.

### SSL/TLS Termination
CDN handles TLS handshake at the edge — closer to the user:

```
User ←── TLS ──▶ Edge PoP (15ms TLS handshake) ──▶ Origin (unencrypted or re-encrypted)
vs.
User ←──────────── TLS ──────────────────────▶ Origin (150ms TLS handshake)
```

### WAF (Web Application Firewall)
Block SQL injection, XSS, and OWASP Top 10 attacks at the edge before they reach your application code.

---

## CDN Providers Comparison

| Provider | Strength | PoPs | Notable Features |
|----------|---------|------|-----------------|
| Cloudflare | Security, edge compute | 300+ | Workers, DDoS, Zero Trust, R2 storage |
| AWS CloudFront | AWS integration | 400+ | Lambda@Edge, deep AWS ecosystem |
| Akamai | Enterprise, media | 4,000+ | Largest network, streaming |
| Fastly | Developers, real-time purge | 80+ | Compute@Edge, instant purge |
| Vercel Edge | Frontend/Next.js | Global | Zero-config, developer DX |

---

## When You Don't Need a CDN

- Internal tools with no geographic distribution requirement
- APIs serving only dynamic, user-specific content
- Very early stage when latency isn't the bottleneck
- Development environments

---

## Key Takeaways

1. **CDNs reduce latency by serving content from the nearest PoP** — 10-20ms vs 100-200ms for distant users
2. **Pull CDN is simpler** — content cached on first access; push CDN pre-warms all edge nodes
3. **Cache-Control headers control CDN behavior** — set long TTLs for static assets, use cache-busting
4. **Edge functions move computation to the edge** — A/B testing, auth, personalization with near-zero latency
5. **CDNs absorb DDoS attacks** — your origin server is shielded behind the CDN's distributed network
6. **SSL termination at the edge** — cuts TLS handshake latency dramatically for global users
7. **For most web apps, a CDN is one of the highest-ROI infrastructure investments you can make**

---

*Previous: [Rate Limiting ←](05-rate-limiting.md) · Next: [Microservices vs Monolith →](07-microservices-vs-monolith.md)*
