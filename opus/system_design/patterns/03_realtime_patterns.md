# Real-time & Streaming Patterns

## 1. Real-time Communication Patterns

```
┌────────────────────────────────────────────────────────────────────┐
│ Pattern         │ Latency    │ Use Case                           │
├─────────────────┼────────────┼────────────────────────────────────┤
│ Short Polling   │ 1-30s      │ Legacy fallback                    │
│ Long Polling    │ ~instant   │ Chat (HTTP fallback)               │
│ WebSocket       │ <100ms     │ Chat, gaming, live collab          │
│ SSE             │ <1s        │ Notifications, live feeds          │
│ WebRTC          │ <50ms      │ Video/audio calls, P2P             │
└─────────────────┴────────────┴────────────────────────────────────┘
```

---

## 2. Pub/Sub at Scale

```
                    ┌─────────────────────────────────┐
                    │        Pub/Sub System            │
   Publishers       │                                  │     Subscribers
   ──────────       │  Topic: "user-events"           │     ───────────
                    │  ┌─────────────────────────┐    │
  User Service ────▶│  │ Sub 1: Email Service    │────▶ Email Service
  Order Service ───▶│  │ Sub 2: Analytics        │────▶ Analytics
  Payment Service ─▶│  │ Sub 3: Notification     │────▶ Push Notifications
                    │  └─────────────────────────┘    │
                    └─────────────────────────────────┘

Delivery guarantees:
- At-most-once: Fire and forget (fast, may lose)
- At-least-once: Retry until ACK (may duplicate)
- Exactly-once: Idempotent consumers (hardest)

Filtering:
- Topic-based: Subscribe to entire topic
- Content-based: Subscribe with filter (e.g., "only orders > $100")
- Attribute-based: Filter on message metadata
```

---

## 3. Stream Processing

```
Batch Processing:                Stream Processing:
────────────────                 ────────────────────
Process all data at once         Process data as it arrives
(MapReduce, Spark batch)         (Kafka Streams, Flink, Spark Streaming)

Hours-old results                Seconds-old results
Higher throughput                Lower latency
Simple failure handling          Complex (exactly-once semantics)

Lambda Architecture (both):
─────────────────────────────────────────────────────────────
            ┌──────────────────────────────────────┐
            │          Batch Layer (slow, accurate) │
            │          (Spark, Hadoop)              │
Raw Data ──▶│                                      │──▶ Merged View
            │          Speed Layer (fast, approx)   │
            │          (Kafka Streams, Flink)       │
            └──────────────────────────────────────┘

Kappa Architecture (stream only):
──────────────────────────────────
Raw Data → Kafka → Stream Processor → Serving Layer
(Everything is a stream, replay from Kafka for reprocessing)
```

---

## 4. Presence System (Who's Online?)

```
Requirements:
- Show online/offline/last-seen status
- Handle millions of concurrent users
- Tolerate brief disconnections

Architecture:
┌────────┐  heartbeat   ┌──────────────┐   ┌────────────┐
│ Client │──(every 30s)─▶│ Presence     │──▶│   Redis    │
│        │              │ Service      │   │ (user→time)│
└────────┘              └──────────────┘   └────────────┘

Logic:
- Receive heartbeat → update last_seen timestamp in Redis
- User online if last_seen < 60 seconds ago
- Disconnect → wait 60s before marking offline (handles network blips)
- Fan-out status changes to friends (via pub/sub)

Optimization for large friend lists:
- Don't push to ALL friends (too expensive)
- Only push to friends who are ALSO online
- Or: Pull-based (check status when opening chat)
```

---

## 5. Live Leaderboard

```
Requirements: Real-time ranking of millions of users

Solution: Redis Sorted Set (ZSET)

┌────────────────────────────────────────────────────────────┐
│  ZADD leaderboard 1500 "player_123"  → O(log N)          │
│  ZREVRANK leaderboard "player_123"   → rank = 42 (O(logN))│
│  ZREVRANGE leaderboard 0 9           → top 10 (O(log N+K))│
│  ZCOUNT leaderboard 1000 2000        → players in range    │
└────────────────────────────────────────────────────────────┘

For millions of players across multiple Redis instances:
- Shard by game/region
- Periodic aggregation for global leaderboard
- Cache top 100 aggressively (most requested)
```

---

## 6. Collaborative Editing (Google Docs)

```
Challenge: Multiple users editing same document simultaneously

Approaches:
───────────

1. OT (Operational Transformation) — Used by Google Docs
   - Transform operations against concurrent edits
   - Central server resolves conflicts
   - Complex but proven

2. CRDT (Conflict-free Replicated Data Types) — Used by Figma
   - Data structure guarantees convergence
   - No central coordinator needed
   - Simpler logic but larger data size

Architecture:
┌────────┐  ops   ┌──────────────┐  broadcast  ┌────────┐
│Client A│───────▶│  Collab      │────────────▶│Client B│
│        │◀───────│  Server      │◀────────────│        │
└────────┘        │  (transform) │             └────────┘
                  └──────┬───────┘
                         │ persist
                    ┌────▼────┐
                    │   DB    │
                    └─────────┘
```

---

## 7. Notification Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    Notification System                           │
│                                                                  │
│  Event Source ──▶ Priority Queue ──▶ Rate Limiter ──▶ Router    │
│  (order placed)  (urgent vs normal) (max 5/hour)    ┌───┴───┐  │
│                                                      │       │  │
│                                              ┌───────▼─┐ ┌───▼──┐│
│                                              │  Push   │ │ Email ││
│                                              │ (APNS/  │ │(SES/ ││
│                                              │  FCM)   │ │SMTP) ││
│                                              └─────────┘ └──────┘│
│                                                                  │
│  User preferences: "no email at night", "push for orders only"  │
│  Deduplication: Don't send same notification twice               │
│  Templates: Personalized content per channel                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Rate Limiting at Scale

```
Distributed Rate Limiting with Redis:

Token Bucket (Lua script for atomicity):
─────────────────────────────────────────
local tokens = redis.call('GET', key)
local last_refill = redis.call('GET', key..':time')
local now = ARGV[1]

-- Refill tokens based on elapsed time
local elapsed = now - last_refill
local new_tokens = min(max_tokens, tokens + elapsed * refill_rate)

if new_tokens >= 1 then
    redis.call('SET', key, new_tokens - 1)
    redis.call('SET', key..':time', now)
    return 1  -- allowed
else
    return 0  -- rate limited
end

Sliding Window (sorted set):
────────────────────────────
ZADD user:123:requests <timestamp> <request_id>
ZREMRANGEBYSCORE user:123:requests 0 <now - window_size>
ZCARD user:123:requests  -- count in window
```

---

## Interview Application

When designing real-time features, always discuss:
1. **Connection management** — How many concurrent connections? Memory per connection?
2. **Message ordering** — Does order matter? How to guarantee it?
3. **Offline handling** — What happens when user disconnects? Message queuing?
4. **Scale** — WebSocket servers are stateful → need sticky sessions or shared state
5. **Fallback** — What if WebSocket fails? (Long polling fallback)
