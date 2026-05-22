# Chapter 2: Back-of-the-Envelope Estimation

## Why Estimation Matters

Estimation tells you whether you need 1 server or 1000. It drives architecture decisions:
- Should I cache? (depends on QPS)
- Should I shard? (depends on data volume)
- Do I need a CDN? (depends on bandwidth)

---

## Key Numbers Every Engineer Must Know

### Latency Numbers (2024 Reference)

```
Operation                              Time
─────────────────────────────────────────────────
L1 cache reference                     1 ns
L2 cache reference                     4 ns
RAM reference                          100 ns
SSD random read                        16 μs
HDD random read                        2 ms
Send 1KB over network (same DC)        0.5 ms
Read 1 MB from SSD                     1 ms
Read 1 MB from HDD                     20 ms
Read 1 MB from network                 10 ms
Round trip within same datacenter      0.5 ms
Round trip CA → Netherlands            150 ms
```

### Throughput Numbers

```
Component                    Throughput
─────────────────────────────────────────────────
Single MySQL instance        ~ 10K QPS (reads)
Single PostgreSQL            ~ 10K-30K QPS
Single Redis instance        ~ 100K-500K QPS
Kafka (single partition)     ~ 100K msgs/sec
Single web server            ~ 1K-10K req/sec
Load balancer (Nginx)        ~ 100K+ req/sec
```

### Storage & Size

```
Item                         Size
─────────────────────────────────────────────────
1 char (ASCII)               1 byte
1 char (UTF-8, typical)      1-4 bytes
UUID                         16 bytes
Timestamp                    8 bytes
URL (average)                100 bytes
Tweet text                   280 bytes (max)
Email (average)              50 KB
Photo (compressed)           200 KB - 2 MB
Video (1 min, 720p)          ~50 MB
Video (1 min, 4K)            ~400 MB
```

---

## Power of 2 Quick Reference

```
Power    Exact Value          Approx       Bytes
─────────────────────────────────────────────────
10       1,024               1 Thousand    1 KB
20       1,048,576           1 Million     1 MB
30       1,073,741,824       1 Billion     1 GB
40       ~1.1 × 10¹²        1 Trillion    1 TB
50       ~1.1 × 10¹⁵        1 Quadrillion 1 PB
```

---

## Estimation Templates

### Template 1: QPS Estimation

```
Given: X million DAU (Daily Active Users)

Daily requests per user: N
Total daily requests: X million × N

QPS = Total daily requests / 86,400 seconds
Peak QPS = QPS × 2 (or ×3 for spiky traffic)

Example: Twitter
- 300M DAU, each views 10 tweets/day
- Read QPS: 300M × 10 / 86400 ≈ 35K QPS
- Peak: 70K-100K QPS
```

### Template 2: Storage Estimation

```
Given: X new records/day, each Y bytes

Daily storage: X × Y bytes
Yearly storage: Daily × 365
5-year storage: Yearly × 5

Example: URL Shortener
- 100M new URLs/day × 500 bytes = 50 GB/day
- Year: 50 × 365 = 18 TB
- 5 years: 90 TB
```

### Template 3: Bandwidth Estimation

```
Given: QPS and average response size

Bandwidth = QPS × response size

Example: Image Service
- 50K QPS × 200 KB = 10 GB/sec outbound
- This means we need CDN (single server can't handle this)
```

### Template 4: Number of Servers

```
Given: Peak QPS and QPS per server

Servers needed = Peak QPS / QPS per server

Example:
- Peak QPS: 100K
- Each app server handles: 5K QPS
- Need: 100K / 5K = 20 servers (+ buffer = 25-30)
```

---

## Common Estimation Scenarios

### Twitter-scale

```
┌─────────────────────────────────────────────────┐
│ Users: 500M total, 300M DAU                     │
│ Tweets/day: 500M (300M reads, 200M writes)      │
│ Write QPS: ~6K/sec                              │
│ Read QPS: ~300K/sec (timeline requests)         │
│ Tweet storage: 500M × 300B = 150 GB/day         │
│ Media: 10% have images = 50M × 500KB = 25 TB/day│
│ → Need: Heavy caching, CDN, read replicas       │
└─────────────────────────────────────────────────┘
```

### WhatsApp-scale

```
┌─────────────────────────────────────────────────┐
│ Users: 2B total, 1B DAU                         │
│ Messages/day: 100B (100 msgs × 1B users)        │
│ QPS: ~1.2M messages/sec                         │
│ Message size: 100 bytes avg                     │
│ Daily storage: 100B × 100B = 10 TB/day          │
│ → Need: Sharding, message queues, WebSockets    │
└─────────────────────────────────────────────────┘
```

### YouTube-scale

```
┌─────────────────────────────────────────────────┐
│ Users: 2B total, 1B DAU                         │
│ Videos watched/day: 5B (5 per user)             │
│ Videos uploaded/day: 500K                        │
│ Average video: 5 min × 50MB = 250MB             │
│ Upload storage: 500K × 250MB = 125 TB/day       │
│ Bandwidth (views): 5B × 10MB avg = 50 PB/day   │
│ → Need: CDN is CRITICAL, encoding pipeline      │
└─────────────────────────────────────────────────┘
```

---

## Estimation Cheat Sheet (Memorize These)

```
1 day = 86,400 seconds ≈ 10⁵ seconds
1 million requests/day ≈ 12 QPS
1 billion requests/day ≈ 12,000 QPS

100M users × 1KB data = 100 GB
1B users × 1KB data = 1 TB

1 server serves ~1K-10K QPS (depends on complexity)
1 database serves ~10K QPS (reads with indexes)
Redis: ~100K QPS per instance
```

---

## Practice Problems

1. **Estimate storage for Instagram** — 2B users, 100M photos uploaded daily, avg 2MB each
2. **Estimate QPS for Google Search** — 8.5B searches/day
3. **How many chat servers for WhatsApp?** — 100B messages/day, each server handles 50K connections
4. **Storage for Uber trips** — 20M trips/day, each trip = 2KB metadata + GPS points
5. **CDN bandwidth for Netflix** — 200M subscribers, 2 hours/day average, 5 Mbps stream

> Practice doing these in 2-3 minutes. In interviews, you need to estimate quickly and move on.
