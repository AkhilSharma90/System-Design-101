# Blob Storage and Object Storage

When you need to store images, videos, backups, logs, or any unstructured binary data at scale, blob/object storage is the answer. It's one of the most cost-effective and durable storage systems ever built — Amazon S3 alone stores trillions of objects.

---

## What is Object Storage?

Object storage treats data as discrete objects, each with a key, value (the binary data), and metadata. Unlike file systems (hierarchical directories) or block storage (raw volumes for databases), object storage is flat and accessed via HTTP APIs.

```
┌────────────────────────────────────────────────────────┐
│                 Object Storage Model                   │
│                                                        │
│  Bucket: "algoroq-assets"                             │
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
│  URL:    https://cdn.algoroq.com/images/hero.jpg       │
└────────────────────────────────────────────────────────┘
```

---

## Object Storage vs File System vs Block Storage

| Feature | Object Storage (S3) | File System (NFS) | Block Storage (EBS) |
|---------|---------------------|-------------------|---------------------|
| Access method | HTTP REST API | File I/O (open, read) | Block-level I/O |
| Hierarchy | Flat (key is the path) | Directories/folders | None (raw blocks) |
| Scalability | Virtually unlimited | Limited | Limited |
| Latency | Higher (HTTP overhead) | Low | Very low |
| Cost at scale | Very cheap | More expensive | Most expensive |
| Use case | Files, media, backups | Shared file access | Databases, VMs |

---

## How Amazon S3 Works Internally

S3's architecture achieves 11 nines of durability (99.999999999%).

```
                    S3 Cluster

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
             │ store on 3+ nodes in different AZs
     ┌───────┼────────┐
     ▼       ▼        ▼
  ┌──────┐ ┌──────┐ ┌──────┐
  │ AZ-A │ │ AZ-B │ │ AZ-C │   ← 3 Availability Zones
  │ Node │ │ Node │ │ Node │
  └──────┘ └──────┘ └──────┘

  All 3 must acknowledge before "200 OK" is returned.
```

**Achieving 11 nines of durability:**
- Data stored in minimum 3 AZs
- Erasure coding (like RAID but across machines) for large objects
- Continuous data integrity checking (checksums)
- Automatic repair of corrupted or lost data

---

## Storage Classes and Tiering

Not all data is accessed equally. Object storage offers multiple tiers:

```
Hot (frequent access)                 Cold (rare access)
        │                                      │
        ▼                                      ▼
┌───────────────────────────────────────────────────┐
│  S3 Standard   │  S3 IA   │ Glacier  │ Deep Archive│
│  (immediate)   │ (min 30d)│ (min 90d)│  (min 180d) │
│                │          │          │             │
│  ~$0.023/GB   │$0.0125/GB│$0.004/GB │  $0.00099/GB│
│  No retrieval │$0.01/req │~4hr ret. │ ~12hr ret.  │
│    fee         │          │  fee     │    fee      │
└───────────────────────────────────────────────────┘

Lifecycle policy: Move object automatically as it ages
  Day 0:  Upload → S3 Standard (frequently accessed)
  Day 30: → S3 IA (less frequent)
  Day 90: → Glacier (archival)
  Day 365: → Delete
```

---

## Presigned URLs

For secure, temporary access to private objects without exposing your credentials:

```
┌──────────┐      1. Request presigned URL    ┌──────────┐
│  Client  │──────────────────────────────────▶│  Your    │
│          │                                  │  Server  │
│          │◀─── 2. Presigned URL (expires) ───│          │
└──────────┘                                  └──────────┘
     │
     │  3. Upload/download directly
     ▼         (bypasses your server!)
┌──────────┐
│   S3     │
└──────────┘

Presigned URL:
https://bucket.s3.amazonaws.com/file.jpg
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKID%2F20240101%2F...
  &X-Amz-Expires=3600    ← expires in 1 hour
  &X-Amz-Signature=abc...
```

**Benefits:**
- Client uploads directly to S3 — your server doesn't proxy large files
- URL expires after a configured time (no permanent public access)
- Can restrict to specific HTTP method (PUT only, or GET only)

---

## Multipart Upload

For files larger than ~100MB, use multipart upload:

```
┌─────────────────────────────────────────────────────┐
│              Multipart Upload Process               │
│                                                     │
│  1. Initiate:   POST → S3 returns upload_id         │
│                                                     │
│  2. Upload parts (in parallel, any order):          │
│     Part 1 (5MB): PUT /file?partNumber=1&uploadId=  │
│     Part 2 (5MB): PUT /file?partNumber=2&uploadId=  │
│     Part 3 (5MB): PUT /file?partNumber=3&uploadId=  │
│     Part 4 (2MB): PUT /file?partNumber=4&uploadId=  │
│                                                     │
│  3. Complete: POST → S3 assembles parts             │
│                                                     │
│  If a part fails → retry just that part             │
│  If upload abandoned → S3 cleans up after lifecycle │
└─────────────────────────────────────────────────────┘
```

**Benefits:**
- Parallel uploads = faster (3 parts uploaded simultaneously = 3× speed)
- Resumable — failed parts can be retried without starting over
- Required for objects > 5GB

---

## Versioning

Enable versioning to keep a history of every object version:

```
PUT hello.txt (version: abc123)  → "Hello World"
PUT hello.txt (version: def456)  → "Hello Universe"
PUT hello.txt (version: ghi789)  → "Hello Algoroq"

GET hello.txt              → returns "Hello Algoroq" (latest)
GET hello.txt?version=abc123 → returns "Hello World"

Delete: doesn't erase — adds a delete marker
Restore: delete the delete marker to "undelete"
```

**Use for:** Audit trails, accidental overwrite protection, compliance requirements.

---

## CDN Integration

For public assets, combine object storage with a CDN:

```
┌──────────┐          ┌────────────────┐         ┌────────────┐
│  Upload  │ PUT ────▶│  S3 (origin)   │◀─ miss ─│   CDN PoP  │
│  Client  │          └────────────────┘         │ (edge)     │
└──────────┘                                      └──────┬─────┘
                                                         │ cached
                                                         ▼
                                                    End users
                                                    (worldwide)
```

**Pattern:**
1. Upload to S3 (private or public bucket)
2. CloudFront / Cloudflare sits in front of S3
3. Users request from CDN URL → cached at edge → fast global access
4. S3 is the single source of truth; CDN is the cache layer

---

## Content-Addressable Storage

Store objects by the hash of their content — identical files share a single stored copy (deduplication):

```
File A: "Hello World"  → SHA256: 64ec88...  → stored once
File B: "Hello World"  → SHA256: 64ec88...  → same hash → NOT stored again
File C: "Hello Algo"   → SHA256: 7a9f41...  → different hash → stored

Storage savings when many files share content (e.g., container layers, backups)
```

Docker layers use content-addressable storage — shared base layers between images are stored once.

---

## Comparison: Cloud Object Storage Providers

| Feature | AWS S3 | Google Cloud Storage | Azure Blob |
|---------|--------|---------------------|-----------|
| Max object size | 5 TB | 5 TB | 4.7 TB |
| Durability | 11 nines | 11 nines | 11 nines |
| Storage classes | Standard, IA, Glacier | Standard, Nearline, Coldline | Hot, Cool, Archive |
| Consistency | Strong (since 2020) | Strong | Strong |
| Multipart | Yes (5MB min part) | Yes (5MB min part) | Yes (blocks) |
| Static website | Yes | Yes | Yes |
| CDN integration | CloudFront | Cloud CDN | Azure CDN |

---

## Key Takeaways

1. **Object storage is flat, HTTP-accessible, and virtually unlimited** — use it for files, media, and backups
2. **S3 achieves 11 nines durability via 3-AZ replication and erasure coding** — more durable than any on-premise storage
3. **Presigned URLs enable direct client-to-S3 uploads** — your server only generates the URL, not the traffic
4. **Multipart upload is required for large files** — enables parallelism and resumability
5. **Storage tiers dramatically reduce cost** — move infrequently accessed data to IA/Glacier
6. **Always put a CDN in front for public assets** — global users get edge-cached delivery
7. **Versioning and lifecycle policies give you control** — enable for production buckets as a safety net

---

*Previous: [Search Systems ←](08-search-systems.md) · Next: [Monitoring and Observability →](10-monitoring-and-observability.md)*
