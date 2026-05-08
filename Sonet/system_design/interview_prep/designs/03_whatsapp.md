# Design: WhatsApp / Messenger (Chat System)

## Problem Statement
Design a real-time chat application that supports one-on-one and group messaging with message delivery guarantees.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "1-on-1 only, or group chats too? Max group size?" → Both; groups up to 500 members
- "Media support (images, video, audio)?" → Yes, images and documents
- "Online indicators and read receipts?" → Yes
- "Message history (how long to store)?" → Forever
- "Push notifications for offline users?" → Yes
- "Scale?" → 500M DAU, WhatsApp scale

### Functional Requirements
- 1-on-1 messaging and group messaging (up to 500 members)
- Message delivery with receipts (sent, delivered, read — like WhatsApp ticks)
- Online status and last seen
- Push notifications for offline users
- Media sharing

### Non-Functional Requirements
- Messages delivered in < 100ms for online users
- At-most-once delivery to UI; at-least-once to ensure no loss
- Strong consistency within a conversation (messages in order)
- High availability (99.99%)

---

## Step 2: Capacity Estimation

```
DAU = 500 million
Messages/user/day = 40
Total messages/day = 500M × 40 = 20 billion

Write QPS = 20B / 86,400 ≈ 231,000 msg/sec
Read QPS  ≈ same (1:1 read/write for chat)

Storage:
  Avg message = 100 bytes
  Daily = 20B × 100 bytes = 2 TB/day
  5-year text ≈ 3.6 PB
  Media (20% of messages, 100KB avg): 0.2 × 20B × 100KB = 400 TB/day
```

---

## Step 3: High-Level Architecture

```
[User A (Online)]                   [User B (Online)]
       |                                   |
    WebSocket                          WebSocket
       |                                   |
[Chat Server A]                     [Chat Server B]
       |                                   |
       |──> [Message Queue (Kafka)] <──────|
                    |
          [Message Service]
               |       |
        [Message DB]  [Presence Service]
        (Cassandra)    (Redis)
               |
        [Notification Service]
               |
        [APNs / FCM]  ← for offline users
```

---

## Step 4: Core Component: Real-time Messaging

### WebSocket Connections
```
Each connected user maintains a persistent WebSocket connection to a Chat Server.

User A → connects to Chat Server 1
User B → connects to Chat Server 3

To send from A to B:
1. A sends message over WebSocket to Chat Server 1
2. Chat Server 1 looks up which server B is connected to (via Presence Service/Redis)
3. Chat Server 1 forwards to Chat Server 3 via internal routing
4. Chat Server 3 pushes to User B's WebSocket

OR (simpler): Use message queue to decouple
1. Chat Server 1 receives message
2. Publishes to Kafka topic "chat_messages"
3. Chat Server 3 consumes and pushes to B
```

### Message Routing via Redis
```
On connect:  SET ws_server:user_id → chat_server_id (with TTL = connection lifetime)
On message:  GET ws_server:recipient_id → which server to route to
On disconnect: DEL ws_server:user_id
```

---

## Step 5: Message Delivery Receipts

WhatsApp-style ticks:

```
1 grey tick  = Sent (stored in server DB)
2 grey ticks = Delivered (received by recipient's device)
2 blue ticks = Read (recipient opened the conversation)

Implementation:
- Sender receives ACK from server → update to "Sent"
- When recipient's device receives message → device sends "Delivered" ACK to server
  → server notifies sender
- When recipient opens conversation → device sends "Read" ACK
  → server notifies sender

Message status stored in DB:
  message_id | sender_id | status | updated_at
```

---

## Step 6: Handling Offline Users

```
When recipient is OFFLINE:
1. Server cannot push via WebSocket
2. Store message in DB (Cassandra) — message remains pending
3. Trigger push notification via Notification Service
   → APNS (Apple) or FCM (Google)
   → User sees notification, taps it → app opens → connects WebSocket
   → App downloads pending messages from server on reconnect

On reconnect, app sends:
  GET /messages?conversation_id=X&after_message_id=Y
  → Server returns all pending messages
  → App ACKs receipt → server updates message status
```

---

## Step 7: Group Chat

```
Group chat challenges:
1. Fan-out: Message from A to group of 500 → 499 delivers needed
2. Ordering: All members must see messages in same order

Design:
1. Each group has a group_id
2. Message sent to group → stored once in DB with group_id
3. Each member's "inbox" gets reference to that message_id

Inbox table:
  user_id | message_id | conversation_id | read_at | delivered_at

Storage efficiency:
  One message copy → many inbox references
  (Don't store 500 copies of same message)
```

---

## Step 8: Data Model

### messages table (Cassandra — write-heavy, time-series)
```
CREATE TABLE messages (
  conversation_id  UUID,
  message_id       TIMEUUID,    -- auto-ordered by time
  sender_id        BIGINT,
  content          TEXT,
  media_url        TEXT,
  type             ENUM('text', 'image', 'doc'),
  status           ENUM('sent', 'delivered', 'read'),
  created_at       TIMESTAMP,
  PRIMARY KEY (conversation_id, message_id)  -- partition by conversation
) WITH CLUSTERING ORDER BY (message_id DESC);
```

**Why Cassandra?** Optimised for write-heavy time-series. Partition by conversation_id keeps related messages together.

### conversations table
```sql
CREATE TABLE conversations (
  id          UUID PRIMARY KEY,
  type        ENUM('direct', 'group'),
  created_at  DATETIME
);
```

### participants table
```sql
CREATE TABLE participants (
  conversation_id  UUID,
  user_id          BIGINT,
  last_read_at     DATETIME,
  PRIMARY KEY (conversation_id, user_id)
);
```

---

## Step 9: Presence Service (Online Status)

```
Redis-based:
  On user connect:    SET presence:user_id "online" EX 300  (5 min TTL)
  Heartbeat every 30s: EXPIRE presence:user_id 300
  On user disconnect: DEL presence:user_id (or let TTL expire)
  
  Check status:       GET presence:user_id
    → "online" = online
    → nil = offline

"Last seen":
  On disconnect: SET last_seen:user_id <timestamp>
```

---

## Step 10: API Design

```
WebSocket:
  ws://chat.example.com/connect?token=JWT
  
  Client → Server:  { "type": "message", "to": 456, "content": "hey" }
  Server → Client:  { "type": "message", "from": 123, "content": "hey", "msg_id": "abc" }
  Client → Server:  { "type": "ack", "msg_id": "abc", "status": "delivered" }

REST APIs:
  GET  /conversations                    ← list conversations
  GET  /conversations/{id}/messages      ← load message history
  POST /conversations/{id}/messages      ← fallback (non-WebSocket)
  GET  /users/{id}/presence              ← check online status
```

---

## Interview Tips

- Immediately establish: "I'll use WebSocket for real-time bidirectionality. Long polling is too slow."
- Key insight: Session server (which WebSocket server is user on?) solved by Redis lookup.
- Message storage: "I'd use Cassandra — it's optimised for write-heavy, time-ordered data."
- Don't forget offline: "When recipient is offline, push via APNS/FCM, sync on reconnect."

## Common Follow-up Questions
- "How to scale to 500M concurrent WebSocket connections?" → Stateless chat servers, Redis for routing, horizontal scaling
- "How to handle network partitions?" → Message queue (Kafka) decouples delivery; at-least-once semantics
- "End-to-end encryption?" → Key exchange on client side (Signal protocol); server stores only ciphertext
- "How to support voice/video calls?" → WebRTC for peer-to-peer; TURN/STUN servers for NAT traversal
