# Design: Search Autocomplete (Typeahead)

**Frequency:** High | **Difficulty:** ⭐⭐⭐ | **Companies:** Google, Amazon, Microsoft, LinkedIn

---

## 1. Requirements

### Functional
- Return top 5-10 suggestions as user types
- Rank by popularity/relevance
- Personalization (user's history)
- Multi-language support
- Filter inappropriate suggestions

### Non-Functional
- **Latency:** < 100ms (must feel instant)
- **Scale:** 10B queries/day, 500K QPS
- **Availability:** 99.99%
- **Freshness:** New trending queries appear within minutes

---

## 2. Architecture

```
┌────────┐  "amaz"   ┌───────────┐         ┌─────────────────┐
│ Client │──────────▶│  API GW   │────────▶│ Autocomplete    │
│        │◀──────────│           │◀────────│ Service         │
└────────┘ suggestions└───────────┘         └────────┬────────┘
                                                     │
                                      ┌──────────────┼──────────┐
                                      │              │          │
                                 ┌────▼───┐   ┌─────▼────┐ ┌───▼──────┐
                                 │  Trie  │   │ Redis    │ │ Personal │
                                 │  Store │   │ Cache    │ │ History  │
                                 └────────┘   └──────────┘ └──────────┘
                                      ▲
                                      │ update
                               ┌──────┴───────┐
                               │ Data Pipeline│
                               │ (aggregates  │
                               │  query logs) │
                               └──────────────┘
```

---

## 3. Data Structure: Trie

```
Trie for prefix-based lookup:

                    root
                   / | \
                  a  b  c ...
                 /|
                m  p
               /|    \
              a  e    p
             /
            z → ["amazon", "amazon prime", "amazing spider-man"]
              (top suggestions stored at each node)

Each node stores:
- Children (26 letters + special chars)
- Top K suggestions for this prefix (pre-computed!)
- Frequency/score for ranking

Query "ama":
  Traverse: root → a → m → a
  Return pre-computed top suggestions at node 'a' (after "am")
  Time: O(prefix_length) ← VERY FAST
```

---

## 4. Pre-computing Top Suggestions

```
Offline pipeline (runs every 5-15 minutes):

1. Aggregate query logs:
   Query: "amazon prime" → searched 50K times today
   Query: "amazon india" → searched 30K times today

2. For each prefix, compute top K:
   "a"   → ["amazon", "apple", "adidas", "airbnb", "adobe"]
   "am"  → ["amazon", "amazon prime", "amd", "amex", "among us"]
   "ama" → ["amazon", "amazon prime", "amazon india", "amazing", ...]

3. Store in Trie (in-memory) or Redis (distributed):
   Key: prefix → Value: top-10 suggestions

Scoring formula:
  score = frequency × recency_decay × quality_factor
  
  recency_decay: Recent searches weighted more
  quality_factor: Penalize short/spam queries
```

---

## 5. Scaling for 500K QPS

```
Strategy: Prefix-based sharding + caching

Tier 1: CDN / Browser Cache
  - Cache responses for popular prefixes (5-10s TTL)
  - "a", "am", "ama" → cached at edge (most common prefixes)
  - Reduces backend load by 80%+

Tier 2: Redis Cache
  - In-memory distributed cache
  - Key: "prefix:{prefix}" → top 10 suggestions
  - TTL: 5 minutes
  - Handles majority of remaining requests

Tier 3: Trie Servers (sharded)
  - Shard A-F: Server 1
  - Shard G-L: Server 2
  - ...
  - Each server holds trie for its prefix range in-memory

┌─────────┐     ┌─────────┐     ┌─────────────────────────────┐
│ Browser │────▶│   CDN   │────▶│   Redis   │──miss──▶│ Trie  │
│  Cache  │     │  Cache  │     │   Cache   │         │ Server│
└─────────┘     └─────────┘     └───────────┘         └───────┘
   5-10s           5s                5min              source of truth
```

---

## 6. Handling Trending Queries

```
Problem: Sudden events (elections, sports, breaking news)
  New query goes from 0 to 1M searches in minutes

Solution: Real-time trending pipeline

┌────────────┐     ┌──────────┐     ┌─────────────┐
│ Query Logs │────▶│  Kafka   │────▶│ Trending    │
│ (real-time)│     │          │     │ Detector    │
└────────────┘     └──────────┘     └──────┬──────┘
                                           │
                                    ┌──────▼──────┐
                                    │ Update Trie │
                                    │ (hot path)  │
                                    └─────────────┘

Trending detection:
  - Sliding window counter per query (last 5 minutes)
  - If count exceeds threshold → immediately inject into suggestions
  - Decay factor: trending queries lose boost over time
```

---

## 7. Personalization

```
Combine global popularity with personal history:

final_score = α × global_score + β × personal_score + γ × recency

personal_score: Based on user's past searches and clicks
recency: More recent personal searches ranked higher

Storage:
  Redis sorted set per user: user:{id}:history
  Contains last 100 searches with timestamps
  
Lookup:
  1. Get global top-10 for prefix
  2. Get user's matching history
  3. Merge and re-rank
  4. Return personalized top-10
```

---

## 8. Client-Side Optimization

```
Debouncing:
  Don't send request on every keystroke!
  Wait 100-200ms after user stops typing, then send.
  
  Typing "amazon":
    'a' → wait... 'm' → wait... 'a' → wait... → send "ama"
    (not 5 separate requests)

Prefetching:
  User types "a" → we return suggestions
  Speculatively prefetch "am", "an", "ap" results
  → Instant response when user types next character

Client cache:
  Store recent responses in browser memory
  "ama" → [...suggestions...]
  If user backspaces and retypes → no network request needed
```
