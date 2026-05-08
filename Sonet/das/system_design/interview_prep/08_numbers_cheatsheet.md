# Numbers Every Engineer Should Know

Knowing these numbers lets you do capacity estimation in 2 minutes during an interview.

---

## Time Units

```
1 second = 1,000 milliseconds (ms)
1 ms     = 1,000 microseconds (μs)
1 μs     = 1,000 nanoseconds (ns)
```

---

## Latency Numbers (2024 reference)

| Operation | Time | Relative |
|-----------|------|---------|
| L1 cache reference | 0.5 ns | Fastest |
| Branch misprediction | 5 ns | — |
| L2 cache reference | 7 ns | 14x L1 |
| Mutex lock/unlock | 25 ns | — |
| Main memory (RAM) reference | 100 ns | 200x L1 |
| Compress 1 KB with Snappy | 3,000 ns = 3 μs | — |
| Send 2 KB over 1 Gbps network | 20,000 ns = 20 μs | — |
| Read 1 MB from RAM | 250 μs | — |
| Round trip within same datacenter | 500 μs = 0.5 ms | — |
| Read 1 MB from SSD | 1 ms | 4x network |
| **Redis/Memcached read** | **< 1 ms** | Target for cache |
| **DB query (indexed)** | **1–10 ms** | Simple query |
| Read 1 MB from HDD | 20 ms | 80x SSD |
| Send packet US → EU → US | 150 ms | Cross-continent |
| **Target page load time** | **< 200 ms** | User expectation |
| Human perception of "instant" | < 100 ms | UX target |

**Key insight:** Memory access is 200x faster than disk. Cache is 10–100x faster than DB.

---

## Data Size Units

```
1 KB    = 1,024 bytes    ≈ 1,000 bytes
1 MB    = 1,024 KB       ≈ 1 million bytes
1 GB    = 1,024 MB       ≈ 1 billion bytes
1 TB    = 1,024 GB       ≈ 1 trillion bytes
1 PB    = 1,024 TB
```

### Typical Object Sizes
| Object | Size |
|--------|------|
| ASCII character | 1 byte |
| UUID | 36 bytes |
| Tweet text (280 chars) | ~500 bytes |
| Average JSON API response | 1–5 KB |
| Webpage (HTML) | 50–100 KB |
| Profile photo (thumbnail) | 100–500 KB |
| Profile photo (full) | 1–5 MB |
| Song (MP3) | 3–5 MB |
| 1-minute video (compressed) | 50–200 MB |
| Feature-length movie (HD) | 4–10 GB |

---

## QPS Conversion (Requests Per Second)

```
1 million  requests/day  = 1,000,000 / 86,400 ≈ 11.5 RPS
10 million requests/day  ≈ 115 RPS
100 million requests/day ≈ 1,150 RPS
1 billion  requests/day  ≈ 11,574 RPS ≈ 12,000 RPS

Easy formula: N million req/day ≈ N × 12 RPS
```

---

## Time Conversions

```
1 minute  = 60 seconds
1 hour    = 3,600 seconds
1 day     = 86,400 seconds  ← memorise this
1 month   = 2.6 million seconds  ≈ 30 × 86,400
1 year    = 31.5 million seconds ≈ 365 × 86,400
```

---

## Capacity Estimation Templates

### Step 1: Define Scale
```
DAU (Daily Active Users) = ?
Read/Write ratio = ? (e.g., 100:1 read-heavy for Twitter feed)
Average payload size = ?
```

### Step 2: Calculate QPS
```
Write QPS = DAU × writes_per_user / 86,400
Read QPS  = Write QPS × read_write_ratio
Peak QPS  = average QPS × 2–5 (spike factor)
```

### Step 3: Calculate Storage
```
Daily new data = Write QPS × avg_object_size × 86,400
Monthly storage = Daily × 30
5-year storage  = Daily × 1,825

Add 20–30% overhead for metadata, indexes, replication.
```

### Step 4: Calculate Bandwidth
```
Incoming bandwidth = Write QPS × avg_object_size
Outgoing bandwidth = Read QPS × avg_object_size
```

---

## Example: Design Twitter

```
Given:
  DAU = 300 million
  Each user tweets 0.1x/day (1 tweet per 10 days on average)
  Each user reads 100 tweets/day
  Average tweet size = 500 bytes
  Average tweet with media = 1 MB (10% of tweets have media)

Write QPS:
  Tweets/day = 300M × 0.1 = 30 million tweets/day
  Write QPS  = 30M / 86,400 ≈ 347 RPS (text)
  Media writes = 347 × 0.1 = 35 media uploads/sec

Read QPS:
  Reads/day = 300M × 100 = 30 billion/day
  Read QPS  = 30B / 86,400 ≈ 347,000 RPS
  Read/Write ratio ≈ 1000:1

Storage (text only):
  30M tweets/day × 500 bytes = 15 GB/day
  5-year storage  = 15 GB × 365 × 5 ≈ 27 TB

Storage (media):
  30M × 0.1 × 1 MB = 3 million MB/day = 3 TB/day
  5-year: 3 TB × 365 × 5 ≈ 5.5 PB
```

---

## Availability Math

```
Availability % | Downtime/year | Downtime/month | Downtime/week
    99.0%      |  3.65 days    |  7.31 hours    |  1.68 hours
    99.9%      |  8.77 hours   |  43.83 min     |  10.08 min
    99.99%     |  52.6 min     |  4.38 min      |  1.01 min
    99.999%    |  5.26 min     |  26.3 sec       |  6.05 sec

Formula: Downtime = (1 - availability) × time_period
```

---

## Network Bandwidth Reference

```
Ethernet:
  100 Mbps = 12.5 MB/s
  1 Gbps   = 125 MB/s
  10 Gbps  = 1.25 GB/s  (typical datacenter link)

Mobile:
  4G LTE  ≈ 20–50 Mbps
  5G      ≈ 100–400 Mbps
  WiFi 6  ≈ up to 1 Gbps (real-world: ~200 Mbps)
```

---

## Powers of 2 (Useful for Binary Calculations)

```
2^10 = 1,024   ≈ thousand (1K)
2^20 = 1M      ≈ million
2^30 = 1B      ≈ billion
2^32 = 4.3B    ← max 32-bit integer, IPv4 addresses
2^40 = 1T      ≈ trillion
2^64 = 18.4 × 10^18  ← max 64-bit unsigned int
```

---

## Interview Quick-Math Cheatsheet

| I want to know | Quick formula |
|---------------|---------------|
| RPS from DAU | DAU × actions/day ÷ 86,400 |
| Daily storage | Write_RPS × obj_size × 86,400 |
| Servers needed | Peak_RPS ÷ RPS_per_server (assume 1,000–10,000 RPS/server) |
| RAM for cache | Hot_items × obj_size (cache 20% of data covers 80% of reads) |
| Bandwidth in | Write_RPS × obj_size |
| Bandwidth out | Read_RPS × obj_size |
