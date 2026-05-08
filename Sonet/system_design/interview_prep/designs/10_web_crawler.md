# Design: Web Crawler / Search Engine Indexer

## Problem Statement
Design a web crawler that discovers and indexes web pages at scale (like Googlebot).

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Just crawl, or also index and serve search results?" → Just the crawl + storage
- "Scale? How much of the web?" → 1 billion pages
- "Recrawl frequency?" → Popular pages monthly, others less often
- "Handle JavaScript-rendered pages?" → Static HTML first (extend with JS later)
- "Respect robots.txt?" → Yes
- "Distributed or single machine?" → Distributed

### Functional Requirements
- Start from seed URLs, discover and crawl the web
- Respect robots.txt and crawl-delay
- Store web page content and extracted URLs
- Detect duplicate content (don't store same page twice)
- Re-crawl pages periodically based on change frequency

### Non-Functional Requirements
- Scale: 1 billion pages
- Polite: don't overload target servers
- Distributed: handle failures of individual crawlers
- Deduplicated: same URL not crawled twice per cycle

---

## Step 2: Capacity Estimation

```
Target: 1 billion pages
Avg web page = 100 KB (HTML)

Storage:
  HTML content: 1B × 100 KB = 100 TB
  Metadata (URL, crawled_at, etc.): 1B × 1 KB = 1 TB
  Total ≈ 100 TB

Bandwidth:
  Crawl 1B pages in 30 days:
  Rate = 1B / (30 × 86,400) ≈ 385 pages/sec
  Bandwidth = 385 × 100 KB = 38.5 MB/sec ≈ 300 Mbps

With 100 crawler workers:
  Each fetches ≈ 4 pages/sec (reasonable, respectful rate)
```

---

## Step 3: High-Level Architecture

```
[Seed URLs] ─────────────────────────────────┐
                                             ↓
                                     [URL Frontier]
                                     (Priority Queue)
                                          |
                                    [Dispatcher]
                                   / | | | | | \
                          [Crawler 1..N Workers]
                                     |
                          [DNS Cache / Resolver]
                                     |
                         [Target Web Servers]
                                     |
                           (HTML content fetched)
                                     |
                 +───────────────────+───────────────────+
                 |                                       |
        [Content Parser]                        [URL Extractor]
                 |                                       |
        [Duplicate Detector]                   [URL Normaliser]
        (Content hash check)                             |
                 |                              [URL Seen? Check]
                 |                           (Bloom filter / Redis)
                 |                                       |
        [Content Store (S3)]                   new URLs ↓
                 |                          [URL Frontier] ← loop
        [Metadata DB (Cassandra)]
```

---

## Step 4: URL Frontier

The **URL Frontier** is the queue of URLs to crawl. It must:
1. **Prioritise** important URLs (PageRank, freshness signal)
2. **Be polite** (don't hammer one domain too fast)
3. **Be large** (billions of URLs in queue)

### Two-Level Queue Design
```
Level 1: Priority Queue (Front)
  Multiple queues by priority:
    High priority (news sites, popular domains): queue_high
    Medium priority:                             queue_medium
    Low priority (rarely updated, obscure):     queue_low
  
  Prioritiser assigns priority based on:
    - Domain popularity (PageRank signal)
    - Last-modified header / change frequency
    - Manual overrides (seed URLs)

Level 2: Politeness Queue (Back)
  One queue per domain:
    queue_nytimes.com
    queue_github.com
    queue_bbc.co.uk
  
  Dispatcher reads from Level 1, routes to Level 2 domain queues.
  Ensures min 1 second between requests to same domain.

Crawler workers take from domain queues (politely).
```

---

## Step 5: Deduplication

### URL Deduplication (Don't Crawl Same URL Twice)
```
Bloom Filter (fast, memory-efficient):
  Before adding URL to frontier, check Bloom Filter.
  "Definitely not seen" → add to frontier.
  "Probably seen" → skip (accept tiny false positive rate).
  
  Space: 1 billion URLs × ~10 bits = 10 GB RAM (very efficient)
  
  On restart: Bloom Filter rebuilt from seen_urls DB.
```

### URL Normalisation
```
Before dedup check, normalise URLs:
  https://Example.com/Page → https://example.com/page  (lowercase)
  http://example.com/a%20b → http://example.com/a%20b  (decode then re-encode)
  http://example.com/a/../b → http://example.com/b     (resolve path)
  Remove: tracking params (?utm_source=, ?ref=)
  Remove: fragment (#section1 — not a different page)
```

### Content Deduplication (Don't Store Same Content Twice)
```
After fetching a page, compute SimHash of content.
SimHash: 64-bit hash where similar content produces similar hashes.

Check SimHash against seen_hashes DB.
If Hamming distance < threshold → near-duplicate → skip storage.

Use case: Mirror sites, scraped content, pagination (page 2, 3, 4 often same)
```

---

## Step 6: robots.txt Compliance

```
Before crawling a domain for the first time:
  1. Fetch https://example.com/robots.txt
  2. Parse rules:
     User-agent: *
     Disallow: /admin/
     Disallow: /private/
     Crawl-delay: 10    ← respect this!
     Sitemap: https://example.com/sitemap.xml  ← crawl this for URLs

  3. Cache robots.txt for 24 hours
  4. Check every URL against rules before fetching

Non-compliance risks:
  - Being blocked by target site
  - Legal issues (CFAA in US)
  - Ethical issues
```

---

## Step 7: Recrawling Strategy

```
Not all pages change at same rate:
  News pages: change hourly → recrawl frequently
  Wikipedia: changes weekly → recrawl weekly
  Old blog posts: never change → recrawl quarterly

Signals to estimate change frequency:
  - Last-Modified HTTP header
  - ETag (content version)
  - Historical change rate (how often did it change in past crawls)
  - Domain type (news vs static)

Recrawl priority:
  Priority score = freshness_weight / time_since_last_crawl
  High-traffic pages get more freshness weight.
```

---

## Step 8: Distributed Crawler Coordination

```
Multiple crawler machines must coordinate to avoid:
  1. Crawling same URL twice simultaneously
  2. Overloading the URL Frontier

Solution: Consistent Hashing
  URL → hash → assigned crawler machine
  Each machine owns a subset of URL space.
  If machine fails → redistribute its URL range.

Alternative (simpler): 
  Zookeeper coordinates domain assignments.
  Each crawler claims a set of domains.
  URL Frontier is a shared queue (Redis Streams or Kafka).
```

---

## Step 9: Data Storage

```
Crawled content: S3 (raw HTML, one file per page)
  Key: crawl/{domain}/{url_hash}/{timestamp}.html

Metadata DB (Cassandra — write-heavy, time-series):
CREATE TABLE crawl_metadata (
  url_hash       VARCHAR(64) PRIMARY KEY,
  url            TEXT,
  domain         VARCHAR(255),
  crawled_at     TIMESTAMP,
  status_code    INT,
  content_hash   VARCHAR(64),
  content_type   VARCHAR(50),
  last_modified  TIMESTAMP,
  links_found    INT,
  next_crawl_at  TIMESTAMP
);

URL Frontier (Redis / Kafka):
  Priority queue: Redis Sorted Set with score = priority
  Domain queues: Redis Lists per domain
```

---

## Step 10: API Design (Internal)

```
POST /crawl/submit      ← submit URL for crawling
  Body: { "url": "https://...", "priority": "high" }

GET  /crawl/{url_hash}  ← get crawl status and metadata

POST /crawl/bulk        ← submit many URLs (sitemap upload)

GET  /crawl/status      ← overall crawl progress stats
```

---

## Interview Tips

- Start with: "The URL Frontier is the heart of the crawler — it must be prioritised and polite."
- Key insight: "Two-level queue: priority queue to decide what to crawl next, domain queues for politeness."
- Always mention: "Bloom Filter for URL deduplication — 1B URLs fit in 10 GB RAM."
- Robots.txt: "Always mention compliance — interviewers notice when you skip this."

## Common Follow-up Questions
- "How to handle JavaScript-heavy pages?" → Headless browser (Puppeteer/Playwright); expensive — reserved for priority pages
- "How to detect spam/malicious pages?" → URL pattern blocklist, content heuristics, ML spam classifier
- "How to scale to 10B pages?" → Increase partitions, add crawler workers, distributed URL Frontier on Kafka
- "How to build the search index from crawled data?" → Inverted index pipeline: tokenise → stem → TF-IDF → Elasticsearch
