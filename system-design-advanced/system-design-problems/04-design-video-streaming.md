---
slug: design-video-streaming
title: Design a Video Streaming Platform
readTime: 30 min
orderIndex: 4
premium: false
---

# Design a Video Streaming Platform

Design a video streaming service like YouTube or Netflix — supporting video upload, transcoding, and global delivery to millions of concurrent viewers.

---

## Step 1: Requirements

### Functional Requirements
- Users can upload videos
- Videos are processed (transcoded) and made available for streaming
- Users can search for and watch videos
- Adaptive quality based on user's bandwidth
- Video metadata: title, description, thumbnails, view count

### Non-Functional Requirements
- High availability: 99.99%
- Low startup latency: video should start playing in < 2 seconds
- Smooth playback: minimal buffering, adaptive bitrate
- Scale: 1 billion users, 500 hours of video uploaded per minute, 1B views/day

---

## Step 2: Back-of-Envelope Estimation

```yaml
Video uploads:
  500 hours/minute = 30,000 seconds of video/minute
  At 1 Mbps average bitrate: 30,000 MB/min ≈ 500 GB/min of raw video
  After transcoding (multiple resolutions): ~5× = 2.5 TB/min storage

Video views:
  1B views/day = 11,574 views/sec
  Average 7 min per view at 2 Mbps = ~18 MB per view
  Outgoing bandwidth: 11,574 × 2 Mbps = 23 Gbps (served via CDN, distributed)

Storage:
  500 GB/min × 60 × 24 = 720 TB/day of new content
  After 10 years: staggering amounts — YouTube reportedly stores 1 exabyte+
```

---

## Step 3: Architecture Overview

Two distinct paths:

```
Upload Path:                              Watch Path:

User ──▶ Upload Service                   User ──▶ CDN Edge
              │                                      │ (cache miss)
              ▼                                      ▼
         Object Storage                   ┌─────────────────┐
         (raw video)                      │  Video Streaming │
              │                           │     Service      │
              ▼                           └─────────────────┘
      Transcoding Pipeline                       │
      (Kafka → Workers)                          ▼
              │                           Object Storage
              ▼                           (transcoded video)
      Object Storage                            │
      (transcoded variants)                     ▼
              │                           CDN (cached)
              ▼                                 │
         CDN (cache)                            ▼
                                          User's player
```

---

## Step 4: Video Upload Pipeline

### Chunked Upload

Large video files (multi-GB) must be uploaded in chunks for reliability:

```
1. Client requests presigned multipart upload URL from Upload Service
2. Client splits video into 5MB chunks
3. Client uploads chunks in parallel directly to S3:
   Part 1: PUT /upload/abc?part=1  (5MB)
   Part 2: PUT /upload/abc?part=2  (5MB)
   Part 3: PUT /upload/abc?part=3  (5MB)
   ...
4. Client sends "complete upload" request
5. S3 assembles parts into full video file
6. S3 triggers notification → Kafka "video.uploaded" event
```

**Benefits:**
- Resume interrupted uploads (only re-upload failed parts)
- Parallel part uploads = faster total upload time
- S3 handles assembly

---

## Step 5: Video Transcoding Pipeline

Raw uploaded video → multiple formats and resolutions for different devices and bandwidths.

### Why Transcode?

```yaml
User uploads: 4K ProRes footage (2 GB, 10 min video)

We need to produce:
  360p  (mobile, slow connection)    → 100 MB
  480p  (standard)                   → 200 MB
  720p  (HD, most common)            → 500 MB
  1080p (Full HD)                    → 1 GB
  1440p (2K)                         → 2 GB
  2160p (4K, premium)                → 4 GB

Plus:
  VP9, H.264, H.265 (different codec support per browser/device)
  HLS and DASH manifests
  Audio tracks (multiple languages)
```

### Transcoding as a DAG

```go
Input: raw_video.mov
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
  Inspect   Audio   Video
  Metadata  Extract Split
      │       │       │
      │   ┌───┘   ┌───┼────────────┐
      │   ▼       ▼   ▼            ▼
      │  AAC  360p H264  720p H264  1080p H265
      │  MP3  (fast encode)         (slower)
      │   │       │   │            │
      └───┴───────┴───┘────────────┘
              │
         Merge: package into HLS/DASH
              │
         Generate thumbnails
              │
         Write to Object Storage
              │
         Update video metadata DB
              │
         Notify user: "Your video is ready"
```

**Worker pool per transcoding job:**
```
"video.uploaded" event → Kafka
Transcoding Coordinator picks up event
Spawns parallel workers:
  Worker A: encode 360p
  Worker B: encode 720p  ← parallel!
  Worker C: encode 1080p ← parallel!
  Worker D: extract audio

When all workers done → merge → upload to S3 → mark video as ready
```

### Codecs Comparison

| Codec | Compression | Browser Support | Use Case |
|-------|------------|----------------|----------|
| H.264 | Good | Universal | Default, widest compatibility |
| H.265 (HEVC) | 50% better than H.264 | Safari, modern | 4K, bandwidth saving |
| VP9 | Similar to H.265 | Chrome, Firefox | YouTube (royalty-free) |
| AV1 | 30% better than VP9 | Modern browsers | Future standard, YouTube/Netflix |

---

## Step 6: Adaptive Bitrate Streaming (ABR)

Instead of one video quality, the player continuously switches quality based on available bandwidth.

### HLS (HTTP Live Streaming — Apple)

```
master_playlist.m3u8 (manifest):
  #EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=640x360
  360p/playlist.m3u8

  #EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
  720p/playlist.m3u8

  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
  1080p/playlist.m3u8

360p/playlist.m3u8 (segment list):
  #EXTINF:6.0,
  segment_001.ts  ← 6-second chunk
  #EXTINF:6.0,
  segment_002.ts
  ...
```

**Player behavior:**
```
1. Download manifest → learn available quality levels
2. Estimate bandwidth (measure segment download speed)
3. Pick quality level that fits bandwidth (with buffer)
4. Download next segment
5. Continuously re-evaluate → switch quality up or down

Bandwidth 500 Kbps  → 360p
Bandwidth 2 Mbps    → 720p
Bandwidth 5+ Mbps   → 1080p
Bandwidth drops suddenly → buffer 720p, switch to 360p
```

---

## Step 7: Video Storage and CDN

```
Object Storage hierarchy:
  s3://videos/
    raw/          ← original uploads (expensive, hot storage for recent)
    transcoded/
      {video_id}/
        360p/      ← segments + playlist
        720p/
        1080p/
        thumbnails/
    archived/     ← old/unpopular videos in Glacier (cheap cold storage)
```

### CDN for Video Delivery

```
User requests: GET /video/{id}/720p/segment_042.ts

1. Request hits CDN edge (user's nearest PoP)
2. Cache hit (segment cached) → return immediately (15ms latency)
3. Cache miss → CDN fetches from S3, caches, returns

CDN caching strategy for video:
  Hot segment (first 5 min of popular video): cached forever
  Tail of unpopular video: not cached (only fetched on demand)
  Manifest files (.m3u8): short TTL (1 min) — updated as video progresses
```

### Reducing S3 Costs with Storage Tiering

```
Day 0–7:   Standard storage (high access during viral period)
Day 7–30:  S3-IA (less access, 40% cost reduction)
Day 30+:   Glacier (rare access, 70% cost reduction)

Exception: popular videos stay in Standard regardless of age
           (trigger: if view count > threshold, don't archive)
```

---

## Step 8: Thumbnail Generation

```sql
During transcoding pipeline:
  Extract frame at 0s, 5s, 10s, 30s, 60s, 120s...
  → Select highest-quality/most-representative frames
  → Resize to thumbnail dimensions (320×180)
  → Upload to S3 with CDN in front

YouTube's approach:
  Generate 12 candidate thumbnails from the video
  Show top creators: A/B test thumbnails
  Default: display most "interesting" frame (ML-selected)
  Custom thumbnails: creators can upload their own
```

---

## Step 9: Video Search

```
At upload time:
  Video metadata (title, description, tags) → indexed in Elasticsearch
  Speech-to-text (ASR) → transcript indexed in Elasticsearch

At search time:
  Query → Elasticsearch → ranked by:
    Text relevance (BM25)
    View count signal
    Recency
    User's watch history (personalized)

Auto-complete:
  Suggest popular search queries via trie or Elasticsearch completion suggester
```

---

## Step 10: Scaling

### Transcoding
- **Auto-scale worker pool:** Kafka consumer group. More workers = more parallelism.
- **Prioritize viral uploads:** Push trending videos to front of transcoding queue.
- **Regional transcoding:** Transcode in the region closest to the uploader.

### Handling Viral Videos
```
Normal video: 1,000 views/day → CDN handles easily
Viral video:  10M views in 1 hour → CDN cache helps but first-hit latency matters

Solution: Proactively push viral videos to more CDN PoPs when:
  - View rate exceeds threshold (rapidly rising views/min)
  - Video shared on social media (signal of incoming traffic spike)
```

---

## Key Takeaways

1. **Video upload and streaming are entirely different paths** — optimize each independently
2. **Chunked multipart upload enables reliability and parallelism** — essential for large files
3. **Transcoding is a DAG of parallel workers** — encode all resolutions simultaneously
4. **Adaptive Bitrate Streaming (HLS/DASH) adjusts quality dynamically** — buffer-free experience
5. **CDN is non-negotiable for video** — serving from origin at scale is impossible
6. **Storage tiering cuts costs dramatically** — most videos are rarely watched after the first week
7. **Metadata indexing enables search** — Elasticsearch with speech-to-text transcripts for content search

---
