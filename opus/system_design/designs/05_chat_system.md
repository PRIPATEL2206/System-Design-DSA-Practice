# Design: Chat System (WhatsApp/Messenger)

**Frequency:** Very High | **Difficulty:** ⭐⭐⭐ | **Companies:** Meta, Google, Amazon, Microsoft

---

## 1. Requirements

### Functional
- 1:1 messaging (real-time)
- Group chat (up to 500 members)
- Online/offline status
- Message delivery receipts (sent, delivered, read)
- Media sharing (images, files)
- Message history & persistence

### Non-Functional
- **Scale:** 1B DAU, 100B messages/day
- **Latency:** < 200ms message delivery
- **Availability:** 99.99%
- **Ordering:** Messages must appear in correct order per conversation
- **Durability:** Never lose a message

### Estimation
```
Messages/sec: 100B / 86400 ≈ 1.2M messages/sec
Storage: 100B × 100 bytes avg = 10 TB/day
Connections: 500M concurrent WebSocket connections
Media: 5% messages have media = 5B × 200KB = 1 PB/day
```

---

## 2. High-Level Design

```
┌─────────┐                                              ┌─────────┐
│ User A  │                                              │ User B  │
│(sender) │                                              │(receiver)│
└────┬────┘                                              └────▲────┘
     │ WebSocket                                              │ WebSocket
     │                                                        │
┌────▼─────────┐    ┌──────────────┐    ┌─────────────────────▼──┐
│ Chat Server  │───▶│ Message Queue│───▶│     Chat Server        │
│ (handles A)  │    │   (Kafka)    │    │  (handles B)           │
└──────────────┘    └──────────────┘    └────────────────────────┘
     │                                              │
     └──────────── write to ────────────────────────┘
                        │
                  ┌─────▼──────┐
                  │ Message DB │
                  │ (Cassandra)│
                  └────────────┘
```

---

## 3. Connection Management

```
Challenge: 500M concurrent connections, each user on ONE chat server

Session Service (Redis):
┌────────────────────────────────────────┐
│  user_id → chat_server_id              │
│  "user_123" → "chat-server-42"        │
│  "user_456" → "chat-server-17"        │
└────────────────────────────────────────┘

When A sends message to B:
1. A's chat server looks up B's server in Session Service
2. Routes message to B's chat server
3. B's chat server pushes via WebSocket to B

If B is OFFLINE:
1. Store message in DB
2. Send push notification (APNS/FCM)
3. When B comes online → pull undelivered messages
```

---

## 4. Message Flow (Detailed)

```
┌───────────── Message Delivery Flow ─────────────┐

1. User A sends message via WebSocket
2. Chat Server validates & assigns message_id + timestamp
3. Write to Message DB (Cassandra) → durability
4. Publish to Kafka topic (partition by receiver_id)
5. Lookup receiver's chat server in Session Service
6. Route to receiver's chat server
7. Push to User B via WebSocket
8. B's client sends ACK → update delivery status
9. When B reads → send "read" receipt back to A

     A sends                    B receives
        │                           ▲
        ▼                           │
   [Chat Server A]            [Chat Server B]
        │                           ▲
        │ ① persist                 │ ⑤ push
        ▼                           │
   [Cassandra]    ② queue    [Session Service]
        │         ─────────▶   "B is on server-B"
        │            ③              │
        └─── Kafka ─────────────────┘
                    ④ route
└──────────────────────────────────────────────────┘
```

---

## 5. Data Model

```
Messages Table (Cassandra — optimized for time-range queries):
─────────────────────────────────────────────────────────────
Partition Key: conversation_id
Clustering Key: message_id (time-ordered, Snowflake ID)

┌─────────────────┬──────────────────────────────────────┐
│ conversation_id │ PK — hash of sorted(user_a, user_b)  │
│ message_id      │ CK — Snowflake ID (time + sequence)  │
│ sender_id       │ Who sent it                           │
│ content         │ Encrypted message text                │
│ type            │ text / image / file / system          │
│ status          │ sent / delivered / read               │
│ created_at      │ Timestamp                             │
└─────────────────┴──────────────────────────────────────┘

Why Cassandra?
- Write-heavy (1.2M msg/sec)
- Partition by conversation → all messages for a chat are together
- Clustering by time → efficient range queries ("last 50 messages")
- Linear scalability with more nodes
```

### Group Chat
```
Group Messages Table:
  Partition Key: group_id
  Clustering Key: message_id

Group Members Table:
  Partition Key: group_id
  → list of user_ids

User Groups Table (reverse index):
  Partition Key: user_id
  → list of group_ids they belong to
```

---

## 6. Group Messaging

```
User A sends to group (200 members):

Option 1: Fan-out on Write
  For each member → route message individually
  ⚠️ 200 writes per message (expensive for large groups)
  ✅ Fast reads (each user's unread queue is pre-built)

Option 2: Fan-out on Read (Chosen for groups)
  Store once in group_messages
  Each member reads from group when they open chat
  ✅ One write per message
  ⚠️ Slower reads (but acceptable for groups)

Hybrid:
  - 1:1 chats: Fan-out on write (speed matters)
  - Small groups (<100): Fan-out on write
  - Large groups (>100): Fan-out on read + push notifications
```

---

## 7. Delivery & Read Receipts

```
Status transitions:
  SENT → DELIVERED → READ

SENT:      Server persisted the message (ACK to sender)
DELIVERED: Recipient's device received it (device ACK)
READ:      Recipient opened the conversation (client event)

Implementation:
- Each status change = update in DB + notify sender
- Batch read receipts (don't send one per message)
- "Last read" pointer per user per conversation
```

---

## 8. Media Handling

```
1. Client uploads media directly to S3 (pre-signed URL)
2. Get media URL back
3. Send message with media_url (not the file itself!)
4. Recipient downloads from CDN (cached)

┌────────┐  ① get upload URL  ┌───────────┐
│Client A│───────────────────▶│App Server │
│        │◀──── pre-signed ───│           │
│        │                    └───────────┘
│        │  ② upload directly
│        │───────────────────▶ S3
│        │                     │
│        │  ③ send msg         │ CDN
│        │  {url: s3://...}    │
└────────┘                     │
                               ▼
                          Client B downloads from CDN
```

---

## 9. End-to-End Encryption (E2E)

```
Signal Protocol (used by WhatsApp):
1. Each user has public/private key pair
2. Messages encrypted with recipient's public key
3. Server CANNOT read message content
4. Forward secrecy: new key for each message session

Server stores: encrypted blobs (can't decrypt)
Server knows: metadata (who messaged whom, when)
```

---

## 10. Scaling

```
Component           Scale Strategy
─────────────────────────────────────────────────
Chat Servers        Horizontal (add more servers, LB distributes)
Session Store       Redis Cluster (sharded by user_id)
Message DB          Cassandra (add nodes, auto-rebalances)
Media               S3 (unlimited) + CDN
Message Queue       Kafka (partition by conversation_id)
Push Notifications  Dedicated service + APNS/FCM
```

### Handling Server Failures
```
Chat Server dies:
1. WebSocket connections drop
2. Clients reconnect → LB routes to different server
3. New server registers in Session Service
4. Pull undelivered messages from queue/DB
5. Resume normal operation

Zero message loss because:
- Messages persisted BEFORE delivery attempt
- Kafka retains undelivered messages
- Client can request messages since last received ID
```
