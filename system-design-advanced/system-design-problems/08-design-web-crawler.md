---
slug: design-web-crawler
title: Design a Web Crawler
readTime: 25 min
orderIndex: 8
premium: false
---

# Design a Web Crawler

Design a distributed web crawler like Googlebot — capable of crawling billions of web pages, detecting duplicates, respecting website rules, and continuously refreshing its index.

---

## Step 1: Requirements

### Functional Requirements
- Crawl the web starting from seed URLs
- Follow links to discover new pages
- Store page content for indexing
- Respect robots.txt rules
- Avoid revisiting the same page too frequently
- Detect and skip duplicate content

### Non-Functional Requirements
- Scale: crawl 1 billion pages in 30 days
- Politeness: don't hammer any single website
- Robustness: handle timeouts, malformed HTML, infinite redirect loops
- Extensibility: easy to add new crawl logic (e.g., extract metadata)

### Out of Scope
- Ranking pages (that's the search engine layer)
- Rendering JavaScript (advanced; most crawlers skip or handle separately)

---

## Step 2: Back-of-Envelope Estimation

```
Target: 1 billion pages in 30 days
= 1,000,000,000 / (30 × 24 × 3600)
= 1,000,000,000 / 2,592,000
≈ 386 pages/sec

Average page size: 100 KB (HTML only, no media)
Storage: 386 pages/sec × 100 KB = 38.6 MB/sec = ~100 TB/month

Network: 386 pages/sec × 100 KB = 38.6 MB/sec ≈ 300 Mbps outbound
```

---

## Step 3: High-Level Architecture

```
                         ┌──────────────────┐
                         │   URL Frontier   │  ← Priority queue of URLs to crawl
                         │  (Scheduler)     │
                         └────────┬─────────┘
                                  │ next URL to crawl
                         ┌────────▼─────────┐
                    ┌────│   DNS Resolver   │  ← Cached DNS lookups
                    │    └────────┬─────────┘
                    │             │
                    │    ┌────────▼─────────┐
                    │    │    Fetcher       │  ← HTTP client, downloads pages
                    │    │  (Workers pool)  │
                    │    └────────┬─────────┘
                    │             │ raw HTML
                    │    ┌────────▼─────────┐
                    │    │     Parser       │  ← Extract links, metadata
                    │    └──────┬──────┬────┘
                    │           │      │
                    │    New   URLs   Content
                    │           │      │
                    │    ┌──────▼──┐  ┌▼──────────────┐
                    │    │  URL    │  │ Content Store  │
                    │    │  Seen?  │  │  (S3 / HDFS)  │
                    │    └──┬──────┘  └───────────────┘
                    │  No ──┘ Yes → discard
                    │       │
                    └───────┘ (add to URL Frontier)

Supporting components:
  robots.txt cache → respect crawl rules
  URL deduplication store (Bloom filter or DB)
  Content deduplication (SimHash)
```

---

> **Pause and think:** If your crawler fetches 100 pages per second from the same website, what happens? How do you crawl fast globally while being polite to each individual website?

## Step 4: URL Frontier (The Core)

The URL Frontier manages the queue of URLs to be crawled. It must solve two problems simultaneously:

1. **Priority:** Crawl important/fresh pages more often
2. **Politeness:** Don't overwhelm any single host

### Two-Queue Architecture

```
                 ┌─────────────────────────────┐
                 │       URL Frontier           │
                 │                             │
  Incoming ──▶  │  Priority Queues:           │
  new URLs       │    High  (authority sites)  │
                 │    Medium (average pages)   │
                 │    Low   (rarely updated)   │
                 │           │                 │
                 │           ▼                 │
                 │  Politeness Queues:         │
                 │    Queue per hostname:      │
                 │    google.com   [url1, url2]│
                 │    amazon.com   [url3, url4]│
                 │    algoroq.io  [url5]      │
                 │           │                 │
                 │  Crawler Worker             │
                 │  picks from politeness queue│
                 │  when that host is ready    │
                 │  (back-off timer elapsed)   │
                 └─────────────────────────────┘
```

**Priority queue** selects *what* to crawl next.
**Politeness queue** controls *when* to crawl a specific host.

---

## Step 5: URL Deduplication

Before adding a URL to the frontier, check if we've seen it before:

### Option A: Database (Exact)

```sql
CREATE TABLE crawled_urls (
  url_hash VARCHAR(64) PRIMARY KEY,  -- SHA256 of normalized URL
  last_crawled TIMESTAMP
);

Check: SELECT 1 FROM crawled_urls WHERE url_hash = SHA256(normalized_url)
Insert: INSERT INTO crawled_urls (url_hash, last_crawled) VALUES (...)
```

**Problem:** At 1 billion URLs, this table is huge. Every URL check hits the DB.

### Option B: Bloom Filter (Approximate)

A Bloom filter is a probabilistic data structure that answers "definitely not seen" or "probably seen." False positives are possible (says seen when it wasn't) but false negatives are not (never says not seen when it was).

```
Bloom filter for 1 billion URLs:
  - Size: ~1.2 GB in memory (2.4 bytes per URL for 1% false positive rate)
  - Operation: O(k) time — compute k hash functions, check k bits
  - No false negatives: if it's in the filter, we'll find it

vs Database:
  - Size: ~100 GB+ (storing actual URL hashes)
  - Operation: O(1) with index, but disk I/O

Production: Bloom filter for fast first-check, DB for persistent record
  Step 1: Check Bloom filter → "probably seen" → skip; "definitely not" → proceed
  Step 2: Check DB for confirmation before inserting
```

---

## Step 6: URL Normalization

The same page can have many URLs. Normalize before deduplication:

```
Raw URLs → Normalized:

https://Example.COM/Page?b=2&a=1#fragment
→ https://example.com/page?a=1&b=2         (lowercase, sort params, strip fragment)

https://example.com/about/../contact
→ https://example.com/contact              (resolve relative paths)

http://example.com:80/page
→ http://example.com/page                  (remove default port)

https://example.com/page
https://example.com/page/                  (trailing slash: canonicalize to one form)
→ https://example.com/page
```

---

## Step 7: Content Deduplication

Different URLs may serve identical (or near-identical) content:

```
https://example.com/article
https://example.com/article?ref=twitter   ← same content, different URL
https://example.com/article?utm_source=email ← same again
```

### Exact Deduplication (SimHash)

```
Compute fingerprint of page content:
  fingerprint = MD5(page_content)

Before storing:
  If fingerprint seen before → skip (exact duplicate)
  Else → store
```

### Near-Duplicate Detection (SimHash)

SimHash produces a fingerprint where similar documents produce similar hashes (unlike MD5, where tiny changes produce completely different hashes).

```
Document A: "The quick brown fox"
Document B: "The quick brown dog"   ← nearly identical

MD5(A) = 9a... ≠ MD5(B) = 4f...    (completely different)

SimHash(A) = 10110110
SimHash(B) = 10110100              ← only 1 bit different!

Hamming distance = 1 → near-duplicates
```

Google uses SimHash to deduplicate web pages at scale.

---

## Step 8: robots.txt

Every well-behaved crawler respects `robots.txt`:

```
# https://example.com/robots.txt
User-agent: *
Disallow: /admin/
Disallow: /private/
Crawl-delay: 10           ← wait 10 seconds between requests

User-agent: Googlebot
Allow: /                  ← Googlebot can crawl everything
```

**Crawler behavior:**
1. Before crawling any URL on `example.com`, fetch `example.com/robots.txt`
2. Cache the robots.txt content for the domain (24-hour TTL)
3. Check every URL against the rules before fetching
4. Respect `Crawl-delay` directive

```
Robots.txt cache (Redis):
  SET robots:{hostname} {parsed_rules} EX 86400
  GET robots:{hostname}  → rules or nil (need to fetch)
```

---

> **Pause and think:** A website generates calendar pages for every date from year 1 to year 9999 — all with unique URLs. How does your crawler avoid getting stuck crawling millions of near-identical pages?

## Step 9: Spider Traps

Some websites generate infinite unique URLs to trap crawlers:

```
/calendar/2024/01/01/
/calendar/2024/01/02/
/calendar/2024/01/03/
...
/calendar/9999/12/31/   ← infinite unique URLs, same content

/products?sort=price&color=red&size=M&page=1
/products?sort=name&color=red&size=M&page=1
... (combinatorial explosion of query params)
```

**Detection and mitigation:**

```
URL depth limit: Don't crawl URLs more than N levels deep
  https://example.com/a/b/c/d/e/f/... → stop at depth 10

URL pattern detection:
  If 1000+ URLs from same domain with same template → likely a trap
  Example: /items/{id}/ where id is sequential → limit crawl

Query parameter explosion:
  Only crawl first N values of each parameter
  Ignore order-only query params (sort, order)

Max URLs per domain:
  Hard cap: don't crawl more than 1M URLs per domain
```

---

## Step 10: Distributed Crawler Architecture

At scale, you need many machines:

```
Crawler Coordinator
  └── URL Frontier (Kafka + Redis)
          │
     ┌────┼────┐
     ▼    ▼    ▼
  Fetcher Fetcher Fetcher  (N machines, each with pool of HTTP workers)
  Machine Machine Machine
     │    │    │
     └────┼────┘
          │
   Parser Workers (separate pool, CPU-intensive)
          │
     ┌────┼────┐
     ▼    ▼    ▼
  Content Dedup URL Extract URL Frontier update
  Store         (new URLs)   (via Kafka)

Partitioning:
  Assign URL domain ranges to specific Fetcher machines
  → Ensures politeness: one machine per domain, one domain per machine
  → machine-1 handles a-d.* domains
  → machine-2 handles e-h.* domains
  → etc.
```

---

## Step 11: Incremental Crawling

The web changes. Pages must be re-crawled:

```
Recrawl scheduling:
  High-priority pages (news sites, finance): every 15 minutes
  Medium-priority pages (blogs, e-commerce): every 1-7 days
  Low-priority pages (static docs, archives): every 30 days

Detecting changes:
  1. ETag/Last-Modified headers:
     If-None-Match: {etag} → server returns 304 Not Modified if unchanged
     → Don't re-store content, just update crawl timestamp

  2. Content hash comparison:
     Store fingerprint of last crawl
     If new crawl fingerprint == stored → skip processing
```

---

## Key Takeaways

1. **The URL Frontier is the crawler's engine** — priority + politeness queues control what and when to crawl
2. **Bloom filters enable fast URL deduplication at scale** — 1 billion URLs in ~1.2 GB vs 100 GB in a DB
3. **URL normalization is essential** — the same page has many URL forms; canonicalize before deduplication
4. **SimHash detects near-duplicate content** — prevents storing and indexing the same content repeatedly
5. **robots.txt compliance is mandatory** — cache per domain, check before every fetch
6. **Spider traps require URL depth limits and pattern detection** — otherwise the crawler loops forever
7. **Partition domains across crawlers** — ensures politeness and avoids coordination overhead

---


---


*Continue your learning at [algoroq.io](https://algoroq.io) with interactive lessons, progress tracking, and structured learning paths.*
