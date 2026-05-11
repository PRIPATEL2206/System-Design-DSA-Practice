# Design: Notification System

**Frequency:** High | **Difficulty:** ⭐⭐⭐ | **Companies:** Amazon, Meta, Google, Apple

---

## 1. Requirements

### Functional
- Send push notifications (iOS, Android)
- Send SMS and Email
- Support scheduling (send later)
- User notification preferences
- Rate limiting (don't spam users)
- Template management
- Analytics (delivery, open rates)

### Non-Functional
- **Scale:** 10B notifications/day
- **Latency:** < 5 seconds from trigger to delivery
- **Reliability:** At-least-once delivery
- **Ordering:** Not required (best effort)

---

## 2. Architecture

```
┌──────────────┐     ┌──────────────────────────────────────────────────┐
│  Trigger     │     │              Notification System                  │
│  Sources     │     │                                                   │
│              │     │  ┌─────────┐   ┌──────────┐   ┌──────────────┐  │
│ • Order Svc  │────▶│  │Notif API│──▶│ Validator │──▶│ Priority Q   │  │
│ • User Svc   │     │  └─────────┘   │& Enricher│   │ (Kafka)      │  │
│ • Cron Jobs  │     │                 └──────────┘   └──────┬───────┘  │
│ • Marketing  │     │                                        │          │
└──────────────┘     │                         ┌──────────────┼──────┐   │
                     │                         │              │      │   │
                     │                    ┌────▼───┐  ┌───────▼┐ ┌──▼──┐│
                     │                    │  Push  │  │  SMS   │ │Email││
                     │                    │ Worker │  │ Worker │ │Workr││
                     │                    └───┬────┘  └───┬────┘ └──┬──┘│
                     │                        │           │         │   │
                     └────────────────────────┼───────────┼─────────┼───┘
                                              │           │         │
                                         ┌────▼──┐  ┌────▼───┐ ┌───▼───┐
                                         │ APNS/ │  │Twilio/ │ │ SES/  │
                                         │ FCM   │  │Vonage  │ │SendGrid│
                                         └───────┘  └────────┘ └───────┘
```

---

## 3. Core Flow

```
1. Service triggers: POST /notifications
   {
     user_id: "123",
     type: "order_shipped",
     data: { order_id: "456", tracking: "..." },
     channels: ["push", "email"]
   }

2. Validator & Enricher:
   - Check user preferences ("does user want push for this type?")
   - Check rate limits ("max 5 push/hour")
   - Check quiet hours ("no notifications 11pm-7am")
   - Enrich with user data (name, email, device tokens)
   - Apply template → render final message

3. Route to channel-specific queue:
   - Push → push_notifications Kafka topic
   - Email → email_notifications Kafka topic
   - SMS → sms_notifications Kafka topic

4. Workers consume and deliver:
   - Push: Call APNS (iOS) or FCM (Android)
   - Email: Call SES/SendGrid API
   - SMS: Call Twilio API

5. Track delivery status:
   - Store in Analytics DB (ClickHouse)
   - Update notification status (sent/delivered/failed/opened)
```

---

## 4. Key Components

### User Preferences
```
{
  user_id: "123",
  channels: {
    push: { enabled: true, quiet_hours: "23:00-07:00" },
    email: { enabled: true, frequency: "daily_digest" },
    sms: { enabled: false }
  },
  types: {
    order_updates: ["push", "email"],
    marketing: ["email"],
    security: ["push", "sms", "email"]  // all channels for security
  }
}
```

### Rate Limiting
```
Rules:
  - Max 5 push notifications per hour per user
  - Max 1 SMS per day per user (expensive!)
  - Max 3 emails per day per user
  - Marketing: max 1 per week
  - Security alerts: no limit

Implementation: Token bucket per (user, channel) in Redis
```

### Template Engine
```
Template: "Hi {{user.name}}, your order #{{order.id}} has shipped! 
           Track it here: {{tracking_url}}"

Variables resolved at send time from notification data.
Supports: localization (i18n), A/B testing variants
```

---

## 5. Push Notification Deep Dive

```
iOS (APNS):
  Your server → APNS (Apple Push Notification Service) → User's iPhone
  Requires: Device token (unique per app per device)
  Protocol: HTTP/2 persistent connection
  Limit: 4KB payload

Android (FCM):
  Your server → FCM (Firebase Cloud Messaging) → User's Android
  Supports: Individual, topic-based, condition-based
  Limit: 4KB payload

Token Management:
  - User registers device token on app install/login
  - Token can change (app reinstall, OS update)
  - Handle invalid tokens (APNS returns 410 → remove token)
  - One user can have multiple devices (send to ALL tokens)

┌─────────────────────────────────────────────────────┐
│ Device Tokens Table                                  │
│                                                      │
│ user_id │ device_token │ platform │ last_active     │
│ 123     │ abc...       │ ios      │ 2024-01-15      │
│ 123     │ def...       │ android  │ 2024-01-14      │
│ 456     │ ghi...       │ ios      │ 2024-01-10      │
└─────────────────────────────────────────────────────┘
```

---

## 6. Reliability

```
At-least-once delivery:
  - If APNS/FCM call fails → retry with exponential backoff
  - After N retries → move to Dead Letter Queue
  - Idempotency key prevents duplicate sends on retry

Monitoring:
  - Delivery rate per channel (target: >99% for push)
  - Latency P50, P99 (trigger to delivery)
  - Failure rate by provider
  - Queue depth (backlog detection)

Fallback:
  - Push fails → try email
  - Email fails → try SMS (for critical notifications)
```

---

## 7. Scaling

```
10B notifications/day = ~115K/sec

Kafka partitions: 100+ (parallel consumption)
Workers: Auto-scale based on queue depth
APNS/FCM: Connection pooling (persistent HTTP/2)
Email: Batch sending (SES supports 500/sec per call)

Priority queues:
  High: Security alerts, OTP (process immediately)
  Medium: Order updates, messages (process within 30s)
  Low: Marketing, recommendations (process within 5min)
```
