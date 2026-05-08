# Real-World Architectures — How Big Tech Actually Built It

## Google

### Google Search Architecture
```
Web Crawl Pipeline:
  Googlebot crawlers → raw HTML stored in GFS (Google File System)
  MapReduce jobs: parse, index, extract links
  PageRank algorithm run as MapReduce batch job (weekly)
  Inverted index built from extracted terms + PageRank scores
  
Serving:
  Query → spell check → query expansion → index lookup → ranking → result blend
  Bigtable stores index segments distributed across thousands of machines
  
Scale (2024 estimates):
  Crawls: billions of pages/day
  Queries: ~8.5 billion/day (~100,000/sec)
  Index: hundreds of billions of web pages
  Ranking: 200+ signals, ML-ranked (RankBrain, BERT, MUM)
```

### Bigtable (Google's Wide-Column Store)
```
Architecture:
  Row key → sorted, lexicographic → range scans efficient
  Row → multiple column families → each family = multiple columns
  
  Tablet = sorted range of rows (typically 100-200 MB)
  Tablet Server: serves 10-1000 tablets
  Master: assigns tablets to tablet servers, load balances
  
  Write path:
    1. Write to commit log (GFS)
    2. Write to in-memory memtable
    3. When memtable full → flush to SSTable file (GFS)
    4. Compaction: merge SSTables periodically
  
  This is literally the LSM-Tree architecture — Bigtable paper (2006) described it.
  HBase, Cassandra, LevelDB, RocksDB all implement this pattern.
```

### Spanner (Google's Global SQL Database)
```
The problem: How do you run SQL with global ACID transactions at Google scale?

TrueTime API:
  Google's datacenters have GPS clocks and atomic clocks.
  TrueTime gives a timestamp interval: [earliest, latest] — guaranteed to contain real time.
  
  Assigns globally unique commit timestamps with uncertainty window < 7ms.
  Two commits guaranteed ordered if their time intervals don't overlap.
  
  This is how Spanner achieves serialisability without a global lock.

Implications:
  SQL with full ACID across globally distributed data
  External consistency: if T1 commits before T2 starts, T1's timestamp < T2's
  Used internally by: Google Ads, Play Store, YouTube metadata
  Available as: Google Cloud Spanner
```

---

## Amazon / AWS

### Amazon.com Architecture Evolution
```
2001: Single monolith ("obidos") serving all of amazon.com
2004: Team decides to decompose into services
      Jeff Bezos mandate: "Every team must expose its data/functionality via service interfaces.
                           No other form of interprocess communication is allowed."
2006: AWS launches (EC2, S3) — Amazon eating its own dog food
2010: Amazon.com migrated to its own AWS services

Today:
  ~500 microservices for amazon.com
  Services communicate via REST/gRPC APIs
  All state in AWS managed services (DynamoDB, Elasticache, RDS)
```

### DynamoDB Design (Werner Vogels' Vision)
```
Inspired by Amazon's Dynamo paper (2007) — highly influential.

Core design decisions:
  1. Always writable: sacrifices consistency for availability
     Even during network partition, writes accepted
     
  2. Eventual consistency as default
     Strong consistency available at cost of latency
     
  3. Consistent hashing with virtual nodes for partitioning
     N=3 replication by default
     
  4. Conflict resolution: "last writer wins" or application-level
  
Scale:
  Tens of millions of requests/second
  Millions of customers, multiple AWS regions
  
2022 rewrite paper revealed:
  Moved to B-Trees for storage (from LSM-Trees in original Dynamo)
  B-Trees give better read performance; writes still fast with local SSDs
```

### AWS S3 Internals (Simplified)
```
S3 is the world's largest object store.

Internal structure:
  PUT /bucket/key → front-end web server → chunk into 64 MB parts
  → write to 3+ independent storage nodes (quorum write)
  → metadata stored in SimpleDB/DynamoDB (bucket, key, ACL, storage location)
  
Durability 11 nines (99.999999999%):
  Data erasure-coded across multiple availability zones
  Not just 3 copies — Reed-Solomon erasure coding (6+3 scheme)
  Can reconstruct full object from any 6 of 9 fragments
  
  Effective redundancy without 3x storage cost.
```

---

## Meta (Facebook)

### Instagram Feed Architecture
```
Initial architecture (2012, 1M DAU):
  Django web servers + PostgreSQL (100% on AWS)
  
At scale (2B+ MAU today):
  Feed generation: Hybrid fan-out model (see twitter_feed.md)
  
Unicorn (Facebook Graph Search):
  Graph index over social connections
  Real-time updates as people connect, post
  
TAO (The Associations and Objects):
  Facebook's social graph database
  Objects: users, pages, events
  Associations: friendships, likes, shares, comments
  
  Distributed cache + MySQL shards
  Write-through cache + async replication
  Single-datacenter reads when possible
  
Scale: hundreds of billions of objects
       trillions of associations
       billions of reads/second
```

### Facebook's Memcached at Scale
```
2013 paper: "Scaling Memcache at Facebook"

They added Memcached as a look-aside cache for MySQL.
Key challenges:

1. Cache stampede (thundering herd):
   Solution: "Lease" mechanism — one client gets a lease to populate cache.
   Others receive a "wait and retry" response. No stampede.

2. Stale reads after writes (race condition):
   Write path: update MySQL → delete cache key
   Read path: cache miss → read MySQL → set cache
   Race: between delete and read, old value returned
   Solution: Lease invalidates races automatically.

3. Regional pools:
   Some data too large for one cluster → replicated across regional pools
   Different Memcached pools for different data sizes (photo metadata vs user profiles)

4. Cold cluster warmup:
   New Memcached cluster → all misses → DB overloaded
   Solution: forward cache misses to existing warm cluster first
   Gradually warm new cluster from existing hot one
```

---

## Netflix

### Netflix Chaos Engineering
```
Birth of Chaos Engineering:
  2010: Netflix migrated to AWS
  Problem: AWS failures are common; need to build resilient systems
  
  Simian Army:
    Chaos Monkey: kills random EC2 instances in production
    Chaos Gorilla: simulates AZ failure
    Chaos Kong: simulates region failure
    Janitor Monkey: deletes unused AWS resources (cost saving)
    Latency Monkey: injects artificial latency
    
  Philosophy: "The best way to avoid failure is to fail constantly."
  
Today: Chaos Engineering is mainstream (Google, Amazon, LinkedIn all do this)
```

### Netflix Content Delivery
```
Open Connect CDN (Netflix's own CDN):
  Instead of paying Akamai/CloudFront, Netflix built its own CDN.
  Open Connect Appliances (OCAs): custom hardware boxes deployed at ISPs.
  
  Pre-positioning:
    Netflix knows what will be popular (new releases on Fridays).
    Proactively pushes content to OCA servers before peak time.
    90%+ of traffic served from OCA at ISP (< 10ms latency for users).
  
  Result:
    Netflix is ~15% of all downstream internet traffic at peak.
    Served efficiently from ISP-hosted boxes.
    No egress cost from AWS (major savings).
```

### Netflix Microservices (Hystrix, Ribbon, Eureka)
```
Eureka: Service discovery
  Each microservice registers with Eureka on startup.
  Consumers ask Eureka for instance list → client-side load balancing.
  
Ribbon: Client-side load balancer
  Round-robin, weighted response time, availability zone awareness.
  
Hystrix: Circuit breaker (now Resilience4j in newer stacks)
  Wrap every external call in Hystrix Command.
  Automatic circuit breaking, bulkhead, fallback.
  Hystrix Dashboard: real-time visualisation of circuit states.
  
These became open-source → became the blueprint for Java microservices everywhere.
```

---

## Twitter

### Twitter's Move from Ruby to JVM
```
2012: Twitter rewrote core services from Ruby on Rails to JVM (Scala + Finagle)
Reason: Ruby GIL (Global Interpreter Lock) prevented true parallelism.
        JVM handles millions of concurrent connections per node.

Finagle:
  Twitter's RPC framework for Scala/Java
  First system to implement: circuit breakers, retries, load balancing as library
  Later inspired: Netflix OSS, Go's stdlib, many others
```

### Twitter's Timeline Architecture
```
Timeline service uses Redis for pre-built timelines.

Original (2012): Precompute timelines for all users via fan-out on write.
Problem: Beyoncé with 30M followers → 30M timeline writes per tweet → 10 sec lag.

Solution (2013 onwards):
  Split at threshold (~1M followers):
  < 1M followers: Fan-out on write (push to Redis timelines)
  > 1M followers: Fan-out on read (lazy load at query time, cached separately)
  
  At query time: merge user's precomputed timeline + celebrity tweets.
  
"The Beyoncé problem" drove much of Twitter's infrastructure innovation.
```

---

## Uber

### ATOM (Advanced Trips and Orders Mapping)
```
Uber's early architecture: Python Flask monolith → fell apart at scale.

Uber's H3: Geospatial indexing system
  Divides Earth into hexagonal cells at multiple resolutions.
  Resolution 9 hexagon ≈ 0.1 km² (perfect for city block)
  
  Why hexagons (not squares/triangles)?
  All hexagon neighbors are equidistant from center (not true for squares).
  
  Use in Uber:
    Driver location → H3 cell ID → all drivers in nearby cells = radius query
    Surge pricing computed per cell
    Heat maps rendered from cell aggregates
```

### Schemaless (Uber's MySQL Sharding Layer)
```
Problem: MySQL at Uber couldn't scale writes.
Solution: Custom sharding layer on top of MySQL.

Schemaless:
  Key-value store with append-only semantics.
  Uses MySQL as underlying storage (familiar, ACID for local writes).
  Schemaless handles sharding, routing, replication across nodes.
  
CellStore: stores UUID → JSON blob (arbitrary schema)
Appended rows never deleted → full history maintained.
Used for: trip records, driver earnings, supply tracking.
```

---

## Lessons from Big Tech

| Company | Key Architectural Innovation |
|---------|------------------------------|
| Google | MapReduce + GFS; Bigtable (LSM-Trees); Spanner (TrueTime) |
| Amazon | Microservices mandate; Dynamo (consistent hashing, eventual consistency) |
| Meta | TAO (social graph); Memcached at scale (lease mechanism) |
| Netflix | Chaos Engineering; Open Connect CDN; Hystrix (circuit breaker) |
| Twitter | Hybrid timeline fan-out; Finagle (RPC framework) |
| Uber | H3 geospatial indexing; Schemaless (MySQL sharding) |

### Universal Patterns That Emerged
```
1. Read-heavy systems need aggressive caching (all companies)
2. Event-driven architecture decouples services (all companies)
3. Consistent hashing is foundational for distributed storage (Amazon, Meta, Netflix)
4. Fan-out at write vs read is always context-dependent (Twitter, Meta)
5. Start simple, decompose only when pain demands it (Amazon, Netflix)
6. Test failure proactively in production (Netflix, Google)
7. Measure everything with SLIs, SLOs, alerting (Google, Amazon)
```
