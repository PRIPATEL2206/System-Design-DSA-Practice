# System Design Interview Playbook

## How System Design Interviews Are Evaluated

| Criteria | Weight | What They Look For |
|----------|--------|-------------------|
| Requirements | 15% | Asks good questions, scopes appropriately |
| High-level design | 25% | Clear architecture, correct data flow |
| Deep dive | 30% | Technical depth, handles probing questions |
| Trade-offs | 20% | Discusses alternatives, justifies choices |
| Communication | 10% | Organized, clear, drives the conversation |

---

## The 35-Minute Blueprint

### Minutes 0-5: Requirements & Scope
```
"Before I design, let me clarify the requirements."

Ask:
1. Who are the users? How many? (Scale)
2. What are the core features? (Scope — pick top 3-4)
3. Read-heavy or write-heavy? (Architecture direction)
4. Consistency requirements? (Strong vs eventual)
5. Latency requirements? (Real-time vs batch)
6. Any geographic distribution? (Multi-region?)

Then: "Let me do a quick capacity estimate..."
- Users × actions/day = QPS
- Storage per record × records/day × retention = total storage
- Bandwidth = QPS × response size
```

### Minutes 5-20: High-Level Design
```
1. Start with the simplest architecture that works
   Client → Load Balancer → App Server → Database

2. Add complexity based on requirements:
   - High read traffic → Cache layer (Redis)
   - High write traffic → Message Queue (Kafka)
   - Large files → Object Storage (S3) + CDN
   - Search → Search Engine (Elasticsearch)
   - Real-time → WebSockets
   - Analytics → Separate data warehouse

3. Draw clear data flow for top 2-3 operations
   "When a user posts a tweet, here's what happens..."
   "When a user opens their feed, here's the flow..."

4. Define APIs for key operations
   POST /api/v1/tweets { text, media_ids }
   GET /api/v1/feed?cursor=xxx&limit=20
```

### Minutes 20-30: Deep Dive
The interviewer will pick 1-2 areas. Common deep dives:
```
- Database schema and choice
- Sharding/partitioning strategy
- Caching strategy and invalidation
- How to handle failures/edge cases
- How to scale a specific bottleneck
- Consistency guarantees
- Data model for specific use cases
```

### Minutes 30-35: Wrap-up
```
"To summarize the key trade-offs:
- We chose eventual consistency for the feed because [reason]
- We used fan-out-on-write for regular users but fan-out-on-read for celebrities
- The main scaling challenge would be [X], which we address with [Y]

If I had more time, I'd add:
- Rate limiting on the write path
- Monitoring and alerting
- A/B testing framework
- International CDN strategy"
```

---

## System Design Communication Tactics

### How to Drive the Conversation
You should be talking 70% of the time. The interviewer should be asking questions, not directing you.

**DO:**
- "Let me start with the requirements..."
- "I'm going to sketch the high-level architecture first, then dive deep."
- "There are two approaches here. Let me discuss the trade-offs..."
- "Should I go deeper on [X] or move to [Y]?"

**DON'T:**
- Wait for the interviewer to ask "What's next?"
- Say "I don't know what to do next"
- Jump randomly between topics

### How to Discuss Trade-offs
```
"We could use [Option A] or [Option B].

Option A gives us [benefit] but costs us [drawback].
Option B gives us [benefit] but costs us [drawback].

Given our requirements of [X], I'd go with Option [A/B] because [reason].
But we should revisit this if [condition changes]."
```

### When Interviewer Pushes Back
This is a GOOD sign — they want to see how you think under pressure.

```
Interviewer: "But what if the cache goes down?"
You: "Good point. We'd need:
1. Cache cluster with replication (Redis Sentinel or Cluster)
2. Fallback to database (circuit breaker pattern)
3. Cache warming on recovery
4. The system degrades gracefully — slower, but doesn't fail."
```

---

## The 10 Most Asked System Design Questions

### Tier 1 (Prepare First)
| # | System | Key Concept | Typical Level |
|---|--------|-------------|--------------|
| 1 | URL Shortener | Hashing, DB, Cache | Mid/Senior |
| 2 | Rate Limiter | Distributed algorithms | Mid/Senior |
| 3 | News Feed / Timeline | Fan-out, caching | Senior |
| 4 | Chat System | WebSockets, message delivery | Senior |
| 5 | YouTube / Netflix | CDN, transcoding, streaming | Senior |

### Tier 2 (Know Well)
| # | System | Key Concept | Typical Level |
|---|--------|-------------|--------------|
| 6 | Uber / Ride Matching | Geospatial, real-time | Senior |
| 7 | Google Drive / Dropbox | Sync, chunking, conflict | Senior |
| 8 | Notification System | Multi-channel, priorities | Senior |
| 9 | Typeahead / Autocomplete | Trie, pre-computation | Mid/Senior |
| 10 | Distributed Cache | Consistent hashing | Senior+ |

---

## Handling "I've Never Designed This"

You won't have prepared every possible question. Here's how to handle a novel one:

1. **Don't panic.** The framework works for any system.
2. **Map to known patterns:**
   - "This is similar to [X] because it also needs [property]"
   - Most systems combine: storage + compute + caching + messaging
3. **Start from user actions:**
   - "Let me think about what the user does step by step..."
   - Each action → API → service → storage
4. **Ask what's unique about this system:**
   - What makes Twitter different from Facebook? (Real-time feed vs social graph)
   - What makes Uber different from a messaging app? (Geospatial + matching)

---

## Numbers to Memorize

### Latency
```
L1 cache reference:             0.5 ns
L2 cache reference:             7 ns
RAM reference:                  100 ns
SSD random read:                150 μs
HDD seek:                       10 ms
Network (same datacenter):      0.5 ms
Network (cross-continent):      150 ms
```

### Throughput
```
Single server:                  10K-100K QPS (depending on workload)
Redis:                          100K+ QPS
PostgreSQL:                     10K-50K QPS (with indexing)
Kafka:                          Millions messages/sec per cluster
```

### Storage
```
1 char:                         1 byte (ASCII) or 2-4 bytes (UTF-8)
Average tweet:                  ~300 bytes
Average image:                  300 KB
Average video (1 min):          50 MB (compressed)
1 billion records × 1KB each:   ~1 TB
```

### Scale Reference
```
Google Search:            80K+ QPS
Instagram:               1M+ photos uploaded/day
WhatsApp:                100B+ messages/day
YouTube:                 500 hours video uploaded/minute
Twitter:                 6K tweets/second (peak: 150K+)
```

---

## Red Flags That Tank System Design Interviews

1. **Over-engineering:** Adding Kafka, K8s, microservices for 100 users
2. **Under-engineering:** Single database for 100M users
3. **No trade-off discussion:** Just stating decisions without explaining why
4. **Ignoring requirements:** Designing features that weren't asked for
5. **No capacity estimation:** Can't justify infrastructure choices
6. **Single point of failure:** No redundancy in critical path
7. **Ignoring interviewer's hints:** They're guiding you — follow the lead
8. **Getting stuck on one component:** Spend max 5 minutes per deep dive unless asked

---

## Practice Routine

### How to Practice System Design (Solo)
1. Set a 35-minute timer
2. Pick a system from the list above
3. Talk out loud (record yourself if possible)
4. Draw on paper/whiteboard
5. After: Compare against top YouTube walkthroughs (Alex Xu, Gaurav Sen)
6. Note what you missed

### How to Practice with a Partner
1. One person is interviewer, one is candidate
2. Interviewer: Ask clarifying questions, push on weak points, ask "what if X fails?"
3. Switch roles after 40 minutes
4. Debrief: What was strong? What was missed?
