# Design: Notification System (Push Notifications)

## Problem Statement
Design a system that delivers notifications to users across multiple channels: push (mobile), email, SMS, and in-app.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "What channels? Push, email, SMS, in-app?" → All four
- "Priority levels? Urgent (OTP) vs marketing?" → Yes, different SLAs
- "User notification preferences (opt-in/out)?" → Yes
- "Scale?" → 1 billion notifications/day
- "Delivery guarantees? At-least-once?" → Yes, no loss; deduplication at delivery
- "Real-time or can we batch?" → Urgent = real-time (<1s); marketing = batch OK

### Functional Requirements
- Send notifications via push (iOS/Android), email, SMS, in-app
- Support priority levels: critical, high, normal, low
- User preference management (opt-out per channel per type)
- Delivery tracking (sent, delivered, opened)
- Template management (parameterised message templates)

### Non-Functional Requirements
- Critical notifications (OTP, fraud alerts): < 1 second
- Marketing notifications: within 30 minutes for batch sends
- At-least-once delivery (idempotent at receiver)
- 99.99% availability

---

## Step 2: Capacity Estimation

```
1 billion notifications/day
  Push:  600M (60%)
  Email: 300M (30%)
  SMS:   100M (10%)

Peak QPS: 
  Average = 1B / 86,400 ≈ 11,574 notifications/sec
  Campaign blast (10% of users simultaneously): 50M in 10 min = 83,000/sec peak

Storage:
  Notification record = 1 KB
  1B/day = 1 TB/day (notifications log)
  Keep 90 days = 90 TB
```

---

## Step 3: High-Level Architecture

```
[Producers: Order Service, Auth Service, Marketing Service]
                        |
                        | publish event
                        ↓
              [Notification Service API]
                        |
                        | validate + enrich + check prefs
                        ↓
              [Message Queue (Kafka)]
              ┌─────────┬──────────┬──────────┐
          Push Queue  Email Queue  SMS Queue  In-App Queue
              |           |           |           |
         [Push        [Email       [SMS         [In-App
          Worker]      Worker]      Worker]      Worker]
              |           |           |           |
          [APNs/FCM]  [SendGrid]  [Twilio]    [WebSocket]
           (Apple/      (email     (SMS        (real-time
           Google)      SaaS)      SaaS)       in-app)
              |           |           |
        [Delivery Tracking DB]
```

---

## Step 4: Core Flow

### Step 1: Accept Notification Request
```
POST /api/v1/notifications/send
{
  "user_ids": [123, 456, 789],       -- or "segment": "all_premium_users"
  "template_id": "order_shipped",
  "params": { "order_id": "ABC", "eta": "Jan 5" },
  "priority": "high",
  "channels": ["push", "email"]      -- override user prefs? No — check prefs
}
```

### Step 2: Preference Check
```
Before sending, check user preferences:
  user_preferences table:
    user_id | channel | notification_type | opted_in | quiet_hours_start | quiet_hours_end

  If user opted out of marketing push → skip push for that user
  If user in quiet hours (11pm-8am) → delay non-critical notifications
```

### Step 3: Publish to Queue
```
For each eligible (user, channel) pair:
  Publish to Kafka topic by channel + priority:
    "push_notifications_critical"
    "push_notifications_normal"
    "email_notifications"
    "sms_notifications"
    
Priority queue: Critical consumers have higher priority, dedicated partitions.
```

### Step 4: Delivery Workers
```
Push Worker (APNs/FCM):
  1. Consume message from Kafka
  2. Look up device token for user from device_tokens table
  3. Build push payload
  4. Send to APNs (iOS) or FCM (Android)
  5. Handle response:
     - Success → update delivery status = "delivered"
     - Invalid token → remove token from DB
     - Temporary error → retry with exponential backoff
     - Rate limit from APNs → backpressure → slow down

Email Worker (SendGrid / SES):
  1. Consume from queue
  2. Look up user email + name
  3. Render HTML template with params
  4. Send via SendGrid API
  5. Handle bounce/unsubscribe webhooks from SendGrid
```

---

## Step 5: Template System

```
Templates stored in DB:
  template_id | channel | subject_template | body_template | variables[]

Example:
  id: "order_shipped"
  channel: "email"
  subject: "Your order {{order_id}} has shipped!"
  body: "Hi {{user_name}}, your order is on its way. ETA: {{eta}}."

Rendering:
  Mustache / Handlebars templating
  Params injected at send time

Multi-language:
  template_id + locale → different template content
```

---

## Step 6: In-App Notifications

```
Two types:
  1. Badge/count (user not in app): show red badge
  2. In-app banner (user in app): show notification while using app

In-App Notification DB:
  CREATE TABLE in_app_notifications (
    id          BIGINT PRIMARY KEY,
    user_id     BIGINT,
    message     TEXT,
    type        VARCHAR(50),
    is_read     BOOLEAN DEFAULT FALSE,
    created_at  DATETIME
  );

Delivery:
  If user has active WebSocket → push immediately
  If user offline → store in DB; fetch on next app open

GET /api/v1/notifications?user_id=123&unread_only=true
POST /api/v1/notifications/{id}/read
```

---

## Step 7: Delivery Tracking

```
notification_logs table:
  notification_id | user_id | channel | status | sent_at | delivered_at | opened_at

Status flow:
  QUEUED → SENT → DELIVERED → OPENED
                ↘ FAILED (retry) ↗

Open tracking (email):
  Embed 1x1 pixel image: <img src="https://track.example.com/open/{notification_id}" />
  When loaded → GET request → mark as OPENED

Click tracking:
  Wrap URLs: https://track.example.com/click/{notification_id}?redirect=original_url
```

---

## Step 8: Handling Scale — Bulk Sends

```
Marketing campaign: Send to 100M users.

Naive approach: Loop through 100M users in one job → takes hours.

Better: Fan-out pattern
  1. Marketing team creates campaign targeting "premium users"
  2. User Segment Service queries DB → returns user IDs in batches of 10K
  3. Each batch published to Kafka as one message
  4. Worker expands batch → individual notifications per user
  5. Parallel workers process → 10K workers × 1K notifications/sec = 10M/sec

Rate limiting against 3rd party providers:
  APNs: max 300M notifications/hour per app
  SendGrid: rate limit per plan
  Implement token bucket per provider → smooth out sends
```

---

## Interview Tips

- Key insight: "Each channel is a separate consumer group from Kafka — decoupled, independently scalable."
- Always mention: "User preference check before enqueuing — respect opt-outs."
- Third-party providers: "APNs and FCM have rate limits — we need a token bucket in our workers."
- Retry logic: "Exponential backoff for temporary failures; dead letter queue for permanent failures."

## Common Follow-up Questions
- "How to ensure no duplicate notifications?" → Idempotency key per notification; check before sending
- "How to handle APNs token expiry?" → Handle 410 response → remove token from DB
- "How to A/B test notification copy?" → Template variants; track open rates per variant
- "How to personalise at scale?" → ML model pre-computes best time + channel per user
