# Chapter 8: Storage & CDN

## Storage Types

```
┌──────────────────┬────────────────────┬──────────────────────────────┐
│ Type             │ Examples           │ Use Case                     │
├──────────────────┼────────────────────┼──────────────────────────────┤
│ Block Storage    │ EBS, local SSD     │ Databases, OS disks          │
│ File Storage     │ EFS, NFS           │ Shared file systems          │
│ Object Storage   │ S3, GCS, Azure Blob│ Images, videos, backups      │
└──────────────────┴────────────────────┴──────────────────────────────┘
```

---

## Object Storage (S3)

The default for storing **unstructured data** (images, videos, files, backups).

```
Properties:
- Unlimited scale (stores trillions of objects)
- 99.999999999% (11 nines) durability
- Versioning, lifecycle policies
- Cost: ~$0.023/GB/month (standard tier)

Access Patterns:
- PUT: Upload object
- GET: Download object (direct or via CDN)
- Pre-signed URL: Temporary access without making bucket public

Tiers (cost vs access time):
┌────────────────────┬──────────┬──────────────────┐
│ Tier               │ Cost     │ Access Time      │
├────────────────────┼──────────┼──────────────────┤
│ Standard           │ $$       │ Milliseconds     │
│ Infrequent Access  │ $        │ Milliseconds     │
│ Glacier            │ ¢        │ Minutes to hours │
│ Glacier Deep       │ ¢¢       │ 12+ hours        │
└────────────────────┴──────────┴──────────────────┘
```

### Upload Pattern for Large Files
```
Client → Pre-signed URL from App Server
Client → Direct upload to S3 (bypasses your servers!)
S3 → Event notification → Process (thumbnail, transcode)

For very large files: Multipart upload (parallel chunks)
```

---

## CDN Deep Dive

### How CDN Works
```
First request (cache MISS):
User (Mumbai) → CDN Edge (Mumbai) → Origin (US) → CDN caches → User

Subsequent requests (cache HIT):
User (Mumbai) → CDN Edge (Mumbai) → User  [fast! ~20ms]

Cache Key: Usually URL + headers
TTL: Configured per content type
```

### CDN Invalidation
```
1. TTL expiry     → Content refreshed after time
2. Purge/Flush    → Manually clear specific URLs
3. Versioned URLs → style.v2.css (never needs invalidation)
4. Cache-Control  → Headers control caching behavior

Best practice: Use versioned URLs for static assets
  /static/app.a1b2c3.js  ← hash in filename, long TTL
```

### CDN for Dynamic Content
```
Not just static files! Modern CDNs can:
- Cache API responses (short TTL: 5-30 seconds)
- Edge compute (Cloudflare Workers, Lambda@Edge)
- A/B testing at the edge
- Geolocation-based routing
```

---

## Blob Storage Patterns in System Design

### Image Upload & Serving
```
┌────────┐  1.upload   ┌───────────┐  2.store   ┌─────┐
│ Client │────────────▶│ App Server│───────────▶│  S3 │
└────────┘             └───────────┘            └──┬──┘
                                                   │
    4.serve via CDN    ┌───────────┐  3.origin    │
◀──────────────────────│    CDN    │◀─────────────┘
                       └───────────┘

Metadata (filename, user, size) → Database
Actual file → Object Storage (S3)
Serving → CDN (cached at edge)
```

### Video Processing Pipeline
```
Upload → S3 (raw) → Transcode Queue → Workers → S3 (multiple qualities)
                                                      │
                                              ┌───────┴───────┐
                                              │ 240p 360p 720p │
                                              │ 1080p 4K       │
                                              └───────┬───────┘
                                                      │
                                                    CDN → Adaptive bitrate streaming
```

---

## Data Lake Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Data Lake (S3)                        │
├─────────────┬─────────────┬─────────────┬───────────────────┤
│   Raw Zone  │ Cleaned Zone│ Curated Zone│  Consumption Zone │
│  (landing)  │ (validated) │ (enriched)  │  (ready to query) │
├─────────────┼─────────────┼─────────────┼───────────────────┤
│   JSON/CSV  │   Parquet   │   Parquet   │  Parquet/Delta    │
│   as-is     │  schema     │  joined     │  aggregated       │
└─────────────┴─────────────┴─────────────┴───────────────────┘
        ▲               ▲             ▲
        │               │             │
   Ingestion       ETL (Spark)   ML Features
   (Kafka/Kinesis) (Glue/EMR)   (SageMaker)
```

---

## Interview Tips

1. **Never store files in your database** — always use object storage
2. **Always mention CDN** for user-facing content (images, videos, static assets)
3. **Pre-signed URLs** — let clients upload directly to S3 (saves server bandwidth)
4. **Lifecycle policies** — move old data to cheaper tiers automatically
5. **Replication** — "S3 replicates across 3+ AZs automatically for durability"
