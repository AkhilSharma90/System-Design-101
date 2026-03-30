---
slug: blob-object-storage
title: Blob Storage and Object Storage (S3)
readTime: 15 min
orderIndex: 2
premium: false
---

# Blob Storage and Object Storage

When you need to store images, videos, backups, logs, or any unstructured binary data at scale, blob/object storage is the answer. It's one of the most cost-effective and durable storage systems ever built — Amazon S3 alone stores trillions of objects.

---

## What is Object Storage?

Object storage treats data as discrete objects, each with a key, value (the binary data), and metadata. Unlike file systems (hierarchical directories) or block storage (raw volumes for databases), object storage is flat and accessed via HTTP APIs.

```
┌────────────────────────────────────────────────────────┐
│                 Object Storage Model                   │
│                                                        │
│  Bucket: "algoroq-assets"                              │
│  ┌──────────────────────────┬────────────┬──────────┐  │
│  │ Key (path)               │ Value      │ Metadata │  │
│  ├──────────────────────────┼────────────┼──────────┤  │
│  │ images/hero.jpg          │ [binary]   │ size, ct │  │
│  │ videos/lesson-01.mp4     │ [binary]   │ size, ct │  │
│  │ users/42/avatar.png      │ [binary]   │ size, ct │  │
│  │ exports/2024-01-report.csv│ [binary]  │ size, ct │  │
│  └──────────────────────────┴────────────┴──────────┘  │
│                                                        │
│  Access: s3://algoroq-assets/images/hero.jpg           │
└────────────────────────────────────────────────────────┘
```

---

## Object Storage vs File System vs Block Storage

| Feature | Object Storage (S3) | File System (NFS) | Block Storage (EBS) |
|---------|---------------------|-------------------|---------------------|
| Access method | HTTP REST API | File I/O | Block-level I/O |
| Hierarchy | Flat (key is the path) | Directories/folders | None (raw blocks) |
| Scalability | Virtually unlimited | Limited | Limited |
| Latency | Higher (HTTP overhead) | Low | Very low |
| Cost at scale | Very cheap | More expensive | Most expensive |
| Use case | Files, media, backups | Shared file access | Databases, VMs |

---

## How Amazon S3 Achieves 11 Nines of Durability

99.999999999% durability — if you stored 10 million objects, you'd expect to lose one every 10,000 years.

```
  Write: PUT /bucket/object
              │
              ▼
      ┌──────────────┐
      │  Front-End   │  API gateway, auth, routing
      └──────┬───────┘
             │
      ┌──────▼───────┐
      │  Index Layer │  Tracks: key → storage node locations
      └──────┬───────┘
             │  Store on 3+ nodes across different AZs
     ┌───────┼────────┐
     ▼       ▼        ▼
  ┌──────┐ ┌──────┐ ┌──────┐
  │ AZ-A │ │ AZ-B │ │ AZ-C │   ← 3 Availability Zones
  └──────┘ └──────┘ └──────┘

  All 3 must acknowledge before "200 OK" is returned.
```

**How 11 nines is achieved:**
- Data stored in minimum 3 Availability Zones
- Erasure coding (like RAID but across machines) for large objects
- Continuous data integrity checking via checksums
- Automatic detection and repair of corrupted or lost data

---

## Storage Classes and Tiering

Not all data is accessed equally. Lifecycle policies move data automatically between tiers:

```
Hot (frequent access)                           Cold (rare access)
        │                                               │
        ▼                                               ▼
┌─────────────────────────────────────────────────────────────┐
│  S3 Standard    │   S3 IA     │  Glacier   │  Deep Archive  │
│  (immediate)    │  (min 30d)  │  (min 90d) │   (min 180d)   │
│                 │             │            │                │
│  ~$0.023/GB     │ $0.0125/GB  │ $0.004/GB  │  $0.00099/GB   │
│  No retrieval   │  $0.01/req  │ ~4hr ret.  │   ~12hr ret.   │
└─────────────────────────────────────────────────────────────┘

Lifecycle policy example:
  Day 0:    Upload → S3 Standard  (hot, frequent access)
  Day 30:   → S3 IA               (less frequent)
  Day 90:   → Glacier             (archival)
  Day 365:  → Delete
```

---

## Presigned URLs

For secure, temporary access to private objects without exposing credentials:

```
┌──────────┐   1. "I need a presigned URL for upload"   ┌──────────┐
│  Client  │─────────────────────────────────────────────▶│  Server  │
│          │◀── 2. Presigned URL (expires in 1 hour) ────│          │
└──────────┘                                             └──────────┘
     │
     │  3. Client uploads directly to S3
     ▼     (server not in the data path!)
  ┌──────┐
  │  S3  │
  └──────┘
```

**Benefits:**
- Client uploads directly to S3 — your server doesn't proxy large files
- URL expires after a configured time (no permanent public access)
- Can restrict to a specific HTTP method (PUT only, or GET only)

---

## Multipart Upload

For files larger than ~100MB, use multipart upload:

```
1. Initiate:   POST → S3 returns upload_id

2. Upload parts (in parallel, any order):
   Part 1 (5MB): PUT /file?partNumber=1&uploadId=abc
   Part 2 (5MB): PUT /file?partNumber=2&uploadId=abc  } in parallel
   Part 3 (5MB): PUT /file?partNumber=3&uploadId=abc
   Part 4 (2MB): PUT /file?partNumber=4&uploadId=abc

3. Complete: POST → S3 assembles parts into one object

If a part fails → retry just that part (not the whole file)
```

**Benefits:**
- Parallel uploads — 3 parts uploaded simultaneously = 3× throughput
- Resumable — failed parts retry individually without restarting
- Required for objects larger than 5GB

---

## Versioning

Enable versioning to keep full history of every object:

```
PUT hello.txt  → version abc123  →  "Hello World"
PUT hello.txt  → version def456  →  "Hello Universe"
PUT hello.txt  → version ghi789  →  "Hello Algoroq"

GET hello.txt                  → "Hello Algoroq"   (latest)
GET hello.txt?version=abc123   → "Hello World"     (historical)

DELETE hello.txt               → adds delete marker (not truly deleted)
Restore: remove the delete marker → file is back
```

Use for: audit trails, accidental overwrite protection, compliance.

---

## CDN Integration

For public assets, combine object storage with a CDN for global fast delivery:

```
┌──────────┐   PUT    ┌─────────────────┐        ┌────────────┐
│  Upload  │─────────▶│  S3 (origin)    │◀─ miss─│  CDN PoP   │
│  Client  │          └─────────────────┘        │  (edge)    │
└──────────┘                                      └──────┬─────┘
                                                         │ cached
                                                         ▼
                                                    End users globally
```

**Pattern:** S3 is the single source of truth. CloudFront or Cloudflare sits in front. Users request from CDN — cache hit at the nearest edge server, no round trip to S3.

---

## Content-Addressable Storage

Store objects by the hash of their content — identical files share one stored copy:

```
File A: "Hello World"   → SHA256: 64ec88...  → stored
File B: "Hello World"   → SHA256: 64ec88...  → same hash → NOT stored again
File C: "Hello Algo"    → SHA256: 7a9f41...  → different → stored

→ Automatic deduplication, significant storage savings
```

Docker uses content-addressable storage — identical base layers shared between images are stored once.

---

## Cloud Providers Comparison

| Feature | AWS S3 | Google Cloud Storage | Azure Blob |
|---------|--------|---------------------|-----------|
| Max object size | 5 TB | 5 TB | 4.7 TB |
| Durability | 11 nines | 11 nines | 11 nines |
| Storage classes | Standard, IA, Glacier | Standard, Nearline, Coldline | Hot, Cool, Archive |
| Consistency | Strong (since 2020) | Strong | Strong |
| CDN integration | CloudFront | Cloud CDN | Azure CDN |

---

## Key Takeaways

1. **Object storage is flat, HTTP-accessible, and virtually unlimited** — the right home for files, media, and backups
2. **S3 achieves 11 nines durability via 3-AZ replication and erasure coding** — more durable than any on-premise storage
3. **Presigned URLs enable direct client-to-S3 uploads** — your server generates the URL, not the traffic
4. **Multipart upload enables large file parallelism and resumability** — required for files over 5GB
5. **Storage lifecycle tiers dramatically reduce cost** — move cold data to Glacier automatically
6. **Always put a CDN in front for public assets** — global users get edge-cached delivery, S3 serves only cache misses
7. **Versioning is a safety net for production buckets** — enables rollback and audit trails
