# Design: YouTube / Netflix (Video Streaming)

## Problem Statement
Design a video streaming platform where users can upload, process, and watch videos at scale.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Upload + playback, or just playback?" → Both
- "Live streaming or on-demand only?" → On-demand first (live as extension)
- "Adaptive bitrate support?" → Yes (serve based on user's bandwidth)
- "Global users?" → Yes, need CDN
- "What's the scale?" → YouTube scale: 500M DAU, 500 hours of video uploaded/minute

### Functional Requirements
- Users upload videos
- Videos are processed (transcoded to multiple resolutions)
- Users stream videos with adaptive bitrate
- Basic metadata: title, description, tags, like count
- Video search and recommendations (basic)

### Non-Functional Requirements
- Smooth playback (< 2 second start time, no buffering)
- High availability (99.99%)
- Durability: uploaded videos never lost
- Scale: billions of video views/day

---

## Step 2: Capacity Estimation

```
Upload:
  500 hours of video/minute = 30,000 hours/hour
  Avg video = 5 min = 300 sec, 1 GB raw
  500 hours/min = 30,000 min/min of video = 100 GB/min uploaded
  = 100 GB/min × 1440 min/day = 144 TB uploaded/day

Processing overhead (5 resolutions × 3 formats):
  Storage multiplier ≈ 15x
  144 TB × 15 = 2.16 PB/day processed storage

Viewing:
  DAU = 500M, 30 min/day = 250M hours/day
  Bandwidth at 2 Mbps (720p): 250M × 3600 sec × 2 Mbps / 8 = 225 TB/hour of bandwidth
  → Needs massive CDN
```

---

## Step 3: High-Level Architecture

```
[User / Browser]
      |              
      |── Upload ──> [Upload Service] ──> [Raw Video S3]
                            |
                     [Message Queue (Kafka)]
                            |
                   [Video Processing Service]
                   (Transcoding workers)
                            |
                     [Processed Video S3]
                            |
                        [CDN Edge]
                            |
      |── Stream ──> [CDN Edge Server (nearest)]

[Metadata Service] ──> [MySQL (video metadata)]
[Search Service]   ──> [Elasticsearch]
[Recommendation]   ──> [ML Pipeline]
[CDN] ─────────────> [User] (video chunks)
```

---

## Step 4: Video Upload Flow

```
1. User selects video file in browser
2. POST /api/v1/videos/upload/init → returns upload URL (pre-signed S3 URL)
3. Client uploads directly to S3 (multipart upload for large files)
   - Split video into 5-25MB chunks
   - Upload each chunk independently (can resume on failure)
   - S3 assembles chunks into one file
4. Upload complete → S3 triggers Lambda/SNS event
5. Event published to Kafka topic "video_uploaded"
6. Video Processing Workers consume the event

Why client → S3 directly? Avoids your servers becoming a bottleneck for large uploads.
```

---

## Step 5: Video Processing (Transcoding)

This is the most complex part.

```
Raw video (one format, one resolution)
↓
Video Processing Pipeline:
  1. Video segmentation — split into 2-second chunks (HLS/DASH)
  2. Transcoding — convert each chunk to multiple formats/resolutions:
     - 360p (low bandwidth)
     - 480p (standard)
     - 720p (HD)
     - 1080p (Full HD)
     - 4K (ultra)
     × 3 formats: H.264, H.265, VP9
  3. Thumbnail extraction — grab frame at 0%, 25%, 50%, 75%
  4. Audio track extraction (for multi-language)
  5. Upload processed chunks to S3
  6. Update video metadata: status = "ready", manifest URL

Duration:
  1 hour video → 1-3 hours to process at all resolutions
  Use parallel workers (Kubernetes jobs) to speed up
```

### DAG (Directed Acyclic Graph) Architecture
```
Video → [Splitter] → chunks
          ↓ for each chunk:
     [360p worker] [720p worker] [1080p worker] (parallel)
          ↓              ↓             ↓
     [Merger] → final video at each resolution
          ↓
     [Manifest Generator] → m3u8 playlist (HLS)
          ↓
     [Notification] → video is ready
```

---

## Step 6: Video Streaming (Playback)

### Adaptive Bitrate Streaming (ABR)
```
Player downloads a "manifest file" (m3u8) listing all quality levels and chunk URLs.
Player measures bandwidth every few seconds.
Player switches quality level dynamically:
  - Good bandwidth → request 1080p chunk
  - Bandwidth drops → switch to 480p chunk mid-stream
  
Formats:
  HLS (HTTP Live Streaming) — Apple, most common, iOS required
  DASH (Dynamic Adaptive Streaming over HTTP) — Google, more flexible
```

### CDN Strategy
```
[Processed Video S3 (Origin)]
        |
   [CDN (CloudFront / Akamai)]
   - Video chunks cached at edge
   - 95%+ of requests served from CDN (no origin hit)
   - Edge servers deployed in 100+ cities
   
User in Mumbai:
  Request → CDN node in Mumbai (< 10ms)
  Cache miss → CDN fetches from origin, caches
  Next user in Mumbai → served from cache
```

---

## Step 7: Data Model

### videos table
```sql
CREATE TABLE videos (
  id           VARCHAR(11) PRIMARY KEY,   -- YouTube-style ID (e.g., "dQw4w9WgXcQ")
  user_id      BIGINT NOT NULL,
  title        VARCHAR(200),
  description  TEXT,
  status       ENUM('uploading','processing','ready','failed'),
  manifest_url VARCHAR(500),              -- HLS manifest file URL on CDN
  thumbnail_url VARCHAR(500),
  duration_sec INT,
  view_count   BIGINT DEFAULT 0,
  like_count   INT DEFAULT 0,
  created_at   DATETIME
);
```

### View counting at scale
```
Don't: UPDATE videos SET view_count = view_count + 1 per view
  → Millions of DB writes/sec = DB is bottleneck

Do: 
  Write view events to Kafka
  Consumer batch-aggregates and updates DB every 1 minute
  Cache current count in Redis (fast reads)
  Eventual consistency: view count may lag by 1-2 min (fine for YouTube)
```

---

## Step 8: Search & Recommendations

### Search (Elasticsearch)
```
On video publish:
  Index { video_id, title, description, tags, transcript } in Elasticsearch

On search:
  GET /search?q=funny+cats&page=1
  → Elasticsearch full-text query
  → Return ranked video IDs
  → Fetch metadata from MySQL
```

### Recommendations (Basic)
```
Collaborative filtering:
  "Users who watched video A also watched video B"
  → Precomputed offline (Spark batch job overnight)
  → Stored in Redis as sorted sets: similar_videos:{video_id}
  → Served from Redis on page load
```

---

## Step 9: API Design

```
POST /api/v1/videos/upload/init
Response: { "upload_url": "s3.amazonaws.com/...", "video_id": "abc123" }

GET  /api/v1/videos/{id}
Response: { "id", "title", "manifest_url", "thumbnail_url", "view_count" }

GET  /api/v1/videos/{id}/stream
Response: 302 redirect to CDN manifest URL

GET  /api/v1/search?q=query&sort=relevance&page=1

POST /api/v1/videos/{id}/view     ← increment view count (async)
POST /api/v1/videos/{id}/like
```

---

## Interview Tips

- Immediately split the design into **Upload Pipeline** and **Streaming/Playback** — interviewers love this.
- Key insight: "Client uploads directly to S3 — we don't want videos going through our servers."
- Transcoding: "I'd use a worker pool consuming from Kafka. Each resolution processed in parallel."
- CDN is non-negotiable: "Without CDN, we can't serve global video at this scale."

## Common Follow-up Questions
- "How to handle a 4-hour movie upload?" → Multipart upload to S3, resume on failure
- "How to support live streaming?" → Different architecture: RTMP ingest → HLS segments → CDN with short TTL
- "How to prevent piracy?" → Signed CDN URLs that expire, DRM (Widevine, FairPlay)
- "How to detect copyright infringement?" → Content fingerprinting (hash audio/video frames), compare against database
