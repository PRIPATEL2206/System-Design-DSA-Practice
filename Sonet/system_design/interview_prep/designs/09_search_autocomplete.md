# Design: Search Autocomplete / Typeahead

## Problem Statement
Design the search suggestion system that shows relevant completions as a user types (like Google Search suggestions or Amazon product search typeahead).

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Based on what signals? Popularity, personalisation, or both?" → Popularity primarily; personalisation as extension
- "How many suggestions to return?" → Top 5
- "Update frequency for suggestions?" → Suggestions update every 1 hour (not real-time)
- "Multi-language?" → English only initially
- "Mobile and web?" → Both
- "Scale?" → Google scale: billions of queries/day

### Functional Requirements
- As user types, return top 5 query suggestions matching the prefix
- Suggestions ranked by popularity (search frequency)
- Results in < 100ms (must feel instantaneous)

### Non-Functional Requirements
- High read throughput: billions of queries/day
- Low latency: < 50ms P99
- Availability: 99.99%
- Suggestions update without downtime (hourly batch refresh)

---

## Step 2: Capacity Estimation

```
10 billion searches/day on Google
Each search triggers ~5 keystrokes (avg query length = 5 chars)
= 50 billion autocomplete requests/day

Read QPS = 50B / 86,400 ≈ 578,000 RPS
Peak Read QPS ≈ 1–2 million RPS

Response size: 5 suggestions × 50 chars avg = 250 bytes
Bandwidth: 1M RPS × 250 bytes = 250 MB/sec
```

---

## Step 3: High-Level Architecture

```
[User types "prog"]
        |
        | GET /autocomplete?q=prog
        ↓
[API Gateway]
        |
   [Trie Cache (Redis)]
        |
   ──hit──> return top 5 suggestions
   ──miss──> [Trie Service] ──> build from [Trie DB / Sorted Set]
        
[Data Pipeline (offline, runs every hour)]
  [Search Logs] ──Spark job──> [Frequency Aggregator] ──> [Trie Builder] ──> [Trie DB / Redis]
```

---

## Step 4: The Trie Data Structure

A **Trie** (prefix tree) is perfect for prefix matching.

```
Queries: "apple", "app", "application", "apply", "apt"

Trie:
       root
        |
        a
        |
        p
       / \
      p    t
     /|\    \
    l  l  y  apt(3)
    |  i
   apple  application
   (10)   (25)

Each node stores:
  - Character
  - Children (dict of char → node)
  - Is_end: bool (is this a complete query?)
  - Top 5 completions for this prefix (pre-computed!)
  - Frequency count
```

### Pre-computed Top-5 at Each Node
```
The key optimisation: Each trie node pre-stores its top 5 completions.

Node "pro" → completions: ["programming", "project", "properties", "protocol", "proof"]
Node "prog" → completions: ["programming", "program", "progress", "progressive", "prognosis"]

On query "prog":
  1. Traverse trie: p → r → o → g
  2. Return node.top5 ← O(1) lookup!

Building top5 at each node (bottom-up):
  Leaf nodes have exact frequencies.
  Internal nodes = merge children's top lists, keep top 5.
```

---

## Step 5: Data Pipeline (Offline Trie Builder)

```
Every hour:
  1. Aggregate search logs (Spark / Flink)
     - Count query frequency in last 7 days (weighted: recent = more weight)
  
  2. Filter:
     - Remove offensive/inappropriate queries
     - Remove personally identifiable queries (misspelled names, SSNs)
     - Apply safe search rules
  
  3. Build new Trie from top N (e.g., top 1 million queries)
  
  4. Serialize trie to binary format
  
  5. Upload to S3 → distribute to Trie Servers
     (Blue-green deployment: new trie loaded while old serves traffic)
     
  6. Update Redis cache with new trie data
```

---

## Step 6: Storage — Trie in Redis

Storing a full trie as a tree object in Redis is memory-heavy. Better approach:

### Redis Sorted Set Approach
```
For each prefix, store a Sorted Set with top completions and their scores:

Key:   "autocomplete:pro"
Type:  Sorted Set
Members: { "programming": 5000000, "project": 3000000, "properties": 2000000 }

On query "pro":
  ZREVRANGE autocomplete:pro 0 4  → top 5 by score
  → O(log N) lookup

Space: 
  Top 1M queries → ~1000 prefixes per query on avg (1 to 10 chars)
  1M × 1000 prefixes × few hundred bytes = ~1 TB in Redis
  
Filter by prefix length: Only store prefixes up to 10 chars long.
```

---

## Step 7: Personalisation Extension

```
Base result = global popularity ranking
Personalised = blend global + user-specific signal

User-specific signals:
  - User's own recent searches (stored in Redis per user, last 100 searches)
  - User's language/locale
  - User's search history similarity (collaborative filtering)

Blend formula:
  Score = 0.7 × global_popularity + 0.3 × user_affinity_score

Implementation:
  Return global top 5 from Trie Service.
  Blend with user-specific Redis data in the App layer.
  Fallback: global results if no user data.
```

---

## Step 8: Latency Optimisation

```
Goal: < 50ms total (including network round trip)

1. Debounce on client:
   Don't fire request on every keystroke.
   Wait 50ms of inactivity after last keystroke.
   
2. Prefix caching in CDN:
   Popular prefixes ("the", "how", "wh") served from CDN edge.
   These prefixes have stable results that change hourly.
   CDN TTL = 1 hour.
   
3. Browser-side cache:
   Cache results for prefixes typed in current session.
   "prog" result is reused when user types "progr".
   
4. Reduce payload:
   Return only text, no metadata.
   5 suggestions × 30 chars = 150 bytes (tiny response).
```

---

## Step 9: API Design

```
GET /api/v1/autocomplete?q=prog&limit=5&locale=en
Response (< 50ms):
{
  "query": "prog",
  "suggestions": [
    { "text": "programming" },
    { "text": "program" },
    { "text": "progress" },
    { "text": "progressive" },
    { "text": "prognosis" }
  ]
}
```

---

## Step 10: Trie Updates — Handling New Trending Queries

```
Problem: "ChatGPT" becomes a top query overnight. Trie only updates hourly.
Solution:
  Real-time trending queries list (separate from main trie):
    Kafka consumer tracks query counts in 5-min windows.
    Queries exceeding threshold added to "trending" Redis sorted set.
    Autocomplete blends trie results with trending list.

Trie node deletion (removing offensive queries):
  Maintain a blocklist (Redis Set).
  Filter suggestions against blocklist before returning.
  No need to rebuild trie for each removal.
```

---

## Interview Tips

- Lead with: "This is a massive read workload — billions of RPS. The key is pre-computation, not real-time lookup."
- Trie insight: "Pre-store top 5 at each node so lookup is O(prefix length), not O(subtree)."
- Data pipeline: "Trie is rebuilt every hour from aggregated search logs — not per query."
- CDN caching: "Common short prefixes (1-3 chars) are served from CDN edge — 0 server calls."

## Common Follow-up Questions
- "How to handle multi-language?" → Separate trie per language, detect language from user locale
- "How to handle misspellings?" → Levenshtein distance check on miss; DL automaton for fast fuzzy match
- "What if trie gets too large for memory?" → Store in distributed memory (Redis Cluster); or limit to top 10M queries
- "How to test trie updates?" → Shadow mode: new trie serves 1% traffic alongside old trie; compare results
